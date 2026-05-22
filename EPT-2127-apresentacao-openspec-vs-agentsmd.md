# Context Engineering para Repositórios de Plataforma
## OpenSpec vs AGENTS.md — Qual adotar?

**Card**: EPT-2127 | **Autor**: Osório Santos | **Data**: Maio 2026

---

## O que é o OpenSpec?

O **OpenSpec** é um framework open source da Fission AI para **Spec-Driven Development** — uma camada de especificação entre a intenção do desenvolvedor e o código.

### Filosofia

A IA e o humano **concordam sobre o que será construído** antes de escrever código.

### Como funciona

```
Propose → Specs → Design → Tasks → Implement → Verify → Archive
```

| Etapa | O que gera | Propósito |
|-------|-----------|-----------|
| **Propose** | `proposal.md` | Por que e o que vai mudar |
| **Specs** | `specs/*.md` (delta specs) | O que muda no comportamento |
| **Design** | `design.md` | Como implementar tecnicamente |
| **Tasks** | `tasks.md` | Checklist de implementação |
| **Implement** | Código real | IA executa as tasks |
| **Archive** | Move para `archive/` | Preserva histórico de decisões |

### Dados do projeto

- 48K stars no GitHub
- 65 contribuidores
- MIT License, v1.3.1
- Suporta 25+ ferramentas de IA

### Instalação

```bash
npm install -g @fission-ai/openspec@latest
openspec init
```

Cria uma pasta `openspec/` no repositório com configs, schemas e instruções para a IA.

---

## O Experimento: Mesmo pedido, 3 abordagens

### O pedido (idêntico nos 3 cenários)

> "Adicionar suporte a VPC Connector no `service_config` para permitir que Cloud Functions acessem recursos em redes VPC privadas (Cloud SQL, Memorystore, APIs internas)."

Repositório: `ebb-terraform-gcp-cloud-functions` (~285 linhas de HCL)

---

## Resultado 1: Com OpenSpec

### O que aconteceu

O fluxo exigiu **3 fases** e gerou **3 PRs**:

| PR | Fase | Arquivos | Linhas |
|----|------|----------|--------|
| #19 | Propose | 14 arquivos | +1.375 |
| #17 | Apply | 5 arquivos | +15 |
| #18 | Archive | 6 arquivos | +34 |
| **Total** | | **25 arquivos** | **+1.424** |

### Scaffolding gerado (PR #19 — Propose)

- `openspec/config.yaml`
- 4 prompts (`.github/prompts/opsx-*.prompt.md`)
- 4 skills (`.github/skills/openspec-*/SKILL.md`)
- `proposal.md`, `design.md`, `tasks.md`
- `specs/vpc-connector/spec.md`

**1.375 linhas de planejamento** para uma mudança de 15 linhas de código.

### Código gerado (PR #17 — Apply)

**`variables.tf`**:
```hcl
vpc_connector                    = optional(string, null)
vpc_connector_egress_settings    = optional(string, "PRIVATE_RANGES_ONLY")
```

**`main.tf`**:
```hcl
vpc_connector                    = each.value.vpc_connector
vpc_connector_egress_settings    = each.value.vpc_connector_egress_settings
```

⚠️ Default `"PRIVATE_RANGES_ONLY"` para egress **com `null` para connector** — se alguém não setar `vpc_connector`, a API do GCP retorna erro 400 (egress settings sem connector é inconsistência lógica)
❌ Sem guard clause — envia egress settings mesmo sem connector configurado
✅ Atualizou `example.tf` e `README.md`

### Interações necessárias

```
/opsx:propose → revisar artifacts → /opsx:apply → /opsx:archive
```

**3+ interações** com a IA para completar o ciclo.

---

## Resultado 2: Sem contexto nenhum

### O que aconteceu

| PR | Arquivos | Linhas |
|----|----------|--------|
| #20 | 3 arquivos | +11 |

### Código gerado

**`variables.tf`**:
```hcl
vpc_connector                    = optional(string, null)
vpc_connector_egress_settings    = optional(string, null)
```

**`main.tf`**:
```hcl
vpc_connector                    = each.value.vpc_connector
vpc_connector_egress_settings    = each.value.vpc_connector_egress_settings
```

