# EPT-2127 — Estudo: OpenSpec, AGENTS.md e Context Engineering para Repositórios de Plataforma

**Card**: [EPT-2127](https://fxsolutions.atlassian.net/browse/EPT-2127)
**Autor**: Osório Santos
**Data**: Maio 2026
**Escopo**: Módulos Terraform (`ebb-terraform-gcp-*`), Cloud Functions (`ebb-governance-clean`) e demais repos do time

---

## Índice

1. [Contexto: O Problema que Queremos Resolver](#1-contexto-o-problema-que-queremos-resolver)
2. [Parte 1: OpenSpec](#2-parte-1-openspec)
   - [O que é](#21-o-que-é-o-openspec)
   - [Conceitos Fundamentais](#22-conceitos-fundamentais)
   - [Como Funciona na Prática](#23-como-funciona-na-prática)
   - [Workflow Completo](#24-workflow-completo)
   - [Customização com Schemas](#25-customização-com-schemas)
   - [Aplicação em Módulos Terraform](#26-openspec-aplicado-a-módulos-terraform)
   - [Aplicação em Cloud Functions](#27-openspec-aplicado-a-cloud-functions)
   - [Quando o OpenSpec Faz Sentido](#28-quando-o-openspec-faz-sentido)
   - [Prós e Contras](#29-prós-e-contras-do-openspec)
   - [Veredicto](#210-veredicto-openspec)
3. [Parte 2: AGENTS.md e VS Code Agent Primitives](#3-parte-2-agentsmd-e-vs-code-agent-primitives)
   - [O que é o AGENTS.md](#31-o-que-é-o-agentsmd)
   - [Como Funciona](#32-como-funciona)
   - [Estrutura e Formato](#33-estrutura-e-formato)
   - [VS Code Agent Primitives](#34-vs-code-agent-primitives-o-ecossistema-completo)
   - [Como Gerar um AGENTS.md](#35-como-gerar-um-agentsmd)
   - [Aplicação em Módulos Terraform](#36-agentsmd-aplicado-a-módulos-terraform)
   - [Aplicação em Cloud Functions (caso real)](#37-agentsmd-aplicado-a-cloud-functions-caso-real)
   - [Prós e Contras](#38-prós-e-contras-do-agentsmd)
   - [Veredicto](#39-veredicto-agentsmd)
4. [Comparação Final: OpenSpec vs AGENTS.md](#4-comparação-final-openspec-vs-agentsmd)
   - [Evidência Prática: Experimento com PRs](#41-evidência-prática-experimento-com-prs)
5. [Proposta e Plano de Implementação](#5-proposta-e-plano-de-implementação)
6. [Conclusão](#6-conclusão)
7. [Referências](#7-referências)

---

## 1. Contexto: O Problema que Queremos Resolver

### Situação Atual

O time de Plataforma possui **12 módulos Terraform**, **Cloud Functions**, repositórios de IAC e pipelines de CI/CD, gerenciados por **mais de 10 pessoas**. Cada desenvolvedor interage com assistentes de IA (GitHub Copilot, Claude, ChatGPT, etc.) sem nenhum contexto compartilhado sobre:

- Convenções de código e padrões do time
- Naming de recursos, labels obrigatórias, estrutura de diretórios
- Design decisions tomadas no passado e seus motivos
- Pitfalls conhecidos de APIs GCP
- Como adicionar ou modificar recursos seguindo o padrão existente

**O resultado**: cada pessoa recebe sugestões diferentes da IA, sem consistência. Prompts são vagos porque a IA não tem contexto sobre nossos padrões. Isso gera retrabalho em code review e inconsistência entre ambientes.

### O que Queremos

1. **Padronização**: qualquer pessoa do time, usando qualquer IA, recebe sugestões que seguem nossos padrões
2. **Redução de custo com tokens**: contexto focado por repositório em vez de carregar informação irrelevante
3. **Zero friction**: a solução deve ser automática — o dev não precisa executar comandos nem instalar nada

### Perfil dos Nossos Repositórios

Antes de analisar as ferramentas, é importante entender o que gerenciamos:

| Tipo | Repos | Tamanho Médio | Linguagem | Tipo de Mudança |
|------|-------|--------------|-----------|-----------------|
| Módulos Terraform | 12 (`ebb-terraform-gcp-*`) | ~200 linhas de `.tf` (49 a 433) | HCL | Adicionar variáveis, ajustar defaults, validações |
| Cloud Functions | Vários (`ebb-governance-clean`, etc.) | ~1.750 linhas | Python | Adicionar recursos, ajustar regras, melhorar reports |
| IAC Resources | 1 (`ebb-iac-resource`) | Grande (multi-domínio) | HCL/Terragrunt | Criar recursos em 3 ambientes |
| IAM Roles | 1 (`ebb-iac-iams`) | Médio | HCL/Terragrunt | Configurar permissões |

**Nenhum destes repositórios** possui atualmente qualquer customização de IA (exceto o `ebb-governance-clean`, que já implementou um sistema completo como prova de conceito).

---

## 2. Parte 1: OpenSpec

### 2.1 O que é o OpenSpec

O [OpenSpec](https://github.com/Fission-AI/OpenSpec) é um framework open source criado pela Fission AI para **Spec-Driven Development (SDD)**. Ele adiciona uma camada de especificação entre a intenção do desenvolvedor e o código, forçando que humano e IA concordem sobre **o que** será construído antes de escrever qualquer linha.

A filosofia do OpenSpec é:

```
→ fluid not rigid         — sem fases rígidas, trabalhe no que fizer sentido
→ iterative not waterfall — aprenda enquanto constrói, refine conforme avança
→ easy not complex        — setup leve, mínima cerimônia
→ brownfield-first        — funciona com codebases existentes, não só projetos novos
```

**Dados do projeto**: 48K stars no GitHub, 65 contribuidores, MIT license, v1.3.1 (maio 2026). Suporta 25+ ferramentas de IA (Copilot, Claude Code, Cursor, Codex, etc.).

**Instalação**:

```bash
npm install -g @fission-ai/openspec@latest
cd seu-projeto
openspec init
```

O `openspec init` cria uma pasta `openspec/` no repositório e configura skills/instruções para a IA que você usa.

---

### 2.2 Conceitos Fundamentais

#### Specs (Source of Truth)

Specs descrevem **como o sistema se comporta hoje**. São organizados por domínio em `openspec/specs/`:

```
openspec/specs/
├── auth/
│   └── spec.md           # Comportamento de autenticação
├── payments/
│   └── spec.md           # Processamento de pagamentos
└── notifications/
    └── spec.md           # Sistema de notificações
```

Cada spec contém **requirements** (o que o sistema faz) e **scenarios** (exemplos concretos em formato Given/When/Then):

```markdown
# Auth Specification

## Purpose
Authentication and session management for the application.

## Requirements

### Requirement: User Authentication
The system SHALL issue a JWT token upon successful login.

#### Scenario: Valid credentials
- GIVEN a user with valid credentials
- WHEN the user submits login form
- THEN a JWT token is returned
- AND the user is redirected to dashboard

#### Scenario: Invalid credentials
- GIVEN invalid credentials
- WHEN the user submits login form
- THEN an error message is displayed
- AND no token is issued
```

**Palavras-chave RFC 2119**:
- `MUST` / `SHALL` — requisito absoluto
- `SHOULD` — recomendado, mas exceções existem
- `MAY` — opcional

**Importante**: Um spec descreve **comportamento**, não implementação. Se a implementação pode mudar sem alterar o comportamento visível, não pertence ao spec.

#### Changes (Mudanças Propostas)

Cada mudança é empacotada em uma pasta dentro de `openspec/changes/`:

```
openspec/changes/add-dark-mode/
├── proposal.md           # POR QUE e O QUE (intent, scope, approach)
├── specs/                # Delta specs — O QUE muda nos specs existentes
│   └── ui/
│       └── spec.md       # ADDED/MODIFIED/REMOVED requirements
├── design.md             # COMO implementar (approach técnico, decisions)
└── tasks.md              # Checklist de implementação
```

**Por que pastas?**
1. **Tudo junto**: Proposal, design, tasks e specs em um único lugar
2. **Trabalho paralelo**: Múltiplas changes podem coexistir sem conflito
3. **Histórico limpo**: Ao concluir, a change vai para `archive/` com todo o contexto
4. **Fácil de revisar**: Abre a pasta, lê o proposal, verifica os deltas

#### Delta Specs (o conceito mais importante)

Delta specs descrevem **o que muda** em relação ao spec atual, usando seções:

```markdown
# Delta for Auth

## ADDED Requirements

### Requirement: Two-Factor Authentication
The system MUST support TOTP-based two-factor authentication.

#### Scenario: 2FA enrollment
- GIVEN a user without 2FA enabled
- WHEN the user enables 2FA in settings
- THEN a QR code is displayed for authenticator app setup

## MODIFIED Requirements

### Requirement: Session Expiration
The system MUST expire sessions after 15 minutes of inactivity.
(Previously: 30 minutes)

## REMOVED Requirements

### Requirement: Remember Me
(Deprecated in favor of 2FA. Users should re-authenticate each session.)
```

| Seção | Significado | No archive |
|-------|------------|------------|
| `## ADDED Requirements` | Comportamento novo | Adicionado ao spec principal |
| `## MODIFIED Requirements` | Comportamento alterado | Substitui o requirement existente |
| `## REMOVED Requirements` | Comportamento removido | Deletado do spec principal |

**Por que deltas em vez de specs completos?**
- **Clareza**: Mostra exatamente o que muda, sem diff mental
- **Sem conflitos**: Duas changes podem tocar o mesmo spec se modificam requirements diferentes
- **Review eficiente**: Revisor vê a mudança, não o contexto inalterado
- **Ideal para brownfield**: A maioria do trabalho é modificar comportamento existente

#### Artifacts (Documentos de Planejamento)

Cada change contém até 4 artifacts, que formam uma cadeia:

```
proposal ──────► specs ──────► design ──────► tasks ──────► implement
    │               │             │              │
   why            what           how          steps
 + scope        changes       approach      to take
```

**Proposal (`proposal.md`)** — Captura intenção, escopo e approach:

```markdown
# Proposal: Add Dark Mode

## Intent
Users have requested a dark mode option to reduce eye strain.

## Scope
In scope:
- Theme toggle in settings
- System preference detection
- Persist preference in localStorage

Out of scope:
- Custom color themes (future work)
- Per-page theme overrides

## Approach
Use CSS custom properties for theming with a React context for state management.
```

**Design (`design.md`)** — Captura approach técnico e decisions:

```markdown
# Design: Add Dark Mode

## Technical Approach
Theme state managed via React Context.

## Architecture Decisions

### Decision: Context over Redux
Using React Context because:
- Simple binary state (light/dark)
- No complex state transitions
- Avoids adding Redux dependency

## File Changes
- `src/contexts/ThemeContext.tsx` (new)
- `src/components/ThemeToggle.tsx` (new)
- `src/styles/globals.css` (modified)
```

**Tasks (`tasks.md`)** — Checklist de implementação:

```markdown
# Tasks

## 1. Theme Infrastructure
- [ ] 1.1 Create ThemeContext with light/dark state
- [ ] 1.2 Add CSS custom properties for colors
- [ ] 1.3 Implement localStorage persistence

## 2. UI Components
- [ ] 2.1 Create ThemeToggle component
- [ ] 2.2 Add toggle to settings page
```

#### Archive (Finalização)

Quando uma change é concluída:

1. **Merge deltas**: Cada seção ADDED/MODIFIED/REMOVED é aplicada ao spec principal
2. **Move para archive**: A pasta da change vai para `changes/archive/YYYY-MM-DD-nome/`
3. **Preserva contexto**: Todos os artifacts ficam intactos no archive para consulta futura

```
Antes do archive:                    Depois do archive:

openspec/                            openspec/
├── specs/                           ├── specs/
│   └── auth/                        │   └── auth/
│       └── spec.md ◄──merge──┐      │       └── spec.md  ← atualizado com 2FA
└── changes/                  │      └── changes/
    └── add-2fa/              │          └── archive/
        ├── proposal.md       │              └── 2025-01-24-add-2fa/
        ├── design.md         │                  ├── proposal.md
        ├── tasks.md          │                  ├── design.md
        └── specs/            │                  ├── tasks.md
            └── auth/         │                  └── specs/auth/spec.md
                └── spec.md ──┘
```

---

### 2.3 Como Funciona na Prática

O desenvolvedor interage com o OpenSpec via **slash commands** na IA:

| Comando | O que faz |
|---------|-----------|
| `/opsx:propose "descrição"` | Cria pasta de change + gera todos os artifacts de planejamento |
| `/opsx:explore` | Investigação antes de propor (para requisitos unclear) |
| `/opsx:apply` | IA implementa as tasks, marcando como concluídas |
| `/opsx:verify` | IA valida se implementação bate com specs |
| `/opsx:sync` | Merge deltas nos specs principais |
| `/opsx:archive` | Finaliza change: sync + move para archive |

**Exemplo de sessão completa**:

```
Dev: /opsx:propose add-dark-mode
IA:  Created openspec/changes/add-dark-mode/
     ✓ proposal.md — why we're doing this, what's changing
     ✓ specs/       — requirements and scenarios
     ✓ design.md    — technical approach
     ✓ tasks.md     — implementation checklist
     Ready for implementation!

Dev: /opsx:apply
IA:  Implementing tasks...
     ✓ 1.1 Add theme context provider
     ✓ 1.2 Create toggle component
     ✓ 2.1 Add CSS variables
     ✓ 2.2 Wire up localStorage
     All tasks complete!

Dev: /opsx:archive
IA:  Archived to openspec/changes/archive/2025-01-23-add-dark-mode/
     Specs updated. Ready for the next feature.
```

Existe também o **expanded workflow** (para mais controle):

| Comando | O que faz |
|---------|-----------|
| `/opsx:new nome` | Cria apenas o scaffold da change (pasta vazia) |
| `/opsx:continue` | Cria o próximo artifact (passo a passo) |
| `/opsx:ff` | Cria todos os artifacts de uma vez (fast-forward) |
| `/opsx:verify` | Valida implementação (completeness, correctness, coherence) |
| `/opsx:bulk-archive` | Arquiva múltiplas changes de uma vez |

---

### 2.4 Workflow Completo

```
┌──────────────────────────────────────────────────────────────────────┐
│                          OPENSPEC FLOW                               │
│                                                                      │
│   ┌────────────────┐                                                 │
│   │  1. PROPOSE    │  /opsx:propose "descrição da mudança"           │
│   │                │  Cria change + artifacts de planejamento         │
│   └───────┬────────┘                                                 │
│           │                                                          │
│           ▼                                                          │
│   ┌────────────────┐                                                 │
│   │  2. REVIEW     │  Dev revisa proposal, specs, design, tasks      │
│   │     ARTIFACTS  │  Ajusta se necessário (iterativo)               │
│   └───────┬────────┘                                                 │
│           │                                                          │
│           ▼                                                          │
│   ┌────────────────┐                                                 │
│   │  3. IMPLEMENT  │  /opsx:apply                                    │
│   │     TASKS      │  IA executa tasks, marca como feitas            │
│   │                │◄──── Pode voltar e atualizar artifacts          │
│   └───────┬────────┘                                                 │
│           │                                                          │
│           ▼                                                          │
│   ┌────────────────┐                                                 │
│   │  4. VERIFY     │  /opsx:verify                                   │
│   │                │  Checa completeness, correctness, coherence     │
│   └───────┬────────┘                                                 │
│           │                                                          │
│           ▼                                                          │
│   ┌────────────────┐     ┌──────────────────────────────────────┐    │
│   │  5. ARCHIVE    │────►│  Deltas merge nos specs principais   │    │
│   │                │     │  Change vai para archive/             │    │
│   └────────────────┘     │  Specs = novo source of truth        │    │
│                          └──────────────────────────────────────┘    │
│                                                                      │
│   Ciclo: specs → changes → implement → archive → specs atualizados   │
└──────────────────────────────────────────────────────────────────────┘
```

**Trabalho paralelo**: Múltiplas changes podem existir simultaneamente. Cada uma é uma pasta isolada. Isso permite que um dev trabalhe em `add-dark-mode` enquanto outro trabalha em `fix-auth-bug`, sem conflitos.

---

### 2.5 Customização com Schemas

Schemas definem quais artifacts existem e suas dependências. O schema padrão é:

```yaml
# openspec/schemas/spec-driven/schema.yaml
name: spec-driven
artifacts:
  - id: proposal
    generates: proposal.md
    requires: []              # Pode ser criado primeiro

  - id: specs
    generates: specs/**/*.md
    requires: [proposal]      # Precisa do proposal

  - id: design
    generates: design.md
    requires: [proposal]      # Pode ser paralelo com specs

  - id: tasks
    generates: tasks.md
    requires: [specs, design] # Precisa de ambos
```

**Custom schemas** são possíveis. Exemplo de schema leve para infraestrutura:

```yaml
# openspec/schemas/iac-change/schema.yaml
name: iac-change
artifacts:
  - id: proposal
    generates: proposal.md
    requires: []

  - id: tasks
    generates: tasks.md
    requires: [proposal]
# Sem specs, sem design — direto ao ponto
```

---

### 2.6 OpenSpec Aplicado a Módulos Terraform

Agora vamos aplicar tudo que vimos ao nosso caso concreto. Nossos 12 módulos Terraform possuem:

| Característica | Valor |
|---------------|-------|
| Quantidade | 12 repositórios (`ebb-terraform-gcp-*`) |
| Tamanho médio | ~200 linhas de `.tf` (49 a 433 linhas) |
| Estrutura | `main.tf` + `variables.tf` + `outputs.tf` + `examples/` + `README.md` |
| Tipo de mudança típica | Adicionar variáveis, novo recurso, ajustar defaults, corrigir validações |
| AI customization hoje | **Nenhum** (só `.github/workflows/`) |

#### Como ficaria com OpenSpec

```
ebb-terraform-gcp-pubsub/
├── main.tf                     # 130 linhas
├── variables.tf                # 80 linhas
├── outputs.tf                  # 50 linhas
├── examples/                   # Exemplos de uso
├── README.md                   # Auto-gerado por terraform-docs
├── openspec/                   # ← ADICIONADO
│   ├── specs/
│   │   └── pubsub/
│   │       └── spec.md         # "O módulo DEVE suportar topics com schemas..."
│   │                           # "O módulo DEVE validar labels obrigatórias..."
│   │                           # Scenario: "GIVEN topic sem labels WHEN plan THEN error"
│   └── changes/
│       └── add-cloud-storage-sub/
│           ├── proposal.md     # "Adicionar suporte a Cloud Storage subscriptions"
│           ├── specs/
│           │   └── pubsub/
│           │       └── spec.md # "ADDED: variáveis bucket, filename_prefix..."
│           ├── design.md       # "Novo dynamic block cloud_storage_config..."
│           └── tasks.md        # "[ ] Adicionar variáveis [ ] Criar dynamic block..."
```

#### Por que NÃO se encaixa

**1. O `variables.tf` já é a spec.**

O `variables.tf` do módulo PubSub já contém exatamente o que um spec.md descreveria:

```hcl
variable "pubsub_topics" {
  type = list(object({
    project                          = string              # ← tipo documentado
    name                             = string              # ← campo obrigatório
    schema                           = optional(string, "") # ← opcional com default
    encoding                         = optional(string, "JSON")
    topic_message_retention_duration = optional(string, "604800s")
    kms_key_name                     = optional(string, "")
    labels                           = optional(map(string), {})
  }))
  default = []
}
```

Um `spec.md` do OpenSpec diria: _"O módulo DEVE aceitar uma variável `topic_message_retention_duration` do tipo string com default `604800s`"_. Isso **já está declarado** no código. É **duplicação pura** que precisa ser mantida manualmente em sincronia.

**2. O `design.md` não tem o que dizer.**

Módulos Terraform são **declarativos por natureza**. O "design" é o próprio código. Dizer "usar um dynamic block para `cloud_storage_config`" no `design.md` é descrever o que vai estar no `main.tf`. Não existe decisão arquitetural — existe um padrão estabelecido (`for_each`, `dynamic`, region `southamerica-east1`).

**3. O overhead é desproporcional ao tamanho.**

Um módulo de 200 linhas teria uma pasta `openspec/` com 4+ arquivos de planejamento. Para adicionar uma variável (que é a mudança mais comum), o dev precisaria:

```
/opsx:propose "adicionar variável vpc_connector ao Cloud Run"
→ Gera proposal.md, specs/, design.md, tasks.md
→ Dev revisa 4 arquivos
→ /opsx:apply (adiciona 3 linhas no variables.tf e 5 no main.tf)
→ /opsx:archive
```

Sem OpenSpec, o mesmo trabalho é: abrir o repo, pedir ao Copilot, fazer PR. **1 minuto vs 10 minutos.**

**4. O custo de manter specs sincronizados não compensa.**

Cada vez que alguém adiciona uma variável em `variables.tf`, o spec correspondente precisa ser atualizado (ou o archive precisa ser feito). Se alguém esquece de fazer archive, os specs ficam desatualizados e perdem valor. Em repos com mudanças pontuais e infrequentes, esse custo se acumula sem retorno.

---

### 2.7 OpenSpec Aplicado a Cloud Functions

O `ebb-governance-clean` é uma Cloud Function com lógica de negócio real:

| Característica | Valor |
|---------------|-------|
| Tamanho | ~1.750 linhas de Python (13 arquivos) |
| Tipo | Cloud Function Gen 2 (Python 3.12) |
| Complexidade | Média — governance rules, grace periods, multi-project, email reports |
| Tipo de mudança | Adicionar tipo de recurso, ajustar regras, melhorar reports |

#### Como ficaria com OpenSpec

```
ebb-governance-clean/
├── openspec/
│   ├── specs/
│   │   ├── governance-rules/
│   │   │   └── spec.md
│   │   │   # Requirement: Resource Detection
│   │   │   # The system SHALL mark resources without label createdby=terraform
│   │   │   # Scenario: GIVEN recurso sem label WHEN scan executa THEN marca com deletion_date
│   │   │   #
│   │   │   # Requirement: Grace Period
│   │   │   # The system SHALL respect configurable grace period per project
│   │   │   # Scenario: GIVEN deletion_days=30 WHEN resource marked THEN deletion_date = today + 30
│   │   │
│   │   ├── notifications/
│   │   │   └── spec.md
│   │   │   # Requirement: Daily Alerts
│   │   │   # The system SHALL send consolidated email per project
│   │   │   # Requirement: Weekly Reports
│   │   │   # The system SHALL send weekly report on Fridays
│   │   │
│   │   └── resource-types/
│   │       └── spec.md
│   │       # Requirement: Supported Resources
│   │       # The system SHALL support: Compute, Cloud SQL, Pub/Sub, Cloud Functions, Cloud Run
│   │
│   └── changes/
│       └── add-artifact-registry/
│           ├── proposal.md
│           ├── specs/resource-types/spec.md   # ADDED: Artifact Registry
│           ├── design.md
│           └── tasks.md
├── main.py
├── resources/
└── ...
```

#### O que funciona bem

Para **mudanças complexas** — como "mudar o grace period para ser por tipo de recurso em vez de por projeto" — o OpenSpec traria valor real:

- O **`proposal.md`** forçaria o dev a pensar no impacto antes de codar
- O **delta spec** deixaria explícito: _"MODIFIED: grace period agora é `Dict[resource_type, int]` em vez de `int`"_
- O **`design.md`** documentaria mudanças em `config.py`, `labels.py` e formato do `PROJECTS_CONFIG`
- O **`verify`** validaria que todos os `process_*` foram atualizados

#### O que NÃO funciona

Para a **mudança mais comum** — "adicionar novo tipo de recurso" — o overhead é desproporcional:

| Com OpenSpec | Sem OpenSpec (hoje) |
|-------------|-------------------|
| `propose` → revisar proposal | `/add-resource-type artifact-registry` |
| Revisar delta specs | IA segue o template, cria tudo |
| Revisar design | Dev faz PR |
| `apply` → IA implementa | |
| `archive` → merge specs | |
| **5 steps, 4 arquivos extras** | **1 comando, zero overhead** |

O repositório já possui um **`.prompt.md`** que faz o scaffold completo de um novo recurso (criar módulo em `resources/`, adicionar snapshot, wiring no `main.py`, atualizar `requirements.txt`). Esse prompt é **mais eficiente que o OpenSpec** para essa tarefa porque é específico e otimizado para o caso de uso.

#### O fator tamanho

Com ~1.750 linhas, o `ebb-governance-clean` está na fronteira. O OpenSpec é recomendado para apps grandes (5k+ linhas) onde a IA pode "esquecer" requirements durante implementações longas. Neste repo, o contexto inteiro cabe na janela da IA sem problemas.

---

### 2.8 Quando o OpenSpec Faz Sentido

Para ser justo, o OpenSpec foi projetado para cenários específicos. Aqui estão os casos onde ele brilha:

| Cenário | Por que OpenSpec brilha |
|---------|----------------------|
| **Apps grandes (5k+ linhas)** com múltiplos fluxos e features | Specs previnem que IA "esqueça" requirements durante implementações longas |
| **APIs com contratos** (REST, gRPC, event schemas) | Delta specs capturam mudanças de contrato com precisão |
| **Times distribuídos** onde PR diff não basta para entender o "porquê" | Proposal + design documentam intent e decisões |
| **Features que levam semanas** com múltiplas iterações | Changes organizam trabalho em progresso, suportam parallel work |
| **Projetos greenfield** sem padrões definidos | Specs criam o contrato comportamental antes do código |
| **Múltiplos stakeholders** que precisam revisar intent vs código | Proposal é legível para não-devs |

**Nossos repositórios de Plataforma não se encaixam nestes cenários**: são pequenos (50-1750 linhas), têm padrões bem definidos, mudanças são de escopo limitado, Terraform é declarativo por natureza, e o time é co-localizado.

---

### 2.9 Prós e Contras do OpenSpec

#### Prós

| Vantagem | Detalhe |
|----------|---------|
| **Spec-first thinking** | Força planejamento antes de implementação. Reduz retrabalho |
| **Delta specs** | Conceito poderoso — mostra exatamente o que muda sem diff mental |
| **Histórico de decisões** | O archive preserva por que cada mudança foi feita |
| **Parallel work** | Múltiplas changes coexistem sem conflito |
| **Review de intent** | Revisores entendem a mudança sem ler código |
| **Universal** | Suporta 25+ ferramentas de IA |
| **Customizável** | Schemas permitem adaptar o workflow ao time |
| **Open source + ativo** | 48K stars, 65 contribuidores, releases frequentes |

#### Contras

| Desvantagem | Impacto para nós |
|-------------|-----------------|
| **Overhead para mudanças pequenas** | 4 arquivos de planejamento para adicionar 1 variável Terraform |
| **Duplicação com Terraform** | `spec.md` replica o que `variables.tf` já define |
| **Manutenção de specs** | Se alguém esquece de fazer archive, specs ficam desatualizados |
| **Requer Node.js** | Instalação global de npm package + `openspec init` por repo |
| **Learning curve** | Time de 10+ pessoas precisa aprender workflow novo |
| **Custo em tokens** | ~2000+ tokens por interação (specs + artifacts + instruções do framework) |
| **Não resolve nosso problema principal** | O problema é falta de contexto para IA, não falta de specs |
| **Workspaces multi-repo em beta** | Feature de coordenação entre repos ainda não está pronta |

---

### 2.10 Veredicto OpenSpec

> **Não adotar OpenSpec nos repositórios de Plataforma.**
>
> O OpenSpec é uma ferramenta inovadora com conceitos valiosos (delta specs, artifact-guided workflow, archive), mas foi projetado para codebases de aplicação grandes com lógica complexa. Para nossos módulos Terraform e Cloud Functions:
>
> - O Terraform já é **declarativo por natureza** — `variables.tf` é a spec
> - As mudanças são **pequenas e padronizadas** — o overhead de planejamento não se paga
> - O **tamanho dos repos** (50-1750 linhas) não justifica uma camada extra de especificação
> - O **problema real** é falta de contexto para IA, não falta de specs
>
> **Recomendação**: Adotar a **filosofia** do OpenSpec (pensar antes de codar, documentar intent, review de comportamento vs código) sem a ferramenta. Os conceitos de delta specs e spec-first thinking podem ser incorporados em PR descriptions e code review.

---

## 3. Parte 2: AGENTS.md e VS Code Agent Primitives

### 3.1 O que é o AGENTS.md

O `AGENTS.md` é um arquivo Markdown que, quando commitado na raiz de um repositório, é **automaticamente injetado no contexto** de toda interação com assistentes de IA que suportam o padrão. Não precisa de instalação, configuração, CLI, nem ação do desenvolvedor.

É um **open standard** com suporte crescente. Atualmente funciona com:
- GitHub Copilot (VS Code, JetBrains, Neovim)
- Claude (via Claude Code)
- Cursor
- E outros que adotam o padrão

**O mecanismo é simples**: quando alguém abre o repositório e faz qualquer pergunta ao Copilot, o conteúdo do `AGENTS.md` é carregado automaticamente no contexto da conversa. A IA "lê" o arquivo e segue as instruções ao gerar código, responder perguntas e fazer sugestões.

**Alternativa**: O `copilot-instructions.md` (em `.github/`) faz a mesma coisa, mas é específico do GitHub Copilot. O `AGENTS.md` é o open standard (funciona com múltiplas ferramentas) e suporta **hierarquia em subpastas** (ideal para monorepos).

> **Regra**: Use **apenas um** dos dois — `AGENTS.md` ou `.github/copilot-instructions.md` — nunca ambos no mesmo repo.

---

### 3.2 Como Funciona

```
Dev abre o repo no VS Code
       │
       ▼
Copilot detecta AGENTS.md na raiz
       │
       ▼
Conteúdo é injetado no contexto de TODA interação
       │
       ▼
Dev pergunta: "adiciona suporte a Cloud Armor"
       │
       ▼
Copilot responde seguindo as convenções do AGENTS.md:
  ✓ Usa for_each (não count)
  ✓ Adiciona labels obrigatórias
  ✓ Segue naming convention
  ✓ Gera validações no variables.tf
```

**Zero friction**: O dev não precisa executar nenhum comando, instalar nada, nem lembrar de invocar o AGENTS.md. Basta abrir o repo e trabalhar normalmente.

**Hierarquia**: Em workspaces multi-folder ou monorepos, cada subpasta pode ter seu próprio `AGENTS.md`:

```
workspace/
├── AGENTS.md              # Regras gerais do workspace
├── modulo-a/
│   ├── AGENTS.md          # Regras específicas do módulo A (complementa o pai)
│   └── main.tf
└── modulo-b/
    ├── AGENTS.md          # Regras específicas do módulo B (complementa o pai)
    └── main.tf
```

O `AGENTS.md` da subpasta **complementa** (não substitui) o do nível acima. Isso permite contexto genérico no topo e específico por componente.

---

### 3.3 Estrutura e Formato

O `AGENTS.md` é Markdown puro, sem frontmatter, sem formato rígido. A estrutura recomendada é:

```markdown
# Agent Instructions — nome-do-repo

## Project Overview
Descrição concisa do projeto: o que faz, tecnologia, runtime.

## Quick Commands
Comandos de build, test, deploy — para a IA saber executar.

## Architecture
Tabela ou descrição da estrutura de arquivos e responsabilidades.

## Conventions
Padrões que a IA DEVE seguir ao gerar código neste repo.

## Key Design Decisions
Decisões tomadas e por quê — para a IA não reverter.

## Pitfalls
Erros conhecidos e armadilhas que a IA deve evitar.
```

**Princípios de escrita**:

1. **Mínimo e focado**: Somente o que é relevante para TODA interação com IA neste repo
2. **Conciso e acionável**: Cada linha deve guiar comportamento
3. **Link, não embuta**: Referencie docs existentes em vez de copiar conteúdo
4. **Atualize**: Quando práticas mudam, atualize o AGENTS.md

**Anti-patterns**:

| Evitar | Por que |
|--------|---------|
| Copiar o README inteiro | Duplicação sem valor adicional |
| Regras já cobertas por linters | Redundante — o linter já garante |
| Conteúdo genérico ("escreva código limpo") | Não guia comportamento específico |
| Arquivo gigante (500+ linhas) | Consome tokens excessivos sem retorno |

---

### 3.4 VS Code Agent Primitives: O Ecossistema Completo

O `AGENTS.md` é a base, mas o VS Code oferece **3 primitivos adicionais** que formam um sistema completo de context engineering:

#### Instructions (`.github/instructions/*.instructions.md`)

Regras que se aplicam **automaticamente** quando a IA edita arquivos que batem com um glob pattern.

```markdown
---
applyTo: "**/*.py"
description: "Python conventions for the GCP governance Cloud Function."
---

# Python GCP Governance Conventions

- Import config via `from config import *` e labels via `from labels import *`.
- Resource functions: `process_<service>(project_id, deletion_days, known_resources)` → `(marked, deleted)`.
- Each resource dict: `{"name", "kind", "location", "labels", "deletion_date"}`.
- Wrap per-resource ops in try/except — one failure must never abort the loop.
- Never inline label logic — use `labels.py` helpers.
```

**Diferença do AGENTS.md**: O AGENTS.md é carregado SEMPRE. Instructions são carregadas **somente quando relevantes** (ao editar `.py`, `.tf`, etc.). Isso economiza tokens.

#### Prompts (`.github/prompts/*.prompt.md`)

Templates de tarefas reutilizáveis com inputs parametrizados. O dev invoca com `/nome-do-prompt`.

```markdown
---
description: "Scaffold a new GCP resource type module for the governance function."
---

# Add Resource Type: {{ resource_name }}

1. Create `resources/{{ resource_name }}.py` with `process_{{ resource_name }}()` function
2. Add `get_all_marked_{{ resource_name }}()` in `resources/snapshot.py`
3. Wire into `main.py`: import + processing block + extend lists
4. Add GCP client library to `requirements.txt`
```

**Uso**: Dev digita `/add-resource-type resource_name=artifact-registry` no chat e a IA executa todos os passos automaticamente.

#### Custom Agents (`.github/agents/*.agent.md`)

Personas especializadas com tools restritas e comportamento focado.

```markdown
---
description: "Use when: validating Terraform modules, checking conventions, reviewing variables."
tools: [read, search]
---

You are a Terraform module reviewer. Check:
1. All resources use `for_each` (never `count`)
2. Mandatory labels are validated
3. `dynamic` blocks follow the pattern
4. README.md is auto-generated marker exists
```

| Primitivo | Quando carrega | Uso |
|-----------|---------------|-----|
| `AGENTS.md` | Sempre | Contexto geral do projeto |
| `.instructions.md` | Ao editar arquivos matching | Convenções por linguagem/tipo de arquivo |
| `.prompt.md` | Quando invocado (`/nome`) | Tarefas repetitivas parametrizadas |
| `.agent.md` | Quando invocado ou delegado | Workflows especializados com tools restritas |

---

### 3.5 Como Gerar um AGENTS.md

#### Passo a passo para qualquer repositório

**1. Identifique o essencial** — Responda estas perguntas:

| Pergunta | Seção correspondente |
|----------|---------------------|
| O que este repo faz, em 2-3 linhas? | `## Overview` |
| Quais os comandos para rodar/testar/deploy? | `## Quick Commands` |
| Qual a estrutura de arquivos e quem faz o quê? | `## Architecture` |
| Quais padrões o time segue que não são óbvios? | `## Conventions` |
| Que decisões foram tomadas e por quê? | `## Key Design Decisions` |
| Quais erros a IA comete frequentemente neste repo? | `## Pitfalls` |

**2. Escreva conciso** — Exemplo de seção `Conventions` ruim vs boa:

```markdown
# ❌ RUIM — genérico, não acionável
## Conventions
- Write clean code
- Follow best practices
- Use meaningful variable names

# ✅ BOM — específico, acionável
## Conventions
- All resources use `for_each` on the input list (never `count`).
- Region is hardcoded to `southamerica-east1` via `message_storage_policy`.
- README.md is auto-generated by `terraform-docs` — do not edit manually.
- PR title format: `type(scope): description [EPT-XXXX]`
```

**3. Dimensione** — Guia de tamanho:

| Tamanho do repo | AGENTS.md ideal | Tokens |
|----------------|----------------|--------|
| Pequeno (<500 linhas) | 30-50 linhas (~1KB) | ~250-400 |
| Médio (500-2000 linhas) | 50-100 linhas (~2KB) | ~400-700 |
| Grande (2000+ linhas) | 80-150 linhas (~3KB) | ~600-1000 |

**4. Valide** — Depois de criar, teste:
- Abra o repo no VS Code
- Peça ao Copilot uma tarefa comum (ex: "adiciona suporte a X")
- Verifique se a resposta segue as convenções do AGENTS.md
- Ajuste o que a IA ignorou ou interpretou errado

**5. Itere** — O AGENTS.md é um documento vivo. Atualize quando:
- Uma convenção muda
- Um novo pitfall é descoberto
- O Copilot insiste em gerar código fora do padrão (adicione regra explícita)

---

### 3.6 AGENTS.md Aplicado a Módulos Terraform

#### O que existe hoje nos 12 módulos

```
ebb-terraform-gcp-*/
├── .github/
│   └── workflows/         # ← Só isso. Nenhum contexto para IA.
├── main.tf
├── variables.tf
├── outputs.tf
├── examples/
├── README.md
├── CHANGELOG.md
├── lefthook.yml
└── pyproject.toml
```

**Resultado atual**: Qualquer pessoa que pedir ao Copilot "adiciona suporte a BigQuery subscription no módulo PubSub" recebe código genérico que:
- Pode usar `count` em vez de `for_each`
- Não sabe das labels obrigatórias
- Não segue o naming convention (`ebb-`)
- Não sabe que o README é auto-gerado

#### O que ficaria com AGENTS.md

```
ebb-terraform-gcp-pubsub/
├── AGENTS.md                # ← 1 arquivo, ~1.5KB, ~400 tokens
├── .github/
│   └── workflows/
├── main.tf
├── variables.tf
├── outputs.tf
├── examples/
└── README.md
```

**Conteúdo do AGENTS.md** (exemplo para o módulo PubSub):

```markdown
# Agent Instructions — ebb-terraform-gcp-pubsub

## Overview
Terraform module for GCP Pub/Sub resources: topics, subscriptions, and schemas.
Versioned via git tags (semver). Consumed by `ebb-iac-resource` via Terragrunt.

## Quick Commands
```bash
terraform fmt           # Format
terraform validate      # Validate syntax
terraform-docs .        # Regenerate README.md
```

## Structure
| File | Responsibility |
|------|---------------|
| `main.tf` | Resource definitions (`google_pubsub_topic`, `_subscription`, `_schema`) |
| `variables.tf` | Input contract: types, defaults, validations |
| `outputs.tf` | Exported values for consumers |
| `examples/` | Usage examples for Terragrunt consumers |
| `README.md` | **Auto-generated** by terraform-docs — do NOT edit manually |

## Conventions
- All resources use `for_each` on the input list variable (never `count`).
- Region hardcoded to `southamerica-east1` via `message_storage_policy`.
- Optional blocks use `dynamic` with conditional `for_each`.
- Variable defaults must be sensible for production use.
- New optional fields: use `optional(type, default)` syntax.

## Labels (Mandatory)
All resources requiring labels must validate these mandatory keys:
`team`, `environment`, `createdby`, `github_project`, `domain`, `name`,
`cost-center`, `feature_id`, `data-classification`, `compliance`,
`access-level`, `automation`, `shutdown-schedule`, `tier`.
Ref: https://bexs.atlassian.net/wiki/spaces/EP/pages/2670624781

## Versioning
- CHANGELOG.md follows Keep a Changelog format.
- Tags: semver (breaking = major, feature = minor, fix = patch).
- PR title: `type(scope): description [EPT-XXXX]`
- Commit: include Jira card `[EPT-XXXX]` — pipeline blocks PR without it.

## Pitfalls
- Pub/Sub labels use `update_topic`/`update_subscription` with field masks, not `set_labels`.
- Schema deletion requires all topics using it to be deleted first.
- `message_retention_duration` on subscriptions is string in seconds ("86400s").
- `message_storage_policy` cannot be empty — always set `allowed_persistence_regions`.
```

**O que muda para o dev**: Ao pedir "adiciona suporte a BigQuery subscription", o Copilot agora:
- Usa `for_each` (leu em Conventions)
- Adiciona o campo como `optional` com default sensato (leu em Conventions)
- Sabe que precisa do terraform-docs (leu em Structure)
- Gera o campo com tipo string e validação (leu o padrão existente)

**Esforço**: ~15 minutos por módulo para criar. O conteúdo é 80% igual entre módulos (mudam overview, pitfalls, e detalhes específicos do recurso GCP).

#### Template base para todos os módulos

Para acelerar, usar um template que varia apenas em 3 seções:

```markdown
# Agent Instructions — ebb-terraform-gcp-{RECURSO}

## Overview
Terraform module for GCP {RECURSO_NOME}: {DESCRIÇÃO}.
Versioned via git tags (semver). Consumed by `ebb-iac-resource` via Terragrunt.

## Quick Commands
{MESMOS PARA TODOS}

## Structure
{MESMA PARA TODOS — main.tf, variables.tf, outputs.tf, examples/, README.md}

## Conventions
{MESMAS PARA TODOS — for_each, dynamic, region, optional()}

## Labels (Mandatory)
{MESMO PARA TODOS}

## Versioning
{MESMO PARA TODOS}

## Pitfalls
{ESPECÍFICO POR MÓDULO — armadilhas da API GCP deste recurso}
```

---

### 3.7 AGENTS.md Aplicado a Cloud Functions (Caso Real)

O `ebb-governance-clean` já implementou o sistema completo como **prova de conceito**. Vamos analisar o que foi feito e como funciona.

#### O que existe hoje

```
ebb-governance-clean/
├── AGENTS.md                                        # ~75 linhas, ~500 tokens
├── .github/
│   ├── instructions/
│   │   └── python-gcp-conventions.instructions.md   # ~18 linhas, ~150 tokens
│   └── prompts/
│       └── add-resource-type.prompt.md              # Template parametrizado
├── main.py
├── config.py
├── labels.py
├── emailer.py
├── reporting.py
├── resources/
│   ├── compute.py
│   ├── cloudsql.py
│   ├── pubsub.py
│   ├── cloudfunctions.py
│   ├── cloudrun.py
│   └── snapshot.py
└── templates/
```

#### Camada 1: AGENTS.md — Contexto sempre presente

O AGENTS.md do governance-clean contém:

- **Overview**: Cloud Function Gen 2, Python 3.12, governance enforcement
- **Quick Commands**: `functions-framework`, `gcloud functions deploy`, `pip install`
- **Architecture**: Tabela com cada arquivo e sua responsabilidade
- **Conventions**: Signature de functions, error handling, labels.py usage, print format
- **Key Design Decisions**: Sem testes no repo, multi-project via JSON env var, weekly reports on Fridays
- **Adding a New Resource Type**: Passos 1-4 para adicionar um recurso (resumo)
- **Pitfalls**: `PROJECTS_CONFIG` JSON, Cloud SQL labels, Pub/Sub field masks, email template guide

**Resultado**: Em toda interação, a IA sabe que é uma Cloud Function Python, conhece a estrutura, sabe como adicionar recursos, e evita os pitfalls. Custo: ~500 tokens.

#### Camada 2: Instructions — Regras por linguagem

O `python-gcp-conventions.instructions.md` é ativado **somente quando o dev edita um `.py`**:

```markdown
---
applyTo: "**/*.py"
---
- process_<service>(project_id, deletion_days, known_resources) → (marked, deleted)
- Each resource dict: name, kind, location, labels + optional deletion_date
- Wrap per-resource in try/except
- Never inline label logic — use labels.py helpers
- Print: "  → " actions, "    ✓ " success, "    ⚠ " warnings, "    🗑️ " deletions
```

**Resultado**: Ao editar Python, a IA segue exatamente o padrão de signatures, error handling e print format. Custo: ~150 tokens (somente quando edita `.py`).

#### Camada 3: Prompts — Tarefas repetitivas

O `add-resource-type.prompt.md` é invocado pelo dev quando necessário:

```
Dev: /add-resource-type resource_name=artifact-registry

IA: (executa automaticamente)
  1. Cria resources/artifact_registry.py com process_artifact_registry()
  2. Adiciona get_all_marked_artifact_registry() em resources/snapshot.py
  3. Adiciona import + bloco de processamento em main.py
  4. Adiciona google-cloud-artifact-registry ao requirements.txt
```

**Resultado**: Uma tarefa que levaria 20+ minutos manualmente é executada em 1 comando, seguindo exatamente o padrão existente.

#### Por que este modelo funciona

| Peça | Quando ativa | Tokens | O que resolve |
|------|-------------|--------|--------------|
| `AGENTS.md` | **Sempre** | ~500 | IA sabe o que é o projeto, decisões, pitfalls |
| `.instructions.md` | **Ao editar `.py`** | ~150 | IA segue conventions de código |
| `.prompt.md` | **Quando invocado** | ~200 | Tarefa repetitiva automatizada |
| **Total** | | **~650-850** | Contexto completo para qualquer tarefa |

Compare com OpenSpec: ~2000+ tokens carregados a cada interação, independente da tarefa.

---

### 3.8 Prós e Contras do AGENTS.md

#### Prós

| Vantagem | Detalhe |
|----------|---------|
| **Zero setup** | Não precisa instalar nada — é um arquivo no repo |
| **Zero learning curve** | Dev não precisa aprender workflow novo |
| **Automático** | Carregado em toda interação, sem ação do dev |
| **Versionado** | Commitado no git, versionado com o código, compartilhado com o time |
| **Econômico em tokens** | 400-700 tokens vs 2000+ do OpenSpec |
| **Universal** | Funciona com Copilot, Claude, Cursor e outros |
| **Hierárquico** | Subpastas podem complementar o AGENTS.md pai |
| **Ecossistema completo** | Instructions + Prompts + Agents completam as lacunas |
| **Incremental** | Comece com AGENTS.md, adicione instructions/prompts conforme necessidade |
| **Caso real funcionando** | `ebb-governance-clean` já implementou e valida o modelo |

#### Contras

| Desvantagem | Mitigação |
|-------------|-----------|
| **Manutenção manual** | Atualizar quando convenções mudam — mas mudanças são infrequentes |
| **Sem spec-first workflow** | Não força planejamento antes de implementação — usar PR descriptions |
| **Sem histórico de decisões** | Não tem archive como OpenSpec — usar Jira/Confluence para historico |
| **Sem delta specs** | Mudanças de comportamento não são capturadas formalmente — usar PR diff |
| **Qualidade depende do autor** | AGENTS.md ruim = contexto ruim — usar template e review |
| **Não substitui documentação** | É contexto para IA, não documentação para humanos — manter README |

---

### 3.9 Veredicto AGENTS.md

> **Adotar AGENTS.md em todos os repositórios gerenciados pelo time de Plataforma.**
>
> É a solução que resolve exatamente o problema identificado (falta de contexto compartilhado para IA) com o menor custo de implementação e manutenção:
>
> - **Para módulos Terraform**: `AGENTS.md` com conventions, labels, pitfalls (~400 tokens)
> - **Para Cloud Functions**: `AGENTS.md` + `.instructions.md` + `.prompt.md` (~650 tokens)
> - **Para IAC Resources**: `AGENTS.md` com regras cross-env, naming, formatação Terragrunt
>
> O `ebb-governance-clean` já funciona como prova de conceito e referência de implementação.

---

## 4. Comparação Final: OpenSpec vs AGENTS.md

### Para Módulos Terraform

| Critério | OpenSpec | AGENTS.md |
|----------|----------|-----------|
| Setup por repo | `npm install -g` + `openspec init` | Criar 1 arquivo |
| Manutenção | Manter specs sincronizados com `variables.tf` | Atualizar quando convenções mudam |
| Overhead por mudança | 4+ arquivos de planejamento | Zero — automático |
| Duplicação | Alta — specs replicam `variables.tf` | Nenhuma — complementa |
| Adoção pelo time | Treinar 10+ pessoas | Zero learning curve |
| Tokens por interação | ~2000+ | ~400 |
| Valor | Baixo — Terraform já é declarativo | **Alto** — contexto essencial |

### Para Cloud Functions / Apps

| Critério | OpenSpec | AGENTS.md + Primitives |
|----------|----------|----------------------|
| Mudança simples (ex: novo recurso) | 5 steps, 4 arquivos extras | `/add-resource-type` (1 comando) |
| Mudança complexa | Bom — força planejamento | Bom — AGENTS.md dá contexto |
| Documentação viva | Specs evoluem com archive | Requer update manual |
| Onboarding | Dev lê specs | Dev lê AGENTS.md + README |
| Custo total | Alto (install + train + maintain) | Baixo (já implementado) |

### Custo × Benefício Geral

| Métrica | Sem nada (hoje) | Com OpenSpec | Com AGENTS.md |
|---------|-----------------|-------------|---------------|
| **Setup por repo** | 0 | ~30min | ~15min |
| **Manutenção mensal** | 0 | Alta | Baixa |
| **Tokens por interação** | 0 | ~2000+ | ~400-650 |
| **Qualidade das sugestões IA** | Baixa | Alta | Alta |
| **Consistência entre devs** | Nenhuma | Alta (se usarem) | Alta (automático) |
| **Learning curve** | 0 | Alta | Zero |
| **Funciona com qualquer AI** | — | Sim (25+) | Sim (Copilot, Claude, etc.) |
| **Tempo para adoção total** | — | Semanas | 1-2 dias |

### 4.1 Evidência Prática: Experimento com PRs

Para validar as conclusões teóricas, realizamos um experimento controlado no repositório `ebb-terraform-gcp-cloud-functions`. O **mesmo pedido** foi feito 3 vezes, variando apenas o contexto disponível para a IA:

> **Pedido**: "Adicionar suporte a VPC Connector no `service_config` para permitir que Cloud Functions acessem recursos em redes VPC privadas (Cloud SQL, Memorystore, APIs internas)."

Os resultados estão em 5 PRs abertos (marcados como `[DO NOT MERGE]`) para comparação direta:

#### Cenário A: OpenSpec (PRs #19 → #17 → #18)

O fluxo completo do OpenSpec requereu **3 PRs separados** correspondendo a 3 fases:

| PR | Branch | Fase | Arquivos | Linhas adicionadas |
|----|--------|------|----------|--------------------|
| [#19](https://github.com/Ebury-Brazil/ebb-terraform-gcp-cloud-functions/pull/19) | `openspec-proposal` | Propose | 14 arquivos | +1.375 |
| [#17](https://github.com/Ebury-Brazil/ebb-terraform-gcp-cloud-functions/pull/17) | `openspec-apply` | Apply | 5 arquivos | +15 |
| [#18](https://github.com/Ebury-Brazil/ebb-terraform-gcp-cloud-functions/pull/18) | `openspec-archive` | Archive | 6 arquivos | +34 |
| | | **Total** | **25 arquivos** | **+1.424** |

**O que aconteceu**: O `/opsx:propose` gerou toda a infraestrutura OpenSpec — `config.yaml`, 4 prompts (`.prompt.md`), 4 skills (`SKILL.md`), o proposal, design, specs e tasks — somando **1.375 linhas de scaffolding**. O `/opsx:apply` fez as mudanças reais no código (+15 linhas em `main.tf`, `variables.tf`, `example.tf`, `README.md`). O `/opsx:archive` moveu os artefatos para `archive/` e promoveu a spec.

**Qualidade do código gerado** (`main.tf`):
```hcl
vpc_connector                    = each.value.vpc_connector
vpc_connector_egress_settings    = each.value.vpc_connector_egress_settings
```
Passa os valores diretamente — sem guard clause. Se `vpc_connector` for `null`, envia `null` para a API (funciona, mas é menos explícito).

**Variáveis** (`variables.tf`):
```hcl
vpc_connector                    = optional(string, null)
vpc_connector_egress_settings    = optional(string, "PRIVATE_RANGES_ONLY")
```
Default `null` para connector, `"PRIVATE_RANGES_ONLY"` para egress. **Problema sutil**: se o usuário não definir `vpc_connector` (fica `null`), o valor `"PRIVATE_RANGES_ONLY"` ainda é enviado para a API — e a API do GCP **rejeita com erro 400** quando `vpc_connector_egress_settings` é definido sem um `vpc_connector` válido. A ausência de guard clause torna este código potencialmente defeituoso.

#### Cenário B: Sem contexto nenhum (PR #20)

| PR | Branch | Arquivos | Linhas adicionadas |
|----|--------|----------|--------------------|
| [#20](https://github.com/Ebury-Brazil/ebb-terraform-gcp-cloud-functions/pull/20) | `sem-contexto` | 3 arquivos | +11 |

**O que aconteceu**: A IA fez exatamente o que foi pedido — adicionou as variáveis e os atributos no `service_config`. Sem exemplo, sem docs.

**Qualidade do código** (`main.tf`):
```hcl
vpc_connector                    = each.value.vpc_connector
vpc_connector_egress_settings    = each.value.vpc_connector_egress_settings
```
Idêntico ao OpenSpec — sem guard clause.

**Variáveis** (`variables.tf`):
```hcl
vpc_connector                    = optional(string, null)
vpc_connector_egress_settings    = optional(string, null)
```
Default `null` para ambos. Na prática isso **funciona corretamente**: quando ambos são `null`, o Terraform omite os campos da API e a GCP aplica seu próprio default (`PRIVATE_RANGES_ONLY`) apenas quando um connector estiver presente. É uma abordagem segura, porém não explicita o comportamento esperado no código.

**O que faltou**: Não atualizou o exemplo em `examples/`, não atualizou o README. Fez o mínimo e parou — **não teve contexto para saber que esses arquivos precisam ser atualizados juntos**.

#### Cenário C: AGENTS.md (PR #21)

| PR | Branch | Arquivos | Linhas adicionadas |
|----|--------|----------|--------------------|
| [#21](https://github.com/Ebury-Brazil/ebb-terraform-gcp-cloud-functions/pull/21) | `with-agentsmd` | 4 arquivos | +72 (63 do AGENTS.md + 9 de código) |

**O que aconteceu**: Com o `AGENTS.md` descrevendo a arquitetura, convenções e estrutura do módulo, a IA produziu o resultado mais completo com **a menor interação**.

**Qualidade do código** (`main.tf`):
```hcl
vpc_connector                    = each.value.vpc_connector != "" ? each.value.vpc_connector : null
vpc_connector_egress_settings    = each.value.vpc_connector != "" ? each.value.vpc_connector_egress_settings : null
```
**Guard clause condicional** — só passa os valores se `vpc_connector` não for vazio. Consistente com o padrão existente do módulo (ex: `service_account_email`). **A IA entendeu as convenções do código porque o AGENTS.md as descreveu.**

**Variáveis** (`variables.tf`):
```hcl
vpc_connector                    = optional(string, "")
vpc_connector_egress_settings    = optional(string, "PRIVATE_RANGES_ONLY")
```
Default `""` (string vazia) para connector — consistente com o padrão existente de `service_account_email = optional(string, "")`. Default `"PRIVATE_RANGES_ONLY"` para egress — documenta explicitamente o padrão GCP.

**O que a IA fez a mais**: Atualizou o `examples/function-01-full/example.tf` com uso demonstrativo do VPC Connector, incluindo comentários. A **guard clause é crítica**: a API do GCP rejeita com erro 400 se `vpc_connector_egress_settings` for definido sem um `vpc_connector` válido. Somente o cenário C trata esse caso corretamente.

#### Resumo Comparativo dos 3 Cenários

| Métrica | OpenSpec (A) | Sem contexto (B) | AGENTS.md (C) |
|---------|:------------:|:-----------------:|:--------------:|
| **Arquivos gerados (scaffolding)** | 25 | 0 | 1 (`AGENTS.md`) |
| **Linhas adicionadas (total)** | 1.424 | 11 | 72 |
| **Linhas de código real** | 15 | 11 | 9 |
| **Atualizou `example.tf`** | Sim | Não | Sim |
| **Atualizou `README.md`** | Sim | Não | Não |
| **Guard clause no `main.tf`** | Não | Não | **Sim** |
| **Default egress seguro (sem risco de erro 400)** | Não — envia egress sem connector | Sim — `null` omite ambos | **Sim** — guard clause protege |
| **Segue convenções do módulo** | Parcial | Não | **Sim** |
| **PRs necessários** | 3 | 1 | 1 |
| **Interações com a IA** | 3+ (propose → apply → archive) | 1 | 1 |
| **Ratio overhead / código real** | **95:1** | **0:1** | **7:1** |

#### Conclusão do Experimento

1. **OpenSpec** produziu código com um **bug sutil**: o default `"PRIVATE_RANGES_ONLY"` para egress é enviado à API mesmo quando `vpc_connector` é `null`, o que causa erro 400 no `terraform apply`. Além disso, ratio de **95:1** entre scaffolding e código real. Para uma mudança de 15 linhas, criou 1.375 linhas de artefatos. O workflow forçado de 3 fases (propose → apply → archive) transformou uma tarefa de 5 minutos em algo que requer múltiplas iterações.

2. **Sem contexto** produziu código funcional e **tecnicamente seguro** (ambos `null` = omitidos da API), mas **incompleto**. A IA não sabia que `examples/` deve ser atualizado junto, não seguiu o padrão `optional(string, "")` do módulo. Fez o mínimo e parou.

3. **AGENTS.md** produziu o **código de maior qualidade** com a menor interação. A guard clause condicional previne o erro 400 da API, os defaults seguem o padrão do módulo, e a atualização do exemplo demonstra que **63 linhas de contexto permanente valem mais que 1.375 linhas de specs descartáveis**.

O experimento confirma a análise teórica: para repositórios do nosso perfil, **AGENTS.md oferece o melhor equilíbrio entre qualidade e eficiência**.

---

## 5. Proposta e Plano de Implementação

### Fase 1 — Módulos Terraform (12 repos)

**Esforço**: 1 dia
**Ação**: Criar `AGENTS.md` em cada módulo a partir de template base

```
Para cada módulo em ebb-terraform-gcp-*:
  1. Criar AGENTS.md a partir do template (80% igual)
  2. Customizar: overview (2 linhas), pitfalls (específicos do recurso GCP)
  3. Commitar na main via PR
```

**Template base**: Seções padronizadas (Structure, Conventions, Labels, Versioning) + seções customizáveis (Overview, Pitfalls).

### Fase 2 — Cloud Functions e Apps Python

**Esforço**: Variável (2-4h por repo)
**Ação**: Para repos com lógica de negócio, adicionar 3 camadas

```
repo/
├── AGENTS.md                                    # Sempre: overview, arch, decisions, pitfalls
├── .github/
│   ├── instructions/
│   │   └── <lang>-conventions.instructions.md   # Ativado ao editar arquivos da linguagem
│   └── prompts/
│       └── <tarefa-comum>.prompt.md             # Templates para tarefas repetitivas
```

**Referência**: Usar `ebb-governance-clean` como modelo.

### Fase 3 — Template de Repositório

**Esforço**: 2h
**Ação**: Integrar no `ebb-repository-templates` para novos repos nascerem com contexto

### Resumo de Ações

| Ação | Prioridade | Esforço |
|------|-----------|---------|
| Criar `AGENTS.md` nos 12 módulos Terraform | Alta | 1 dia |
| Documentar padrão do `ebb-governance-clean` como referência | Alta | 2h |
| Adicionar ao `ebb-repository-templates` | Média | 2h |
| Apresentar conceitos do OpenSpec como referência teórica | Baixa | Na apresentação |
| Reavaliar OpenSpec quando/se adotarmos apps maiores (Node/Go) | Futura | — |

---

## 6. Conclusão

### OpenSpec

Ferramenta inovadora com conceitos valiosos — delta specs, artifact-guided workflow, archive com histórico de decisões. Porém, projetada para codebases de aplicação grandes com lógica complexa. Para nossos repositórios de Plataforma (módulos Terraform de 50-433 linhas, Cloud Functions de ~1.750 linhas), o overhead de manutenção supera o benefício. **O Terraform é declarativo por natureza — `variables.tf` é a spec.**

**Recomendação**: Adotar a **filosofia** (spec-first thinking, review de intent) sem a ferramenta. Reavaliar se o time passar a gerenciar aplicações maiores.

### AGENTS.md + VS Code Primitives

Solução prática, zero friction, que resolve exatamente o problema: **padronizar o contexto que toda IA recebe ao trabalhar em nossos repositórios**. Um arquivo por repo, commitado no git, funcionando automaticamente para todos. O caso real do `ebb-governance-clean` comprova o modelo.

**Recomendação**: Adotar imediatamente nos 12 módulos Terraform e expandir para demais repos.

---

## 7. Referências

- [OpenSpec — GitHub](https://github.com/Fission-AI/OpenSpec) (48K stars, MIT)
- [OpenSpec — Concepts](https://github.com/Fission-AI/OpenSpec/blob/main/docs/concepts.md) — Specs, changes, deltas, archive
- [OpenSpec — Workflows](https://github.com/Fission-AI/OpenSpec/blob/main/docs/workflows.md) — Patterns e best practices
- [OpenSpec — Customization](https://github.com/Fission-AI/OpenSpec/blob/main/docs/customization.md) — Custom schemas
- [VS Code — Custom Instructions](https://code.visualstudio.com/docs/copilot/customization/custom-instructions) — Instructions e AGENTS.md
- [VS Code — Custom Agents](https://code.visualstudio.com/docs/copilot/customization/custom-agents) — Agent primitives
- [Vídeo referenciado no card EPT-2127](https://www.youtube.com/watch?v=dLs-Pbn8stU)
- `ebb-governance-clean` — Prova de conceito funcional com AGENTS.md + Instructions + Prompts
- PRs de experimento comparativo: [#17](https://github.com/Ebury-Brazil/ebb-terraform-gcp-cloud-functions/pull/17), [#18](https://github.com/Ebury-Brazil/ebb-terraform-gcp-cloud-functions/pull/18), [#19](https://github.com/Ebury-Brazil/ebb-terraform-gcp-cloud-functions/pull/19), [#20](https://github.com/Ebury-Brazil/ebb-terraform-gcp-cloud-functions/pull/20), [#21](https://github.com/Ebury-Brazil/ebb-terraform-gcp-cloud-functions/pull/21) — Mesmo pedido, 3 abordagens diferentes
