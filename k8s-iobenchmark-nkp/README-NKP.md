# IO Benchmark — Nutanix Kubernetes Platform (NKP)

Versão adaptada do pacote `k8s-iobenchmark/` para rodar sobre **Nutanix Kubernetes Platform (NKP)**, usando o storage nativo do Nutanix Cluster em vez de StorageClasses genéricas.

## O que muda em relação ao pacote Kubernetes vanilla

| Arquivo | Mudança |
|---|---|
| `storageclass.yaml` | Usa o **Nutanix CSI Driver** (`csi.nutanix.com`) com **Nutanix Files** (NFS) como opção ativa. Nutanix Volumes (iSCSI) disponível comentado. Nutanix Objects (S3) documentado como opção sem StorageClass. |
| `statefulset.yaml` | `storageClassName: nutanix-files` (era `nfs-storage`) |
| `pvc.yaml` | Referência atualizada para `nutanix-files` |
| `secret.yaml` | Comentários apontam para credenciais do Nutanix Objects em vez de S3 genérico |
| `service.yaml` | Nota sobre usar `type: LoadBalancer` se o cluster tiver MetalLB via Kommander |
| `namespace.yaml`, `configmap.yaml`, `deployment.yaml`, `hpa.yaml` | Sem mudanças funcionais — apenas headers/comentários |

O código Python (`configmap.yaml`) é **idêntico** ao pacote vanilla — a aplicação já suporta os 3 modos de storage (`file` para Files/Volumes, `s3` para Objects) via a env var `STORAGE_TYPE`.

## Pré-requisitos específicos do NKP

1. **Nutanix CSI Driver** instalado no cluster — normalmente via addon "Nutanix CSI Storage" no Kommander, ou manualmente:
   ```bash
   kubectl get pods -n ntnx-system | grep csi
   ```
2. **Secret de credenciais do CSI** já criado pelo instalador (padrão: `nutanix-csi-credentials` no namespace `ntnx-system`):
   ```bash
   kubectl get secret -n ntnx-system nutanix-csi-credentials
   ```
   Se o nome/namespace for diferente no seu cluster, ajuste em `storageclass.yaml`.
3. **File Server** já criado no Prism Central (Files Console) — edite `nfsServerName` em `storageclass.yaml` com o nome exato.

## Quick Start

```bash
kubectl apply -f k8s-iobenchmark-nkp/namespace.yaml
kubectl apply -f k8s-iobenchmark-nkp/storageclass.yaml   # confirme nfsServerName antes
kubectl apply -f k8s-iobenchmark-nkp/secret.yaml          # só necessário se for usar Nutanix Objects
kubectl apply -f k8s-iobenchmark-nkp/configmap.yaml
kubectl apply -f k8s-iobenchmark-nkp/service.yaml
kubectl apply -f k8s-iobenchmark-nkp/statefulset.yaml
kubectl apply -f k8s-iobenchmark-nkp/deployment.yaml
kubectl apply -f k8s-iobenchmark-nkp/hpa.yaml
```

## Trocar para Nutanix Volumes (iSCSI) em vez de Files

1. Em `storageclass.yaml`, descomente o bloco `nutanix-volumes` e edite `storageContainer`
2. Em `statefulset.yaml`, troque `storageClassName: nutanix-files` por `nutanix-volumes`

## Trocar para Nutanix Objects (S3) em vez de armazenamento em arquivo

1. Crie um bucket no Prism Central (Objects) e gere Access/Secret Key
2. Preencha `secret.yaml` com os valores em base64
3. Em `statefulset.yaml`, descomente `STORAGE_TYPE: "s3"` e as 4 env vars de credenciais S3
4. **Não é necessário aplicar** `storageclass.yaml`/PVCs — o worker S3 do io-generator ignora o volume de dados

## Validação

Todos os manifests foram validados com `yamllint` e checagem cruzada de nomes (StorageClass, ConfigMap, Secret, Service, namespace) — sem erros.
