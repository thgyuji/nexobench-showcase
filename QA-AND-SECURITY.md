# NexoBench — Relatório de Qualidade (QA) e Engenharia de Segurança

A confiabilidade é o pilar central do NexoBench. Para um aplicativo desktop que lida diretamente com código-fonte, branches do Git e automações autônomas com agentes de inteligência artificial, falhas silenciosas ou perda de dados são inaceitáveis.

Este documento consolida a metodologia de testes, os resultados da auditoria de QA e as garantias de integridade implementadas no projeto.

---

## 1. Métricas de Qualidade e Cobertura de Testes

| Métrica | Resultado | Observações |
| :--- | :--- | :--- |
| **Total de Testes Automatizados** | **70 / 70 Aprovados** | Zero falhas, zero testes ignorados. |
| **Compatibilidade Node.js Nativo** | **Node.js 22.23.2** | 100% de aprovação na suíte completa. |
| **Compatibilidade Runtime Electron** | **Node.js 24.22.0** | 100% de aprovação dentro do Electron 44.9.0 (`ELECTRON_RUN_AS_NODE=1`). |
| **Verificação Sintática de Código** | **Aprovado** | `npm run check` com análise estrita em todos os scripts ESM, CJS e JS. |
| **Validação de Pacotes Linux** | **Aprovado** | Instaladores `.deb` amd64 e `.AppImage` x86_64 gerados e validados no smoke test. |

---

## 2. Garantia de Preservação e Imutabilidade de Dados

Durante as auditorias de QA, foi verificada a proteção dos arquivos do workspace contra corrupções involuntárias (ex: pressionamento acidental de `Ctrl+S` em visualizadores que não suportam edição textual):

### Teste de Estresse de Gravação Textual em Visualizadores Protegidos:
- Foram realizadas tentativas explícitas de salvamento textual através da interface e via requisições HTTP simuladas diretamente para a API.
- **Resultado:** O serviço local interceptou todas as tentativas, rejeitou a operação com status `HTTP 415 Unsupported Media Type` e preservou os bytes exatos dos arquivos.

```
+----------------+---------------------+---------------------+------------------------------------------------------------------+
| Arquivo        | Bytes Antes         | Bytes Depois        | Hash SHA-256 (Idêntico Antes e Depois)                           |
+----------------+---------------------+---------------------+------------------------------------------------------------------+
| pixel.png      | 68 bytes            | 68 bytes            | 9f81650ff972f04ecb98e05d93c4cc65f22fb721385b8529d3ae3c00d479f786 |
| sample.sqlite  | 8192 bytes          | 8192 bytes          | 5a6f78aa05ae32b1dfe90ce1ed2a08ef091c9a1ff56c1387453fcddbd54b971b |
| binary.bin     | 5 bytes             | 5 bytes             | ef7e301027f931dfba06c7ded4ef305797f43cc115a664f9af9b57d08c3172c2 |
+----------------+---------------------+---------------------+------------------------------------------------------------------+
```

---

## 3. Matriz de Correções e Fechamento de Achados de QA

| ID | Prioridade | Problema Identificado | Solução Implementada |
| :--- | :---: | :--- | :--- |
| **QA-01** | **P1** | Risco de sobrescrever arquivos binários ou SQLite como texto vazio. | Bloqueio em duas camadas: no cliente (desativação de botões e atalhos) e no backend com reclassificação de tipo em disco e retorno `HTTP 415`. |
| **QA-02** | **P1** | Modelo do Monaco permanecia ativo ao trocar para visualizadores. | Desvinculação explícita de modelos do Monaco e do buffer de Diff ao navegar para arquivos não textuais. |
| **QA-03** | **P1** | Iframe de preview web bloqueado por elementos ocultos no DOM. | Algoritmo refinado de detecção de visibilidade que inspeciona ancestrais, estilos computados e atributo `hidden`. |
| **QA-04** | **P2** | Abas do editor comprimiam botões de ação em larguras intermediárias. | Linha dedicada e rolável para as abas de arquivo, separando os botões de ação para linha secundária. |
| **QA-05** | **P2** | Barra de busca sobrepunha seletores de modo em janelas menores. | Redimensionamento flexível da barra de pesquisa e transição para ícone de busca em larguras <= 900px. |
| **QA-06** | **P2** | Busca de arquivos dependia de pastas terem sido previamente expandidas. | Criação do `workspace-catalog.js`, indexando o catálogo completo do projeto sem depender do estado da árvore visual. |
| **QA-07** | **P2** | Agentes não instalados podiam exibir falso positivo de prontidão. | Validação estrita de executáveis (`which` + execução real de `--version`), diferenciando executável ausente de erro de versão. |
| **QA-08** | **P2** | Falhas e erros de agentes eram perdidos ao recarregar a página. | Projeção estruturada de erros (`data.error`) persistida no banco SQLite local e renderizada visualmente no histórico. |
| **OBS-01**| **Robustez**| Diálogos nativos `window.prompt` podiam travar a renderização. | Substituição total por diálogos customizados em HTML com suporte a teclado (`Tab`, `Enter`, `Escape`). |
| **OBS-02**| **Feedback**| Toasts exibiam referências cíclicas ou `[object Object]`. | Módulo universal `error-message.js` com normalização de erros, eventos e mensagens amigáveis. |

---

## 4. Scripts de Verificação Automatizada Disponíveis

Para assegurar testes contínuos e reprodutibilidade, o projeto disponibiliza scripts especializados:

1. **`npm test`**: Executa a suíte de 70 testes automatizados com o test runner nativo do Node.js (`node --test tests/*.test.mjs`).
2. **`node scripts/smoke-package.mjs`**: Valida a integridade do pacote gerado pelo Electron-Builder, testando a subida do backend empacotado, autenticação com credencial privada, proteção de gravação e terminal PTY.
3. **`node scripts/verify-window.mjs`**: Automatiza a janela empacotada do Electron em ambiente temporário via Chrome DevTools Protocol (CDP), verificando abertura de diálogos, terminal real e preview nativo.
4. **`node scripts/verify-inference.mjs`**: Script de verificação de ponta a ponta com agentes reais e consumo de API para validação de entrega final.

---

<div align="center">

*Qualidade de software com evidências verificáveis, testes automatizados e proteção de dados.*

</div>
