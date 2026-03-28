# engineer-workflow-claude

> Um workflow de desenvolvimento multi-agente e spec-first para o [Claude Code](https://claude.ai/code) — transformando a descrição de uma tarefa em código funcional e revisado através de um pipeline estruturado de agentes especializados.

![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)

---
<img width="855" height="506" alt="image" src="https://github.com/user-attachments/assets/45ff66c7-87f7-44ba-9267-b22b24e201aa" />

## Visão Geral

Este projeto implementa um **pipeline de agentes orquestrado** sobre o Claude Code. Em vez de pedir a um único modelo que detecte, projete, implemente e revise tudo de uma vez, cada responsabilidade é decomposta em um subagente especializado — cada um com um papel focado, um modelo específico e uma entrada condensada.

O workflow impõe uma **disciplina spec-first**: nenhum código é escrito até que uma decisão arquitetural e uma especificação formal tenham sido revisadas e aprovadas pelo desenvolvedor. Isso cria um gate humano no loop que impede construir a coisa errada antes que as perguntas certas sejam respondidas.

Isto não é um framework e não substitui o desenvolvedor. É uma camada de processo estruturada — um conjunto de comandos e agentes que trazem ordem, rigor e consistência à forma como o Claude Code é usado em um projeto de software.

---

## Arquitetura

O sistema é composto por **15 subagentes especializados** orquestrados por **10 comandos**. Os agentes são atribuídos a modelos de acordo com a complexidade da tarefa:

```mermaid
graph TD
    subgraph Haiku["⚡ Haiku — Estrutural / Rápido"]
        D[Detector]
        LC[LogClassifier]
        RC[ReviewCoordinator]
    end

    subgraph Sonnet["🧠 Sonnet — Análise / Código"]
        A[Architect]
        I[Implementer]
        RV[Reviewer]
        PR[PerformanceReviewer]
        TR[TestReviewer]
        SR[SpecReviewer]
        SecR[SecurityReviewer]
        AW[ADRWriter]
        OW[OpenAPIWriter]
        SW[SpecWriter]
        T[Tracer]
        E[Explainer]
    end

    User -->|comando| Orchestrator
    Orchestrator --> Haiku
    Orchestrator --> Sonnet
    Haiku --> Sonnet
    Sonnet --> Output[Resultado ao Usuário]
```

**Disciplina de contexto:** os agentes nunca recebem o output bruto das etapas anteriores. Cada repasse é condensado em no máximo 20 linhas — reduzindo o custo de tokens e mantendo cada agente focado no seu contrato de entrada.

---

## Comandos

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
5. **Síntese do plano** — Plano de implementação gerado (≤ 80 linhas)

**Saída:** `/specs/[slug].md` (draft → aprovada) + plano de implementação.

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

## Referência de Agentes

| Agente | Modelo | Responsabilidade | Entrada | Saída |
|--------|--------|------------------|---------|-------|
| **Detector** | Haiku | Identifica linguagem, versão do runtime, bibliotecas principais e convenções do projeto | Manifests (`package.json`, `go.mod`, etc.) | Resumo do stack (≤ 20 linhas) |
| **Architect** | Sonnet | Avalia tradeoffs arquiteturais em três modos: design, review, trace | Resumo do stack + descrição da tarefa | Opções com riscos e avaliação de complexidade |
| **SpecWriter** | Sonnet | Produz uma especificação formal em Gherkin com cenários, invariantes e contratos de output | Tarefa + stack + decisão arquitetural + respostas do dev | `/specs/[slug].md` (status: draft → aprovada) |
| **Implementer** | Sonnet | Escreve código de acordo com a spec aprovada e a decisão arquitetural | Stack + arquitetura + spec aprovada | Código + relatório de conformidade à spec |
| **Reviewer** | Sonnet | Valida DRY, KISS e Clean Code — não SOLID | Diff ou arquivos modificados | Violações categorizadas por severidade |
| **PerformanceReviewer** | Sonnet | Detecta problemas de performance (O(n²), queries N+1, memory leaks, etc.) | Diff/código + resumo do stack | Problemas com níveis de confiança |
| **TestReviewer** | Sonnet | Valida qualidade dos testes: cobertura, ghost tests, acoplamento comportamento vs. implementação | Testes + código modificado | Análise de cobertura e qualidade |
| **SpecReviewer** | Sonnet | Valida conformidade da implementação com a spec: cenários, invariantes, contratos | Spec aprovada + código implementado | Relatório de conformidade por elemento da spec |
| **SecurityReviewer** | Sonnet | Analisa vulnerabilidades de segurança em três vetores: endpoints HTTP, acesso a banco de dados e integrações com serviços terceiros — agnóstico de stack, baseado no OWASP API Security Top 10 (2023) e CWEs relevantes | Diff/arquivos + resumo do stack | Findings por vetor com severidade, referência OWASP/CWE e remediação |
| **ADRWriter** | Sonnet | Gera Architecture Decision Records com contexto, opções, decisão e consequências; auto-incrementa o ID lendo `/docs/adr/` | Contexto da decisão + opções + escolha | `/docs/adr/NNNN-slug.md` |
| **OpenAPIWriter** | Sonnet | Infere paths, schemas, parâmetros e auth a partir de handlers HTTP — agnóstico de framework; merge seguro se spec já existe | Handlers + resumo do stack | YAML OpenAPI 3.0 |
| **ReviewCoordinator** | Haiku | Consolida até 6 outputs de revisores em um único relatório deduplicado | Até 6 outputs de revisores | Relatório unificado com findings priorizados |
| **LogClassifier** | Haiku | Extrai estrutura objetiva de logs brutos (tipo de erro, tokens do stack, contexto da requisição) | Log bruto ou stack trace | Log estruturado (≤ 12 linhas) |
| **Tracer** | Sonnet | Reconstrói a sequência de execução de código que produziu um log | Log classificado + stack + candidatos de entry point | Call chain com nível de confiança |
| **Explainer** | Sonnet | Traduz um trace técnico em narrativa legível e veredito diagnóstico | Trace + log original + stack | Narrativa + veredito |

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

O script de setup copia `CLAUDE.md`, `agents/` e `commands/` para o diretório `.claude/` do projeto — o local que o Claude Code lê para instruções no nível do projeto.

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
