# Worket — Arquitetura de Software e Engenharia de Sistemas

Este documento detalha o design de software, os padrões de arquitetura, o modelo de segurança e os protocolos de comunicação implementados no **Worket**.

---

## 1. Visão Geral da Arquitetura em Camadas

O Worket adota uma arquitetura em camadas desacopladas que garante isolamento de processos, proteção de dados locais e estabilidade de execução:

```
┌─────────────────────────────────────────────────────────────┐
│             Processo de Interface (Electron Renderer)        │
│  - Protocolo Estável: worket://app                          │
│  - Monaco Editor 0.52 (Monaco Models & Web Workers)         │
│  - Xterm.js 6.0 (Terminal Virtual com Addon Fit)            │
│  - Motor SPA Responsivo (Work / Chat / Arena)               │
└──────────────┬──────────────────────────────▲───────────────┘
               │ contextBridge IPC            │
┌──────────────▼──────────────────────────────┴───────────────┐
│            Processo Host Desktop (Electron Main)            │
│  - Janela Nativa com Context Isolation & Sandbox Ativo      │
│  - Gerenciador de Instância Única & Estado de Geometria      │
│  - Alocação Dinâmica de Porta de Loopback                    │
│  - Injeção de Token Privado Efêmero de Autenticação          │
└──────────────┬──────────────────────────────▲───────────────┘
               │ Spawn de Serviço Local       │
┌──────────────▼──────────────────────────────┴───────────────┐
│           Serviço de Engenharia (Worket Node.js Engine)      │
│  - Daemon HTTP + WebSocket em 127.0.0.1:<dynamic_port>      │
│  - Abstração de Adaptadores: HarnessAdapter                 │
│  - Gestor de Arquivos & Proteção Concorrente (files.mjs)    │
│  - PTY Multiplexer & Buffer Ring (terminal.mjs)             │
│  - Suíte Git, Checkpoints & Árvores Temporárias             │
│  - Compilador de Contexto GLM Handoff (handoff.mjs)         │
└──────┬───────────────────────┬──────────────────────────────┘
       │ JSON-RPC / WS         │ HTTP REST + SSE
┌──────▼──────────────┐ ┌──────▼──────────────┐ ┌─────────────┐
│   Daemon Hermes     │ │   Daemon OpenCode   │ │ Futuros:    │
│   (hermes serve)    │ │   (/api/session)    │ │ Claude Code │
│   Porta Dinâmica    │ │   Porta Dinâmica    │ │ Antigravity │
│   Loopback 127.0.0.1│ │   Loopback 127.0.0.1│ │ Codex       │
└─────────────────────┘ └─────────────────────┘ └─────────────┘
```

---

## 2. Princípio Central: Abstração de Harnesses Nativos

A inovação central do Worket é o **`HarnessAdapter`**. A interface não possui acoplamento direto com a sintaxe de agentes específicos; ela consome uma interface abstrata que padroniza o ciclo de vida:

```typescript
interface HarnessAdapter {
  createSession(options: SessionOptions): Promise<string>;
  resumeSession(sessionId: string): Promise<SessionState>;
  executeTurn(prompt: string, attachments: Attachment[]): AsyncIterable<HarnessEvent>;
  interruptExecution(): Promise<void>;
  respondToPermission(requestId: string, approved: boolean): Promise<void>;
  fetchCatalog(): Promise<ModelCatalog>;
}
```

### 2.1. Adaptador Hermes (`HermesAdapter`)
- **Inicialização:** Spawna `~/.local/bin/hermes serve --host 127.0.0.1 --port 0 --isolated --skip-build`.
- **Descoberta:** O Hermes grava um arquivo de prontidão temporário contendo a porta efêmera alocada.
- **Protocolo:** WebSocket JSON-RPC 2.0 em `/api/ws`.
- **Gerenciamento de Raiz:** Rastreia a raiz lógica da sessão (`native_session_bindings`) para que compactações automáticas do Hermes não causem desconexão ou ramificações errôneas na interface.

### 2.2. Adaptador OpenCode (`OpenCodeAdapter`)
- **Inicialização:** Conecta-se à instância existente em `~/.local/state/opencode/service.json` ou spawna `~/.opencode/bin/opencode serve --service --hostname 127.0.0.1 --port 0`.
- **Protocolo:** Chamadas REST para `/api/session/...` e subscrição em `/api/event` via Server-Sent Events (SSE).
- **Projeção de Erros:** Captura e armazena payloads de erro nativos (`data.error`), garantindo que o histórico recarregado mostre a falha exata ocorrida, sem omitir turnos.

---

## 3. Motor de Handoff de Contexto (GLM Engine)

O Worket resolve o problema clássico de esgotamento de contexto de agentes ou necessidade de transferir tarefas entre modelos especializados:

