# Proposta: Atualização do Padrão Cloud DNS — gRPC e Domínios por Aplicação

**Card:** [EPT-2350](https://fxsolutions.atlassian.net/browse/EPT-2350)  
**Autor:** Osório Santos  
**Data:** 11/05/2026  
**Status:** Proposta (draft para revisão)  
**Referência atual:** [013 - Cloud DNS](https://bexs.atlassian.net/wiki/spaces/EP/pages/2868543517/013+-+Cloud+DNS)

---

## 1. Contexto

O padrão atual de DNS (ADR 013) contempla apenas dois tipos de entrada:

| Tipo | Formato |
|------|---------|
| Geral | `*-<env>[-int].ebury.com.br` |
| API/Backend | `*-api-<env>[-int].ebury.com.br` |

Esse modelo apresenta três lacunas:

1. **Sem padrão para gRPC** — registros foram criados ad-hoc (`wp-quotes-grpc`, `ebb-moneyflows-spi-grpc-dev-int`, `finance-h-int-grpc`), cada um com posição diferente do indicador de protocolo.
2. **Sem domínio por aplicação** — o roteamento por path sob um host compartilhado (`webpayments-api-dev.ebury.com.br/v1/portal`, `/v1/hedges`, etc.) impede migração granular e é incompatível com gRPC.
3. **Sem desambiguação de nomes iguais** — aplicações com o mesmo nome em domínios diferentes (ex: `core` em Money Flows e Webpayments) colidem sem um qualificador.

---

## 2. Proposta

### 2.1 Adição: Padrão gRPC

Adicionar à tabela de padrões da ADR 013 uma entrada para serviços gRPC:

| Tipo | Interno DEV | Interno STG | Interno PRD |
|------|-------------|-------------|-------------|
| gRPC | `*-grpc-dev-int.ebury.com.br` | `*-grpc-stg-int.ebury.com.br` | `*-grpc-int.ebury.com.br` |

**Regras:**
- O sufixo `-grpc` aparece **antes** do ambiente e **depois** do nome da aplicação
- gRPC é tipicamente interno — não há padrão externo (pode ser aberto caso a caso)
- Para HTTP/REST o protocolo continua omitido (comportamento atual)

**Exemplos:**

| Hoje (ad-hoc) | Novo padrão |
|----------------|-------------|
| `wp-quotes-grpc.ebury.com.br` | `wp-quotes-grpc-dev.ebury.com.br` |
| `finance-h-int-grpc.ebury.com.br` | `finance-grpc-stg-int.ebury.com.br` |
| `ebb-moneyflows-spi-grpc-dev-int.ebury.com.br` | `mf-spi-grpc-dev-int.ebury.com.br` |

### 2.2 Adição: Domínio por aplicação (host-per-app)

Adicionar ao padrão a possibilidade de cada aplicação ter seu **próprio registro DNS**, eliminando a dependência de roteamento por path.

**Formato:**

```
<aplicacao>[-<protocolo>]-<env>[-int].ebury.com.br
```

**Por que host-per-app?**

| Path-based (atual) | Host-per-app (proposto) |
|---------------------|--------------------------|
| Migração big-bang — todas as apps do host migram juntas | Migração independente — basta mudar o target do registro DNS |
| Requer rewrite/redirect/proxy-forward no ingress | Roteamento direto por host — sem regras extras |
| Incompatível com gRPC (gRPC usa hostname, não path) | gRPC nativo — host-based routing |
| Blast radius alto — problema no ingress afeta todas as apps | Isolado por aplicação |

**Exemplos por time:**

| Hoje (path-based) | Proposto (host-per-app) |
|--------------------|--------------------------|
| `webpayments-api-dev.ebury.com.br/v1/portal` | `wp-portal-api-dev.ebury.com.br` |
| `webpayments-api-dev.ebury.com.br/v1/hedges` | `wp-hedges-dev.ebury.com.br` |
| `clientjourney-api-dev-int.ebury.com.br/temis-config` | `temis-config-dev-int.ebury.com.br` |
| `ebb-moneyflows-api-dev-int.ebury.com.br/v1/transfer` | `mf-transfer-dev-int.ebury.com.br` |

> O padrão path-based continua aceito para aplicações existentes. Aplicações novas devem adotar host-per-app.

### 2.3 Adição: Desambiguação de nomes entre domínios

Quando uma aplicação tem o **mesmo nome** em domínios diferentes, o nome deve ser qualificado com um **prefixo curto do domínio**. A regra é simples:

- **Se o nome da aplicação é único na organização** → usa o nome direto (ex: `temis-config`, `wp-hedges`)
- **Se o nome existe em mais de um domínio** → prefixar com identificador do domínio (ex: `wp-core`, `mf-core`)

**Prefixos de domínio para desambiguação:**

| Domínio | Prefixo | Nota |
|---------|:-------:|------|
| Money Flows | `mf-` | — |
| Ebury Connect / Webpayments | `wp-` | Já usado naturalmente pelas apps do time |
| FinCrime (ex-Client Journey) | `fincrime-` | Evita confusão com FinCore — não usar `fi-` nem `fc-` |
| FinCore (ex-FX) | `fincore-` | Evita confusão com FinCrime — não usar `fi-` nem `fc-` |
| Enablers | `enablers-` | — |
| BigData | `bd-` | — |
| Platform | `pt-` | — |

> **Por que não usar abreviações de 2 letras para FinCrime/FinCore?**  
> `fi-` (FinCrime) e `fc-` (FinCore) são muito similares e geram confusão. Usar o nome por extenso (`fincrime-`, `fincore-`) elimina a ambiguidade.

**Exemplos de colisão resolvida:**

| Aplicação | Domínio | DNS proposto |
|-----------|---------|--------------|
| core | Webpayments | `wp-core-dev.ebury.com.br` |
| core | Money Flows | `mf-core-dev-int.ebury.com.br` |
| core | FinCore | `fincore-core-dev-int.ebury.com.br` |

**Quando o nome já é único (maioria dos casos):**

| Aplicação | DNS proposto |
|-----------|--------------|
| temis-config | `temis-config-dev-int.ebury.com.br` |
| wp-quotes | `wp-quotes-dev.ebury.com.br` |
| finance | `finance-dev-int.ebury.com.br` |
| forex-provider | `forex-provider-dev-int.ebury.com.br` |

---

## 3. Tabela de padrões consolidada (proposta para ADR 013)

| Tipo | Interno DEV | Interno STG | Interno PRD | Externo STG | Externo PRD |
|------|-------------|-------------|-------------|-------------|-------------|
| Geral | `*-dev-int.ebury.com.br` | `*-stg-int.ebury.com.br` | `*-int.ebury.com.br` | — | — |
| API/Backend | `*-api-dev-int.ebury.com.br` | `*-api-stg-int.ebury.com.br` | `*-api-int.ebury.com.br` | `*-api-stg.ebury.com.br` | `*-api.ebury.com.br` |
| **gRPC (novo)** | `*-grpc-dev-int.ebury.com.br` | `*-grpc-stg-int.ebury.com.br` | `*-grpc-int.ebury.com.br` | — | — |
| **App-specific (novo)** | `<app>-dev[-int].ebury.com.br` | `<app>-stg[-int].ebury.com.br` | `<app>[-int].ebury.com.br` | `<app>-stg.ebury.com.br` | `<app>.ebury.com.br` |

> DEV não possui comunicação externa. gRPC é tipicamente interno.

---

## 4. Migração

- **Registros existentes** permanecem inalterados — sem breaking change
- **Aplicações novas** adotam o novo padrão (host-per-app + gRPC quando aplicável)
- **Aplicações existentes** migram gradualmente, podendo manter ambos os hosts durante transição
- **Depreciação** de registros antigos após confirmação de zero-traffic (Cloud DNS query logging, período mínimo de 90 dias)

---

## 5. Referências

- [013 - Cloud DNS (padrão atual)](https://bexs.atlassian.net/wiki/spaces/EP/pages/2868543517/013+-+Cloud+DNS)
- [EPT-2350 — Modernizing DNS Patterns for gRPC and Application-Driven Architectures](https://fxsolutions.atlassian.net/browse/EPT-2350)
- [gRPC Name Resolution](https://grpc.io/docs/guides/custom-name-resolution/)
- [Istio VirtualService host matching](https://istio.io/latest/docs/reference/config/networking/virtual-service/)