⚠️ Default `null` para ambos — funciona (API omite os campos), mas não explicita o padrão GCP
❌ Sem guard clause
❌ Não atualizou `example.tf`
❌ Não atualizou `README.md`
❌ Não seguiu o padrão existente do módulo (`optional(string, "")`)

### O problema

Sem contexto, a IA:
- Não sabia que `examples/` deve ser atualizado junto
- Não conhecia o padrão de `optional(string, "")` usado no módulo
- Não pensou no cenário de falha (egress sem connector)
- Fez **o mínimo e parou**

---

## Resultado 3: Com AGENTS.md

### O que aconteceu

| PR | Arquivos | Linhas |
|----|----------|--------|
| #21 | 4 arquivos | +72 (63 do AGENTS.md + 9 de código) |

### O AGENTS.md usado (63 linhas)

Descreve:
- Overview do módulo (o que faz, provider, recursos)
- Arquitetura (main.tf, variables.tf, outputs.tf, examples/)
- Code style (for_each, dynamic blocks, optional())
- Convenções (naming, lifecycle, labels obrigatórias)
- Build & test commands

### Código gerado

**`variables.tf`**:
```hcl
vpc_connector                    = optional(string, "")
vpc_connector_egress_settings    = optional(string, "PRIVATE_RANGES_ONLY")
```

**`main.tf`**:
```hcl
vpc_connector                    = each.value.vpc_connector != "" ? each.value.vpc_connector : null
vpc_connector_egress_settings    = each.value.vpc_connector != "" ? each.value.vpc_connector_egress_settings : null
```

✅ Default `""` — **consistente** com `service_account_email = optional(string, "")`
✅ Default `"PRIVATE_RANGES_ONLY"` — documenta o padrão GCP explicitamente
✅ **Guard clause** — só envia `vpc_connector_egress_settings` se houver connector configurado (evita erro 400 da API)
✅ Atualizou `example.tf` com demonstração
✅ **Seguiu as convenções existentes** do módulo

### Por que funcionou melhor

A IA leu o AGENTS.md e entendeu:
- Que `optional(string, "")` é o padrão do módulo (não `null`)
- Que `examples/` deve ser atualizado junto
- Que condicionais com string vazia são o padrão (ex: `service_account_email`)
- **A guard clause previne o cenário de falha**: setar egress sem connector causa erro 400 na API do GCP

**1 interação** — pedido direto, resultado completo.

---

## Comparação Direta

> **Nota técnica**: A API do GCP para `google_cloudfunctions2_function` rejeita com erro 400 se `vpc_connector_egress_settings` for definido sem um `vpc_connector` válido. O default da API quando ambos são omitidos é `PRIVATE_RANGES_ONLY`. Isso significa que definir um default explícito para egress **sem guard clause** pode causar falha no `terraform apply` quando o usuário não configura um connector.

| Métrica | OpenSpec | Sem contexto | AGENTS.md |
|---------|:--------:|:------------:|:---------:|
| **Linhas adicionadas** | 1.424 | 11 | 72 |
| **Linhas de código real** | 15 | 11 | 9 |
| **Ratio overhead/código** | **95:1** | 0:1 | 7:1 |
| **Atualizou `example.tf`** | ✅ | ❌ | ✅ |
| **Guard clause** | ❌ | ❌ | ✅ |
| **Default egress seguro** | ❌ (pode causar erro 400) | ✅ (null = omitido) | ✅ (guard clause protege) |
| **Segue convenções** | Parcial | ❌ | ✅ |
| **PRs necessários** | 3 | 1 | 1 |
| **Interações com IA** | 3+ | 1 | 1 |

---

## Consumo de Tokens

| Cenário | Tokens carregados por interação | Observação |
|---------|:-------------------------------:|------------|
| **OpenSpec** | ~2.000+ | Specs + artifacts + instruções do framework são carregados a cada interação |
| **Sem contexto** | ~0 | Nada extra — apenas o prompt do usuário |
| **AGENTS.md** | ~400-650 | Arquivo leve, focado, carregado automaticamente |

### Impacto prático