### Fluxo de Trabalho do Handoff:
1. **Compilação de Histórico:** O backend consolida mensagens anteriores, arquivos alterados no Git e anotações do usuário. Históricos grandes (>180 KB) são divididos em chunks processados sequencialmente.
2. **Chamada Estruturada ao Modelo GLM:** Envia o pacote para o OpenRouter solicitando estritamente um JSON Schema com `response_format: { type: "json_object" }`.
3. **Validação do Dossiê:** A função `validateDossier()` audita a estrutura retornada.
4. **Materialização em Disco (`materializeDossier`):**
   - Verifica segurança contra path traversal (`realpath` restrito ao workspace);
   - Lê os arquivos classificados como *essenciais* direto do disco no momento presente (impedindo que o novo agente use código desatualizado da conversa antiga);
   - Move arquivos > 400 linhas para *arquivos de referência* para economizar janela de contexto.
5. **Transplante:** Cria uma sessão nativa real no agente destino com o dossiê já injetado como estado base.

---

## 4. Sistema de Checkpoints de Árvore Git

Para garantir que o desenvolvedor tenha controle total sobre alterações automatizadas, a classe `Checkpoints` gerencia snapshots transacionais da árvore Git:

```
Turno do Agente Iniciado
        │
        ▼
Captura git write-tree da área de trabalho
        │
        ▼
Cria ref isolada: refs/worket/checkpoints/<id>/before
        │
        ▼
Agente executa alterações nos arquivos
        │
        ▼
Turno Finalizado ──► Grava refs/worket/checkpoints/<id>/after
        │
        ├─► [Sucesso] Usuário continua codificando
        │
        └─► [Falha] Usuário clica em Desfazer ──► git read-tree restaura o estado exato
```

- **Zero Poluição:** Não cria commits ruidosos na branch do usuário; os checkpoints são armazenados em `refs/worket/checkpoints/`.
- **Autolimpeza:** Mantém apenas os últimos 40 checkpoints ativos, eliminando refs antigas automaticamente.

---

## 5. Proteção de Concorrência de Arquivos e Watchers

A camada `lib/files.mjs` e `public/editor-access.js` implementa proteção ativa contra condições de corrida:

1. **Assinatura de Arquivo:** Ao abrir qualquer arquivo, o Worket guarda seu `mtimeMs` e hash SHA-256.
2. **Proteção de Visualizadores (HTTP 415):** Ao receber qualquer requisição de salvamento textual em imagem, SQLite ou binário, a rota rejeita com HTTP 415, abortando a escrita.
3. **Detecção de Conflito de Disco (HTTP 409):** Caso o arquivo tenha sido alterado em disco entre a leitura e a tentativa de gravação, o servidor rejeita com status 409, disparando no cliente a barra de resolução em 3 vias (*Comparar*, *Usar disco*, *Manter rascunho*).

---

## 6. Terminal PTY Virtualizado

A camada `lib/terminal.mjs` gerencia o ciclo de vida dos terminais do sistema:

- **Alocação PTY:** Utiliza `node-pty` compilado nativamente para Linux C++ (`pty.node`).
- **Anel de Buffers:** Cada terminal retém histórico circular recente para recomposição rápida ao alternar abas ou redimensionar janelas.
- **Integração de Shell:** Injeta `scripts/terminal-bashrc.sh` no ambiente para capturar marcadores de comando e códigos de retorno sem bloquear o stdin do usuário.
- **Detecção de Scripts:** O parser lê dinamicamente `package.json` e `Makefile` da raiz do workspace e expõe atalhos rápidos na interface.

---

## 7. Isolamento de Segurança Desktop

O empacotamento desktop (`desktop/main.cjs` e `desktop/security.cjs`) implementa os mais rigorosos padrões de segurança recomendados para aplicações Electron:

| Configuração | Valor | Justificativa |
| :--- | :--- | :--- |
| `contextIsolation` | `true` | Impede que scripts do renderizador acessem protótipos internos do Node. |
| `nodeIntegration` | `false` | Remove completamente os símbolos `require`, `process` e `Buffer` do contexto web. |
| `sandbox` | `true` | Ativa a sandbox de sistema operacional do Chromium para isolar o processo renderizador. |
| `webSecurity` | `true` | Bloqueia requisições cross-origin não autorizadas. |
| Protocolo Personalizado | `worket://app` | Fornece uma origem estável e segura para carregamento de assets locais sem expor o esquema `file://`. |
| Loopback Efêmero | `127.0.0.1:<port>` | O daemon Node só escuta na interface de loopback, rejeitando conexões externas da rede local. |

---

<div align="center">

*Worket — Arquitetura robusta para uma experiência de engenharia de software sem atritos.*

</div>
