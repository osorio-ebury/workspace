# Prompt: Implementar sincronização do certificado ebury-tls para outro domínio

Estou com o card EPT-2392. Preciso implementar a sincronização do certificado ebury-tls do projeto ebb-platform-prod para outro domínio via External Secrets Operator + Workload Identity.

Documentação de referência: https://bexs.atlassian.net/wiki/spaces/EP/pages/3280994306/005+-+Atualiza+o+em+massa+do+Certificado+Ebury+Tls

## O que precisa ser feito:

### 1. IAM no Secret Manager (ebb-iac-resource)

No arquivo `ebb-platform/prd/security/secret-manager/ebb-ebury-tls/terragrunt.hcl`, adicionar IAM bindings com `roles/secretmanager.secretAccessor` + condition scoped ao secret para as service accounts do external-secrets do domínio alvo (dev, staging, prod). O project number do ebb-platform-prod é `1092549940900`.

Formato da condition:

```hcl
condition = {
  title       = "access-only-ebb-ebury-tls-<identificador>"
  description = "Allows access only to the ebb-ebury-tls secret by the <env> external-secrets service account"
  expression  = "resource.name == \"projects/1092549940900/secrets/ebb-ebury-tls\" || resource.name.startsWith(\"projects/1092549940900/secrets/ebb-ebury-tls/versions/\")"
}
```

Verificar o values.yaml do external-secrets de cada ambiente para encontrar a GSA correta (anotação `iam.gke.io/gcp-service-account`).

### 2. ClusterSecretStore + ExternalSecrets (ebb-platform-argocd)

Para cada ambiente (dev, staging, prod), no path `ebb-<dominio>/<env>/external-secrets/`:

a) Reorganizar `k8s-manifests/` em:
   - `k8s-manifests/cluster-secret-store/gcpsm-secret-store.yaml` (mover o existente)
   - `k8s-manifests/cluster-secret-store/gcpsm-secret-store-ebb-platform-prd.yaml` (criar novo)
   - `k8s-manifests/external-secrets/ebury-tls-<namespace>.yaml` (um por namespace)

b) Criar o ClusterSecretStore cross-project:

```yaml
---
apiVersion: external-secrets.io/v1beta1
kind: ClusterSecretStore
metadata:
  name: gcpsm-secret-store-ebb-platform-prd
spec:
  provider:
    gcpsm:
      projectID: "ebb-platform-prod"
      auth:
        workloadIdentity:
          clusterLocation: southamerica-east1
          clusterName: <nome-do-cluster>
          clusterProjectID: <project-id-do-cluster>
          serviceAccountRef:
            name: external-secrets
            namespace: external-secrets-system
```

c) Criar ExternalSecret para cada namespace:

```yaml
---
apiVersion: external-secrets.io/v1beta1
kind: ExternalSecret
metadata:
  name: ebury-tls-<namespace>
  namespace: <namespace>
spec:
  refreshInterval: 1h
  secretStoreRef:
    kind: ClusterSecretStore
    name: gcpsm-secret-store-ebb-platform-prd
  target:
    name: ebury-tls
    template:
      type: kubernetes.io/tls
  data:
    - secretKey: tls.crt
      remoteRef:
        key: ebb-ebury-tls
        property: tls.crt
        conversionStrategy: Default
        decodingStrategy: None
        metadataPolicy: None
    - secretKey: tls.key
      remoteRef:
        key: ebb-ebury-tls
        property: tls.key
        conversionStrategy: Default
        decodingStrategy: None
        metadataPolicy: None
```

d) Atualizar o `kustomization.yaml` para referenciar todos os novos arquivos.

### 3. Namespaces a criar os ExternalSecrets:

- core
- hedge
- istio-gateway-external
- istio-gateway-internal
- kong
- kong
- merchants
- mocks
- monitoring
- portal
- quotes

### 4. PRs

- 1 PR para o ebb-iac-resource (IAM bindings dos 3 ambientes)
- 1 Pr com um namespace para validação do funcionamento em dev
- 1 PR por ambiente no ebb-platform-argocd (dev(restante dos namespaces), stg, prd)

Formato do título: `feat(external-secrets): add ebury-tls sync for all <dominio> <env> namespaces [EPT-2392]`

### 5. Após finalizar

Comentar no card EPT-2392 no Jira com o resumo das alterações e links dos PRs.

OBS: o Addon kong, possui um external-secrets que usa o mesmo valor, mas cria um secrets com nome diferente chamado ebb-kong-ebury-tls, mantenha o nome, faça o pegar o valor do novo ClusterSecrets Store.