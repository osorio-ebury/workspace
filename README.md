# github — Workspace de Referência

Workspace local que centraliza todos os repositórios da plataforma Ebury/Bexs, organizados por time/squad, junto com documentação técnica e configurações de AI.

## Estrutura

```
github/
├── Client Journey/       # Time FinCrime (ex-Client Journey)
├── FX/                   # Time FinCore (ex-FX Engine)
├── Money-Flows/          # Time Money Flows (ex-Banking)
├── Platform/             # Time Platform / Enablers
├── Webpayments/          # Time Webpayments (Ebury Connect)
├── .github/              # Configurações do GitHub Copilot
│   ├── .copilot-instructions.md   # Contexto da plataforma para o Copilot
│   └── agents/
│       └── jira-agent.agent.md    # Agente Copilot para interagir com o Jira
└── *.md                  # Notas e documentação técnica
```

## Convenção de Repositórios

Cada aplicação dentro de um time segue o padrão de três subpastas:

```
<Time>/<nome-app>/
├── ebb-<nome-app>/           # Código fonte
├── ebb-<nome-app>-gitops/    # Manifests K8s (ArgoCD / ArgoWorkflow)
└── ebb-<nome-app>-tests/     # Testes automatizados (BDD)
```

> Os repositórios aninhados são independentes e já possuem versionamento próprio — este repositório raiz **não** os rastreia (ver `.gitignore`).

## Times e Domínios

| Pasta          | Time              | Domínio Terraform       |
|----------------|-------------------|-------------------------|
| `Webpayments/` | Webpayments       | `ebb-ebury-connect`     |
| `FX/`          | FinCore           | `ebb-fincore`           |
| `Client Journey/` | FinCrime       | `ebb-fincrime`          |
| `Money-Flows/` | Money Flows       | `ebb-money-flows`       |
| `Platform/`    | Platform/Enablers | `ebb-enablers`          |

## GitHub Copilot

O diretório `.github/` contém instruções e agentes customizados para o GitHub Copilot:

- **`.copilot-instructions.md`** — descreve a estrutura do workspace, padrões de GitOps, times e convenções da plataforma para que o Copilot tenha contexto ao responder perguntas sobre qualquer repositório.
- **`agents/jira-agent.agent.md`** — agente especializado para consultar e criar issues no board Jira da fxsolutions (`fxsolutions.atlassian.net`). Autenticação via `~/.jira-credentials`.
