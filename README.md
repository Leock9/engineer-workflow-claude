# engineer-workflow-claude

> Um workflow de desenvolvimento multi-agente e spec-first para o [Claude Code](https://claude.ai/code) — transformando a descrição de uma tarefa em código funcional e revisado através de um pipeline estruturado de agentes especializados.

![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)

---
<img width="855" height="506" alt="image" src="https://github.com/user-attachments/assets/45ff66c7-87f7-44ba-9267-b22b24e201aa" />

## Visão Geral

Este projeto implementa um **pipeline de agentes orquestrado** sobre o Claude Code. Em vez de pedir a um único modelo que detecte, projete, implemente e revise tudo de uma vez, cada responsabilidade é decomposta em um subagente especializado — cada um com um papel focado, um modelo específico e uma entrada condensada.

O workflow impõe uma **disciplina spec-first**: nenhum código é escrito até que uma decisão arquitetural e uma especificação formal tenham sido revisadas e aprovadas pelo desenvolvedor. Isso cria um gate humano no loop que impede construir a coisa errada antes que as perguntas certas sejam respondidas.

Isto não é um framework e não substitui o desenvolvedor. É uma camada de processo estruturada — um conjunto de skills e agentes que trazem ordem, rigor e consistência à forma como o Claude Code é usado em um projeto de software.

---

## Arquitetura

O sistema é composto por **26 subagentes especializados** orquestrados por **16 skills**. Os agentes são atribuídos a modelos de acordo com a complexidade da tarefa:

```mermaid
graph TD
    subgraph Haiku["⚡ Haiku — Estrutural / Rápido"]
        D[Detector]
        CE[CodeExplorer]
        EP[EvidenceParser]
        LC[LogClassifier]
        RC[ReviewCoordinator]
        MV[MdValidator]
    end

    subgraph Sonnet["🧠 Sonnet — Análise / Código"]
        A[Architect]
        DS[DiscoverySynthesizer]
        I[Implementer]
        RV[Reviewer]
        PR[PerformanceReviewer]
        TR[TestReviewer]
        SR[SpecReviewer]
        SecR[SecurityReviewer]
        AW[ADRWriter]
        OW[OpenAPIWriter]
        SW[SpecWriter]
        PW[PlanWriter]
        TFW[TestFlowWriter]
        T[Tracer]
        E[Explainer]
        EA[EvidenceAnalyzer]
        ERW[EvidenceReportWriter]
        DocS[DocSyncer]
        AO[AgentOptimizer]
        MA[MdAuthor]
    end

    User -->|skill| Orchestrator
    Orchestrator --> Haiku
    Orchestrator --> Sonnet
    Haiku --> Sonnet
    Sonnet --> Output[Resultado ao Usuário]
```

**Disciplina de contexto:** os agentes nunca recebem o output bruto das etapas anteriores. Cada repasse é condensado em no máximo 20 linhas — reduzindo o custo de tokens e mantendo cada agente focado no seu contrato de entrada.

---

## Skills

### `/coordinator` — Ciclo completo de desenvolvimento

O comando principal. Executa o pipeline completo da detecção ao código revisado.

**Entrada:** Descrição da tarefa em linguagem natural.

**Fluxo:**

```mermaid
stateDiagram-v2
    [*] --> Detecção
    Detecção --> Arquitetura: resumo do stack
    Arquitetura --> GateSpec: STOP — aguarda aprovação do dev

    GateSpec --> Bloqueado_SemSpec: spec não encontrada
    GateSpec --> Bloqueado_Rascunho: spec existe mas não aprovada
    GateSpec --> Implementação: spec aprovada ✅

    Implementação --> Verificação: build + testes
    Verificação --> Implementação: falha (máx. 1 ciclo)
    Verificação --> RevisãoParalela: testes passam
    RevisãoParalela --> ReviewCoordinator: relatório consolidado
    ReviewCoordinator --> Implementação: blockers encontrados (máx. 2 ciclos)
    ReviewCoordinator --> TransiçãoSpec: sem blockers
    TransiçãoSpec --> [*]: spec → implemented
```

| Etapa | Agente(s) | Gate |
|-------|-----------|------|
| 1. Detecção | Detector | — |
| 2. Arquitetura + ADR | Architect → ADRWriter | **STOP** — dev aprova; ADR gerado em `/docs/adr/` |
| 3. Verificação da spec | — | **BLOQUEIO** se não houver spec aprovada |
| 4. Implementação | Implementer | — |
| 3.5. Build + testes | — | **BLOQUEIO** se falhar (máx. 1 ciclo de correção) |
| 5. Revisão | Reviewer + PerformanceReviewer + TestReviewer + SpecReviewer + SecurityReviewer *(paralelo)* → ReviewCoordinator | — |
| 6. Transição de status | — | Spec marcada como `implemented` se sem blockers |

**Saída:** Código implementado + relatório de revisão unificado + spec `implemented` + ADR da decisão arquitetural.

---

### `/refinement` — Spec e plano de implementação (sem código)

Executado antes do `/coordinator`. Gera uma especificação formal e um plano de implementação para aprovação do desenvolvedor.

**Entrada:** Descrição da tarefa.

**Fluxo:**
1. **Detecção** — Detector identifica o stack
2. **Perguntas abertas** — 5 perguntas estruturadas são levantadas (comportamento em falha, idempotência, transições de estado, contratos, decisões diferidas) → **STOP, aguarda respostas**
3. **Análise arquitetural** — Architect avalia opções em modo refinement
4. **Escrita da spec** — SpecWriter produz `/specs/[slug].md` com status `draft` → **STOP, aguarda aprovação**
5. **Síntese do plano** — PlanWriter gera plano de implementação

**Saída:** `/specs/[slug].md` (draft → aprovada) + plano de implementação.

---

### `/discovery` — Exploração do codebase existente

Explora o codebase relevante a uma tarefa e produz um relatório de discovery antes de qualquer decisão arquitetural ou implementação.

**Entrada:** Descrição da tarefa ou área de interesse.

**Fluxo:**
1. **Detecção** — Detector identifica o stack
2. **Inventário** — CodeExplorer enumera arquivos, módulos e símbolos relevantes
3. **Síntese** — DiscoverySynthesizer mapeia modelo de domínio, gaps e constraints

**Saída:** Relatório de discovery em `.claude/discovery/[slug].md` com modelo de domínio, gaps identificados e constraints relevantes.

---

### `/review` — Revisão de código apenas

Revisa código existente ou um diff sem acionar a implementação.

**Entrada:** Lista de arquivos, diff ou `.claude/diff.md` detectado automaticamente.

**Fluxo:**
1. Detecção
2. Seis revisores executam **em paralelo**: Reviewer, PerformanceReviewer, Architect (modo review), TestReviewer, SpecReviewer, SecurityReviewer
3. ReviewCoordinator consolida em um único relatório deduplicado

**Saída:** Relatório de revisão unificado com findings categorizados por severidade.

---

### `/investigate` — Análise de log e stack trace

Reconstrói o caminho de execução que produziu uma entrada de log ou exceção.

**Entrada:** Log bruto ou stack trace.

**Fluxo:**
1. **LogClassifier** — extrai dados estruturados do log bruto
2. **Detector + Architect** *(paralelo)* — identifica o stack e candidatos de entry point
3. **Tracer** — reconstrói a call chain com níveis de confiança
4. **Explainer** — produz uma narrativa legível e um veredito diagnóstico

**Saída:** Narrativa explicativa + veredito (`PROBLEMA_REAL` / `COMPORTAMENTO_ESPERADO` / `COMPORTAMENTO_SUSPEITO` / `INCONCLUSIVO`).

---

### `/security-review` — Revisão de segurança focada em API

Revisa código com foco exclusivo em segurança, sem acionar os demais revisores.

**Entrada:** Lista de arquivos, diff ou `.claude/diff.md` detectado automaticamente.

**Fluxo:**
1. Detecção — Detector identifica o stack
2. SecurityReviewer analisa três vetores de comunicação:
   - **Endpoints HTTP** — autorização por objeto, autenticação, exposição de dados, mass assignment, rate limiting, CORS/headers
   - **Banco de dados** — injection, credenciais hardcoded, queries sem parametrização, dados excessivos
   - **Serviços terceiros** — SSRF, API keys no código, resposta externa sem validação de schema, chamadas sem timeout
3. Relatório entregue em português com referências OWASP API Security Top 10 (2023) e CWE por finding

**Saída:** Relatório de segurança com findings categorizados por severidade (🔴 Blocker / 🟡 Importante / 🟢 Sugestão) e nível de confiança [HIGH/MEDIUM/LOW].

---

### `/evidence` — Validação de evidências de teste

Valida logs de execução de testes contra a spec e o plano aprovados, gerando um documento formal de evidências.

**Entrada:** Logs brutos de execução de testes + slug da spec.

**Fluxo:**
1. **EvidenceParser** — processa logs brutos extraindo sinais estruturados (em segmentos se > 200 linhas)
2. Carrega spec e plano aprovados de `/specs/`
3. **Paralelo (3×)** — EvidenceAnalyzer mapeia evidências para: cenários / invariantes+contratos / NFR+plano
4. **EvidenceReportWriter** — consolida as três análises paralelas em documento final
5. Apresenta resultado ao usuário

**Saída:** Documento de evidências em `.claude/evidence/[slug]-[YYYY-MM-DD].md` com cobertura por seção da spec.

---

### `/adr` — Architecture Decision Record

Persiste a decisão arquitetural aprovada no Step 2 do `/coordinator` como um documento versionado.

**Entrada:** Contexto da decisão (automático do coordinator, ou manual).

**Fluxo:**
1. Determina o próximo ID sequencial lendo `/docs/adr/`
2. ADRWriter gera documento com: contexto, opções consideradas, decisão e consequências
3. Salva em `/docs/adr/NNNN-slug.md`

**Saída:** `/docs/adr/NNNN-slug.md` com status `accepted`.

---

### `/spec-status` — Dashboard de specs

Exibe o estado atual de todas as specs em `/specs/` sem invocar subagentes.

**Entrada:** Nenhuma.

**Saída:**

```
✅ Implemented (N)   — specs com código entregue
🟡 Approved (N)      — aguardando implementação (⚠️ se > 7 dias paradas)
🔵 Pending approval  — rascunho enviado para aprovação
📝 Draft (N)         — em elaboração
```

---

### `/openapi` — Gerador de contrato OpenAPI

Gera ou atualiza uma especificação OpenAPI 3.0 a partir dos handlers HTTP existentes, sem modificar o código-fonte.

**Entrada:** Arquivo(s) de handler/route.

**Fluxo:**
1. Detector identifica stack e framework HTTP
2. OpenAPIWriter lê handlers e infere: paths, métodos, schemas de request/response, parâmetros, auth
3. Se `/docs/openapi.yaml` já existe: merge — preserva descrições manuais, atualiza schemas alterados
4. Salva em `/docs/openapi.yaml`

**Saída:** `/docs/openapi.yaml` criado ou atualizado + lista de endpoints documentados.

---

### `/diff` — Gerar diff do git

**Entrada:** Branch atual.

**Fluxo:** Detecta a branch base (`origin/main`, `origin/master`, fallbacks locais) → executa `git diff <base>...HEAD` → salva em `.claude/diff.md`.

**Saída:** `.claude/diff.md` com diff completo e resumo das alterações.

---

### `/commit` — Mensagem de commit semântico

**Entrada:** Arquivos staged.

**Fluxo:** Analisa as mudanças staged → infere o tipo (`feat`, `fix`, `test`, `docs`, `chore`, `ci`, `build`, `style`) → gera mensagem em inglês imperativo e minúsculo → **STOP, aguarda aprovação do dev** → cria o commit.

**Saída:** Commit criado com mensagem semântica.

---

### `/sync-docs` — Sincronização do README de documentação

Sincroniza o README do repositório de documentação com o inventário atual de agentes e skills.

**Entrada:** Nenhuma (lê `.claude/agents/*.md` e `.claude/skills/*/SKILL.md` automaticamente).

**Fluxo:**
1. Lê todos os arquivos em `.claude/agents/` — monta inventário de agentes
2. Lê todos os arquivos em `.claude/skills/*/SKILL.md` — monta inventário de skills
3. Lê o README atual do repositório de documentação
4. DocSyncer reescreve as seções desatualizadas (Arquitetura, Skills, Referência de Agentes)
5. Escreve o README atualizado e reporta as alterações

**Saída:** README atualizado com contagens, diagrama e tabelas refletindo o estado atual do workflow.

---

### `/usage` — Análise de consumo de tokens

**Entrada:** `.claude/usage.log` (gerado automaticamente pelo hook de observabilidade).

**Fluxo:** Lê o log → parseia linhas no formato `TIMESTAMP | COMMAND | AGENT | in:N | out:N | cost:$N` → calcula consumo por agente e por comando → identifica gargalos → emite recomendação.

**Saída:** Tabela de consumo por agente e por comando + diagnóstico de gargalos + recomendação principal.

---

### `/testflow` — Geração de fluxo de teste E2E

Gera um documento de fluxo de teste end-to-end concreto a partir de uma spec aprovada e seu plano de implementação.

**Entrada:** Slug da spec (ex.: `minha-feature`).

**Fluxo:**
1. Resolve o slug e valida que a spec e o plano existem em `.claude/specs/`
2. TestFlowWriter gera o documento de fluxo de teste E2E com cenários concretos e passos de execução
3. Salva em `.claude/specs/[slug].testflow.md`

**Saída:** `.claude/specs/[slug].testflow.md` com fluxo de teste E2E baseado na spec e no plano aprovados.

---

### `/mr-watch` — Monitoramento de Merge Requests

Monitora MRs abertos no GitLab e notifica sobre novos comentários e threads de diff no terminal.

**Entrada:** Nenhuma (lê `~/.claude/mr-watch-config.json` automaticamente).

**Fluxo:**
1. Lê arquivo de configuração em `~/.claude/mr-watch-config.json`
2. Busca MRs abertos no projeto configurado
3. Faz polling indefinido por novos comentários e threads de diff — **Ctrl+C para encerrar**

**Saída:** Notificações em tempo real no terminal para novos comentários e threads em MRs abertos.

---

### `/agent-optimizer` — Auditoria de agentes e skills

Audita todos os agentes e skills para conformidade com output limits, input contracts, context-passing discipline e SDD.

**Entrada:** Nenhuma (lê `.claude/agents/*.md` e `.claude/skills/*/SKILL.md` automaticamente).

**Fluxo:**
1. Busca documentação Anthropic em paralelo
2. Monta inventário estruturado de todos os agentes e skills
3. AgentOptimizer executa auditoria e produz relatório em `.claude/discovery/`

**Saída:** Relatório de auditoria em `.claude/discovery/` com findings categorizados por tipo de violação.

---

### `/workflow-coordinator` — Criação e edição de artefatos de workflow

Cria ou atualiza um arquivo de agente, skill ou hook Go a partir de uma spec aprovada.

**Entrada:** Spec aprovada descrevendo o artefato a criar ou modificar.

**Fluxo:**
1. **Spec Gate** — bloqueia se não houver spec aprovada
2. Identifica o tipo de artefato (agente, skill, hook Go) e os caminhos de destino
3. MdAuthor cria ou edita cada artefato
4. MdValidator valida cada artefato contra o Convention Contract
5. Escreve os arquivos e sincroniza o registry

**Saída:** Artefato(s) criado(s) ou atualizado(s) + validação de conformidade com o Convention Contract.

---

## Referência de Agentes

| Agente | Modelo | Responsabilidade | Entrada | Saída |
|--------|--------|------------------|---------|-------|
| **Detector** | Haiku | Identifica linguagem, versão do runtime, bibliotecas principais e convenções do projeto | Manifests (`package.json`, `go.mod`, etc.) | Resumo do stack (≤ 20 linhas) |
| **CodeExplorer** | Haiku | Enumera arquivos, módulos e símbolos relevantes a uma tarefa — sem síntese, apenas inventário | Descrição da tarefa + estrutura do projeto | Lista de arquivos e símbolos relevantes |
| **EvidenceParser** | Haiku | Extrai sinais estruturados de logs de execução de testes — sem análise, apenas extração | Logs brutos de testes (processado em segmentos se > 200 linhas) | Sinais estruturados de teste (passes, failures, erros) |
| **LogClassifier** | Haiku | Extrai estrutura objetiva de logs brutos (tipo de erro, tokens do stack, contexto da requisição) | Log bruto ou stack trace | Log estruturado (≤ 12 linhas) |
| **ReviewCoordinator** | Haiku | Consolida até 6 outputs de revisores em um único relatório deduplicado | Até 6 outputs de revisores | Relatório unificado com findings priorizados |
| **Architect** | Sonnet | Avalia tradeoffs arquiteturais em três modos: design, review, trace | Resumo do stack + descrição da tarefa | Opções com riscos e avaliação de complexidade |
| **DiscoverySynthesizer** | Sonnet | Sintetiza relatório de discovery a partir do inventário do CodeExplorer; mapeia modelo de domínio, gaps e constraints | Inventário do CodeExplorer + resumo do stack | Relatório de discovery com modelo de domínio, gaps e constraints |
| **DocSyncer** | Sonnet | Reescreve seções do README de documentação com base no inventário atual de agentes e skills | Inventário de agentes e skills + README atual | README atualizado |
| **EvidenceAnalyzer** | Sonnet | Mapeia evidências de teste para uma seção da spec (cenários, invariantes/contratos ou NFR/plano) — chamado 3× em paralelo | Sinais estruturados + seção da spec correspondente | Análise de cobertura para a seção |
| **EvidenceReportWriter** | Sonnet | Consolida as três saídas paralelas do EvidenceAnalyzer em documento final de evidências | Três análises do EvidenceAnalyzer | Documento formal de evidências |
| **Explainer** | Sonnet | Traduz um trace técnico em narrativa legível e veredito diagnóstico | Trace + log original + stack | Narrativa + veredito |
| **Implementer** | Sonnet | Escreve código de acordo com a spec aprovada e a decisão arquitetural | Stack + arquitetura + spec aprovada | Código + relatório de conformidade à spec |
| **OpenAPIWriter** | Sonnet | Infere paths, schemas, parâmetros e auth a partir de handlers HTTP — agnóstico de framework; merge seguro se spec já existe | Handlers + resumo do stack | YAML OpenAPI 3.0 |
| **PerformanceReviewer** | Sonnet | Detecta problemas de performance (O(n²), queries N+1, memory leaks, etc.) | Diff/código + resumo do stack | Problemas com níveis de confiança |
| **PlanWriter** | Sonnet | Escreve planos de implementação após aprovação da spec — o "como", complementar ao spec que é o "o quê" | Spec aprovada + decisão arquitetural | Plano de implementação (≤ 80 linhas) |
| **Reviewer** | Sonnet | Valida DRY, KISS e Clean Code — não SOLID | Diff ou arquivos modificados | Violações categorizadas por severidade |
| **SecurityReviewer** | Sonnet | Analisa vulnerabilidades de segurança em três vetores: endpoints HTTP, acesso a banco de dados e integrações com serviços terceiros — agnóstico de stack, baseado no OWASP API Security Top 10 (2023) e CWEs relevantes | Diff/arquivos + resumo do stack | Findings por vetor com severidade, referência OWASP/CWE e remediação |
| **SpecReviewer** | Sonnet | Valida conformidade da implementação com a spec: cenários, invariantes, contratos | Spec aprovada + código implementado | Relatório de conformidade por elemento da spec |
| **SpecWriter** | Sonnet | Produz uma especificação formal em Gherkin com cenários, invariantes e contratos de output | Tarefa + stack + decisão arquitetural + respostas do dev | `/specs/[slug].md` (status: draft → aprovada) |
| **TestReviewer** | Sonnet | Valida qualidade dos testes: cobertura, ghost tests, acoplamento comportamento vs. implementação | Testes + código modificado | Análise de cobertura e qualidade |
| **Tracer** | Sonnet | Reconstrói a sequência de execução de código que produziu um log | Log classificado + stack + candidatos de entry point | Call chain com nível de confiança |
| **ADRWriter** | Sonnet | Gera Architecture Decision Records com contexto, opções, decisão e consequências; auto-incrementa o ID lendo `/docs/adr/` | Contexto da decisão + opções + escolha | `/docs/adr/NNNN-slug.md` |
| **AgentOptimizer** | Sonnet | Audita agentes e skills para output limits, input contracts, context-passing discipline e conformidade com SDD | Inventário de agentes e skills + documentação Anthropic | Relatório de auditoria com findings categorizados |
| **MdAuthor** | Sonnet | Cria ou edita arquivos markdown de agentes e skills a partir de uma spec aprovada, aplicando o Convention Contract | Spec aprovada + conteúdo atual do arquivo (se edição) | Arquivo markdown criado ou atualizado |
| **MdValidator** | Haiku | Valida um artefato markdown contra as sete regras do Convention Contract — retorna pass/fail com detalhes das violações | Arquivo markdown | Resultado pass/fail com lista de violações |
| **TestFlowWriter** | Sonnet | Gera documento de fluxo de teste E2E concreto a partir de uma spec aprovada e seu plano de implementação | Spec aprovada + plano de implementação | `.claude/specs/[slug].testflow.md` |

---

## Princípios de Design

**Spec-first.** O `/coordinator` é bloqueado se não houver spec aprovada. O gate de spec não é opcional — existe para evitar implementar a coisa errada.

**Humano no loop nos pontos de decisão.** Aprovações de arquitetura e spec exigem sign-off explícito do desenvolvedor antes de o pipeline continuar.

**Disciplina de custo.** Nenhum agente recebe output bruto de uma etapa anterior. Cada repasse é condensado (≤ 20 linhas). Agentes sem dependência entre si executam em paralelo.

**Crítico, não complacente.** Os agentes são instruídos a sinalizar problemas reais — inclusive em código produtivo e de alto valor. "Funciona" não é motivo para omitir um finding. A crítica é direta e honesta.

**Separação de responsabilidades.** Cada agente tem uma função. O Implementer não revisa. O Reviewer não implementa. O Architect não escreve código.

---

## Observabilidade — Rastreamento de Tokens

### O que é um hook?

Um **hook** é um script executado automaticamente pelo Claude Code em resposta a eventos do ciclo de vida — sem intervenção manual. É configurado em `settings.local.json` e dispara sempre que o evento correspondente ocorre.

| Evento | Quando dispara |
|--------|---------------|
| `PostToolUse` | Após cada tool terminar |
| `PreToolUse` | Antes de executar uma tool |
| `Stop` | Quando Claude termina de responder |
| `Notification` | Quando Claude emite uma notificação |

### Hook de uso de tokens

O workflow registra um hook `PostToolUse` com matcher `Task` que grava automaticamente em `.claude/usage.log` após cada subagente terminar:

```
2026-03-26T21:00:01Z | coordinator | Reviewer        | in:4200 | out:890  | cost:$0.0042
2026-03-26T21:00:01Z | coordinator | PerfReviewer    | in:3800 | out:720  | cost:$0.0038
2026-03-26T21:00:02Z | coordinator | TestReviewer    | in:4100 | out:650  | cost:$0.0039
2026-03-26T21:00:02Z | coordinator | SpecReviewer    | in:3900 | out:610  | cost:$0.0037
```

### `/usage` — Análise de gargalos

Após acumular execuções no log, rode `/usage` para obter:

- **Consumo por agente** — tabela ordenada por custo total (invocações, tokens entrada/saída, custo médio)
- **Consumo por comando** — agrupado por `/coordinator`, `/review`, `/investigate`
- **Diagnóstico de gargalos:**
  - Tokens de entrada altos → contexto inflado sendo repassado (violação da regra ≤ 20 linhas)
  - Output excessivo → agente gerando mais do que o necessário para o próximo passo
  - Muitas chamadas ao Implementer → spec com bloqueadores recorrentes
- **Recomendação principal** — uma linha direta com o gargalo e a ação sugerida

---

## Quick Start

```bash
# Dentro do diretório de qualquer projeto
curl -sO https://raw.githubusercontent.com/Leock9/flow-copilot/main/.claude/setup-claude.sh
bash setup-claude.sh
```

O script de setup copia `CLAUDE.md`, `agents/` e `skills/` para o diretório `.claude/` do projeto — o local que o Claude Code lê para instruções no nível do projeto.

### Requisitos

- CLI do [Claude Code](https://claude.ai/code) instalado e autenticado
- Permissões da tool `Task` habilitadas (necessário para invocar subagentes)

### Fluxo recomendado para uma nova feature

```
/refinement   →   revisar spec   →   /coordinator   →   /commit
```

1. `/refinement` — responda as perguntas abertas, aprove a spec
2. `/coordinator` — decisão arquitetural → implementação → ciclo de revisão (inclui SecurityReviewer automaticamente)
3. `/commit` — aprove a mensagem de commit semântico

Para revisão de segurança isolada em código existente:

```
/security-review [arquivo(s) ou diff]
```

Para verificar o estado de todas as features em andamento:

```
/spec-status
```

---

## Fonte

Os prompts dos agentes e a lógica de orquestração dos comandos estão em [`Leock9/flow-copilot`](https://github.com/Leock9/flow-copilot) dentro de `.claude/`.