- **OpenSpec**: Cada `/opsx:propose`, `/opsx:apply`, `/opsx:archive` carrega todo o contexto do framework + specs + artifacts existentes
- **AGENTS.md**: Carregado 1x no início da sessão — impacto mínimo no budget de tokens
- Em repos com múltiplas mudanças por sprint, o acúmulo de tokens do OpenSpec é significativo

---

## Prós e Contras

### OpenSpec

| ✅ Prós | ❌ Contras |
|---------|-----------|
| Força planejamento antes de implementar | Overhead desproporcional para repos pequenos (95:1) |
| Delta specs documentam exatamente o que muda | Requer `npm install -g` + Node.js em todas as máquinas |
| Archive preserva histórico de decisões | Learning curve alta — 10+ pessoas precisam aprender |
| Suporta 25+ ferramentas de IA | Manutenção: specs ficam desatualizados se alguém esquece de fazer archive |
| Bom para apps grandes (5k+ linhas) | Consome ~2.000+ tokens por interação |
| Specs previnem que a IA "esqueça" requirements | Duplicação: `spec.md` replica o que `variables.tf` já define |
| | **A pasta `openspec/` é baixada junto com o módulo quando usado como source no Terraform** — conteúdo de planejamento/specs poluem o download do módulo em todos os `terragrunt.hcl` que o referenciam |
| | Workflow de 3 fases (propose → apply → archive) transforma tarefa de 5 min em múltiplas iterações |
| | Não resolve o problema real: falta de **contexto permanente** para a IA |

### AGENTS.md

| ✅ Prós | ❌ Contras |
|---------|-----------|
| Zero setup — é um arquivo Markdown na raiz | Manutenção manual — precisa atualizar quando convenções mudam |
| Zero learning curve — dev não aprende nada novo | Sem spec-first workflow — não força planejamento formal |
| Automático — carregado em toda interação sem ação do dev | Sem histórico de decisões (usar Jira/Confluence) |
| Versionado no git — compartilhado com todo o time | Qualidade depende do autor |
| Econômico: ~400-650 tokens por interação | Não substitui documentação para humanos |
| Funciona com Copilot, Claude, Cursor | |
| Ecossistema completo: Instructions + Prompts + Agents | |
| Incremental: começa com 1 arquivo, expande conforme necessidade | |
| **Não polui o módulo** — 1 arquivo na raiz, ignorável | |
| Caso real funcionando: `ebb-governance-clean` | |

---

## Veredicto

### Para nossos repositórios (módulos Terraform + Cloud Functions):

| | OpenSpec | AGENTS.md |
|--|:-------:|:---------:|
| **Adotar?** | ❌ Não | ✅ Sim |
| **Por quê?** | Overhead > benefício para repos pequenos e declarativos | Resolve o problema exato com custo mínimo |

### Recomendação

1. **Adotar AGENTS.md** nos 12 módulos Terraform (esforço: 1 dia)
2. **Expandir** com `.instructions.md` e `.prompt.md` para Cloud Functions
3. **Adotar a filosofia** do OpenSpec (spec-first thinking) sem a ferramenta
4. **Reavaliar OpenSpec** se/quando gerenciarmos apps grandes (5k+ linhas)

---

## Links dos PRs do experimento

- [PR #19 — OpenSpec Propose](https://github.com/Ebury-Brazil/ebb-terraform-gcp-cloud-functions/pull/19) (+1.375 linhas, 14 arquivos)
- [PR #17 — OpenSpec Apply](https://github.com/Ebury-Brazil/ebb-terraform-gcp-cloud-functions/pull/17) (+15 linhas, 5 arquivos)
- [PR #18 — OpenSpec Archive](https://github.com/Ebury-Brazil/ebb-terraform-gcp-cloud-functions/pull/18) (+34 linhas, 6 arquivos)
- [PR #20 — Sem contexto](https://github.com/Ebury-Brazil/ebb-terraform-gcp-cloud-functions/pull/20) (+11 linhas, 3 arquivos)
- [PR #21 — Com AGENTS.md](https://github.com/Ebury-Brazil/ebb-terraform-gcp-cloud-functions/pull/21) (+72 linhas, 4 arquivos)
