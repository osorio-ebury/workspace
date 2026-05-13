# Trade-off Analysis: External Secrets Operator (ESO) vs Google Secret Manager CSI Driver

> **Jira Card:** [EPT-2204](https://fxsolutions.atlassian.net/browse/EPT-2204)  
> **Epic:** [EPT-536](https://fxsolutions.atlassian.net/browse/EPT-536) — Reorganization of the EBB Infrastructure environment on GCP Cloud  
> **Status:** Tarefas pendentes  
> **Assignee:** Osório Santos  
> **Data:** Abril de 2026

---

## 1. Contexto

Atualmente, os secrets nos clusters GKE da EBB são gerenciados via **External Secrets Operator (ESO)**. Este documento avalia o trade-off entre manter o ESO ou migrar para o **Google Secret Manager CSI Driver**, com o objetivo de definir o caminho a ser seguido.

---

## 2. Visão Geral das Soluções

### 2.1 External Secrets Operator (ESO)

O ESO sincroniza secrets de provedores externos (como o Google Secret Manager) e os materializa como **objetos `Secret` nativos do Kubernetes**, armazenados no **etcd** do cluster.

**Fluxo:**

```
Google Secret Manager → ESO Controller → Kubernetes Secret (etcd) → Pod (env var / volume mount)
```

**Características:**

- Secrets são armazenados como objetos Kubernetes no etcd
- Suporte maduro e amplamente adotado pela comunidade
- Reconciliação automática (sync periódico com a origem)
- Projeto CNCF, com suporte ativo
- Compatível com múltiplos backends (GCP, AWS, Azure, Vault, etc.)
- Permite templates e transformações nos secrets

### 2.2 Google Secret Manager CSI Driver

O CSI Driver monta secrets diretamente como volumes nos pods, sem armazená-los no etcd. Os secrets existem apenas enquanto o pod está em execução.

**Fluxo:**

```
Google Secret Manager → CSI Driver → Volume Mount no Pod → Pod (arquivo no filesystem)
```

**Características:**

- Secrets **não são armazenados no etcd**
- Montados diretamente como arquivos no filesystem do container
- Secrets existem apenas durante o ciclo de vida do pod
- Pode também expor secrets como variáveis de ambiente (mas nesse caso **cria um objeto Secret**, anulando a vantagem)

---

## 3. Exemplos de Implementação

### 3.1 External Secrets Operator (ESO)

#### 3.1.1 SecretStore — Conexão com o Google Secret Manager

```yaml
apiVersion: external-secrets.io/v1beta1
kind: SecretStore
metadata:
  name: gcp-secret-store
  namespace: my-app
spec:
  provider:
    gcpsm:
      projectID: ebb-ebury-connect-dev
      auth:
        workloadIdentity:
          clusterLocation: southamerica-east1
          clusterName: ebb-dev-cluster
          serviceAccountRef:
            name: ebb-my-app
```

#### 3.1.2 ExternalSecret — Sincronização do Secret

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: my-app-credentials
  namespace: my-app
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: gcp-secret-store
    kind: SecretStore
  target:
    name: my-app-credentials        # Nome do K8s Secret criado
    creationPolicy: Owner
  data:
    - secretKey: database-password   # Chave no K8s Secret
      remoteRef:
        key: my-app-db-password      # Nome no GCP Secret Manager
        version: latest
    - secretKey: api-key
      remoteRef:
        key: my-app-api-key
        version: latest
```

#### 3.1.3 ExternalSecret com Template — Transformação de dados

```yaml
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: my-app-config
  namespace: my-app
spec:
  refreshInterval: 1h
  secretStoreRef:
    name: gcp-secret-store
    kind: SecretStore
  target:
    name: my-app-config
    template:
      engineVersion: v2
      data:
        config.yaml: |
          database:
            host: db.example.com
            password: {{ .db_password }}
          api:
            key: {{ .api_key }}
  data:
    - secretKey: db_password
      remoteRef:
        key: my-app-db-password
    - secretKey: api_key
      remoteRef:
        key: my-app-api-key
```

#### 3.1.4 Deployment consumindo o Secret (env var)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
  namespace: my-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      serviceAccountName: ebb-my-app
      containers:
        - name: my-app
          image: my-app:latest
          env:
            - name: DATABASE_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: my-app-credentials
                  key: database-password
            - name: API_KEY
              valueFrom:
                secretKeyRef:
                  name: my-app-credentials
                  key: api-key
```

#### 3.1.5 Deployment consumindo o Secret (volume mount)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
  namespace: my-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      serviceAccountName: ebb-my-app
      containers:
        - name: my-app
          image: my-app:latest
          volumeMounts:
            - name: secrets
              mountPath: /etc/secrets
              readOnly: true
      volumes:
        - name: secrets
          secret:
            secretName: my-app-credentials
```

---

### 3.2 Google Secret Manager CSI Driver

#### 3.2.1 SecretProviderClass — Definição dos secrets a montar

```yaml
apiVersion: secrets-store.csi.x-k8s.io/v1
kind: SecretProviderClass
metadata:
  name: my-app-secrets
  namespace: my-app
spec:
  provider: gcp
  parameters:
    secrets: |
      - resourceName: "projects/ebb-ebury-connect-dev/secrets/my-app-db-password/versions/latest"
        path: "database-password"
      - resourceName: "projects/ebb-ebury-connect-dev/secrets/my-app-api-key/versions/latest"
        path: "api-key"
```

#### 3.2.2 Deployment consumindo via volume mount (modo principal)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
  namespace: my-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      serviceAccountName: ebb-my-app
      containers:
        - name: my-app
          image: my-app:latest
          volumeMounts:
            - name: secrets
              mountPath: /etc/secrets
              readOnly: true
      volumes:
        - name: secrets
          csi:
            driver: secrets-store.csi.k8s.io
            readOnly: true
            volumeAttributes:
              secretProviderClass: my-app-secrets
```

> Os secrets ficam disponíveis como arquivos em `/etc/secrets/database-password` e `/etc/secrets/api-key`.

#### 3.2.3 SecretProviderClass com sync para K8s Secret (env var)

```yaml
apiVersion: secrets-store.csi.x-k8s.io/v1
kind: SecretProviderClass
metadata:
  name: my-app-secrets-sync
  namespace: my-app
spec:
  provider: gcp
  parameters:
    secrets: |
      - resourceName: "projects/ebb-ebury-connect-dev/secrets/my-app-db-password/versions/latest"
        path: "database-password"
      - resourceName: "projects/ebb-ebury-connect-dev/secrets/my-app-api-key/versions/latest"
        path: "api-key"
  # ⚠️ Ao usar secretObjects, o CSI Driver CRIA um objeto K8s Secret (armazenado no etcd)
  # Isso anula a vantagem de não armazenar no etcd
  secretObjects:
    - secretName: my-app-credentials
      type: Opaque
      data:
        - objectName: database-password
          key: database-password
        - objectName: api-key
          key: api-key
```

#### 3.2.4 Deployment consumindo via env var (com sync)

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
  namespace: my-app
spec:
  replicas: 1
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      serviceAccountName: ebb-my-app
      containers:
        - name: my-app
          image: my-app:latest
          env:
            - name: DATABASE_PASSWORD
              valueFrom:
                secretKeyRef:
                  name: my-app-credentials
                  key: database-password
            - name: API_KEY
              valueFrom:
                secretKeyRef:
                  name: my-app-credentials
                  key: api-key
          # ⚠️ O volume DEVE estar montado mesmo se usar apenas env vars
          volumeMounts:
            - name: secrets
              mountPath: /etc/secrets
              readOnly: true
      volumes:
        - name: secrets
          csi:
            driver: secrets-store.csi.k8s.io
            readOnly: true
            volumeAttributes:
              secretProviderClass: my-app-secrets-sync
```

> **Atenção:** Quando o CSI Driver sincroniza secrets como env vars, ele **cria um objeto K8s Secret no etcd**, comportando-se de forma equivalente ao ESO.

---

## 4. Comparação Funcional

| Critério | ESO | CSI Driver |
|---|---|---|
| **Armazenamento no etcd** | Sim | Não (exceto se usar env vars) |
| **Suporte oficial Google** | N/A (projeto open-source CNCF) | ⚠️ **Não é oficialmente suportado pela Google** |
| **Maturidade** | Alta — amplamente adotado | Moderada — menos adoção no ecossistema |
| **Multi-cloud** | Sim (GCP, AWS, Azure, Vault, etc.) | Apenas GCP Secret Manager |
| **Reconciliação automática** | Sim (polling configurável) | Não nativo — requer restart do pod |
| **Uso como env var** | Nativo (via Secret do K8s) | Cria objeto Secret (anula vantagem) |
| **Uso como volume** | Via Secret montado como volume | Nativo |
| **Templates/Transformações** | Sim (ExternalSecret templates) | Não |
| **Compatibilidade com workloads existentes** | Total — já implementado | Requer migração de todos os workloads |
| **Auditoria/Compliance (PCI-DSS)** | Aceitável, porém requer explicação | Pode facilitar auditorias |
| **Complexidade operacional** | Baixa (já em operação) | Alta (migração + novo operador) |

---

## 5. Resumo: Complexidade dos Manifests

| Operação | ESO | CSI Driver |
|---|---|---|
| Expor secret como **env var** | `SecretStore` + `ExternalSecret` + `Deployment` (3 manifests) | `SecretProviderClass` (com `secretObjects`) + `Deployment` com volume obrigatório (2 manifests, porém mais verboso) |
| Expor secret como **arquivo** | `SecretStore` + `ExternalSecret` + `Deployment` com volume (3 manifests) | `SecretProviderClass` + `Deployment` com CSI volume (2 manifests) |
| **Transformar/templatear** secrets | Suportado nativamente via `target.template` | Não suportado |
| **Reconciliação** automática | `refreshInterval` no ExternalSecret | Requer restart do pod |

---

## 6. Riscos da Migração para CSI Driver

| Risco | Severidade | Descrição |
|---|---|---|
| **Provedor não oficialmente suportado** | 🔴 Alta | O provedor do CSI Driver para Google Secret Manager não é oficialmente suportado pela Google. Risco de descontinuidade ou falta de suporte. |
| **Esforço de migração** | 🟡 Média | Todos os workloads existentes precisariam ser migrados, incluindo alterações em manifests, gitops repos e pipelines. |
| **Perda de funcionalidades** | 🟡 Média | Perda de templates, reconciliação automática e suporte multi-cloud do ESO. |
| **Env vars anulam vantagem** | 🟡 Média | Se secrets forem montados como variáveis de ambiente (uso comum), o CSI Driver cria objetos Secret, tornando-se equivalente ao ESO. |
| **Curva de aprendizado** | 🟢 Baixa | Equipe precisaria aprender novo operador e padrões de configuração. |

---

## 7. Análise de Custo-Benefício

### Manter ESO (status quo)

**Benefícios:**
- Zero esforço de migração
- Solução já validada e operacional em todos os ambientes (dev/stg/prd)
- Suporte CNCF ativo e comunidade ampla
- Flexibilidade para múltiplos backends
- Templates e reconciliação automática

**Custos:**
- Secrets armazenados no etcd (mitigável via IAM e RBAC)

### Migrar para CSI Driver

**Benefícios:**
- Secrets não ficam no etcd (quando montados como volume)
- Pode facilitar auditorias de compliance em cenários específicos

**Custos:**
- Esforço significativo de migração de todos os workloads
- Provedor não oficialmente suportado pela Google
- Perda de funcionalidades (templates, reconciliação, multi-cloud)
- Risco operacional durante e após migração

---

## 8. Parecer de Segurança

Análise realizada por **Lucas Cioffi** (equipe de segurança) em 15 de abril de 2026:

> Ambas as opções são aceitáveis do ponto de vista de segurança. Existe, discutivelmente, um pequeno ganho com o CSI Driver, mas não é algo que impeça o uso do ESO.

**Resumo da análise:**

- O ESO armazena secrets como objetos Kubernetes no etcd. Para extraí-los, um atacante precisaria de **acesso elevado à API do K8s**, **acesso ao SO dos nodes**, ou **explorar uma falha na aplicação** (path traversal, RCE, leak de env vars).
- O CSI Driver evita o armazenamento no etcd, mas um atacante ainda poderia extrair os secrets nos **mesmos cenários** descritos acima.
- Na prática, o CSI Driver **não mitiga categorias adicionais de ameaça** em relação ao ESO, **exceto** em dois casos específicos:
  - **Roubo de backups do etcd** — mitigável via IAM para acesso aos backups.
  - **Leitura direta de objetos Secret** (`kubectl get secrets`) — mitigável via RBAC restritivo.
- Se o CSI Driver montar secrets como variáveis de ambiente, ele cria um objeto Secret no etcd — anulando a vantagem.

**Pontos de atenção adicionais:**

- O provedor do CSI Driver para Google Secret Manager **não é oficialmente suportado pela Google**.
- Para credenciais de alta sensibilidade (ex: escopo PCI-DSS), o CSI Driver pode facilitar a argumentação com auditores, embora tecnicamente o nível de proteção seja equivalente.

---

## 9. Recomendação

**Manter o External Secrets Operator (ESO)** como solução padrão para gerenciamento de secrets nos clusters GKE.

### Justificativa

1. **Segurança equivalente:** O ganho de segurança do CSI Driver é marginal e os vetores adicionais mitigados já devem ser tratados via IAM/RBAC independentemente da solução.
2. **Risco do provedor:** O CSI Driver para Google Secret Manager não é oficialmente suportado pela Google, representando risco operacional.
3. **Custo de migração injustificável:** O esforço de migrar todos os workloads não se justifica diante do ganho marginal.
4. **Funcionalidades superiores:** O ESO oferece recursos que o CSI Driver não possui (templates, reconciliação automática, multi-backend).
5. **Estabilidade operacional:** A solução atual está funcional e validada em produção.

### Ações de Hardening Recomendadas (independente da decisão)

Para maximizar a segurança do ESO, recomenda-se:

- [ ] Revisar e restringir IAM de acesso aos backups do etcd
- [ ] Implementar RBAC restritivo para leitura de objetos Secret no cluster
- [ ] Habilitar encryption at rest no etcd (se não habilitado)
- [ ] Auditar periodicamente permissões de acesso a secrets

---

## 10. Referências

- [External Secrets Operator — Documentação Oficial](https://external-secrets.io/)
- [GCP Secret Manager CSI Driver](https://github.com/GoogleCloudPlatform/secrets-store-csi-driver-provider-gcp)
- [Kubernetes Secrets Best Practices](https://kubernetes.io/docs/concepts/security/secrets-good-practices/)
- [CNCF External Secrets — GitHub](https://github.com/external-secrets/external-secrets)
