# IO Benchmark — Nutanix Kubernetes Platform (NKP) — iSCSI

Variante do pacote `k8s-iobenchmark-nkp/` usando **Nutanix Volumes (bloco via iSCSI)** como storage padrão, em vez de Nutanix Files (NFS).

## Diferença em relação a `k8s-iobenchmark-nkp/`

| Arquivo | Mudança |
|---|---|
| `storageclass.yaml` | **Nutanix Volumes** (`storageType: NutanixVolumes`) ativo por padrão. Nutanix Files disponível comentado. |
| `statefulset.yaml` | `storageClassName: nutanix-volumes` (era `nutanix-files`) |
| `pvc.yaml` | Referência atualizada para `nutanix-volumes` |
| Demais arquivos | Idênticos ao pacote `k8s-iobenchmark-nkp/` (código Python é agnóstico de storage) |

## Como funciona o Nutanix Volumes

Cada PVC criado pelo `volumeClaimTemplates` do StatefulSet vira um **Volume Group** dedicado no Prism, montado via **iSCSI** diretamente no node onde o pod está agendado. Isso é equivalente em conceito ao modo "iSCSI" genérico do pacote Kubernetes vanilla, mas usando o driver e gerenciamento nativo do Nutanix Cluster (sem precisar de um target iSCSI externo).

- **AccessMode**: `ReadWriteOnce` — cada Volume Group é montado por um único pod por vez (adequado aqui, já que cada réplica do StatefulSet tem seu próprio PVC)
- **CHAP**: desabilitado por padrão (`chapAuth: "false"`) — habilite se o seu cluster Nutanix exigir autenticação CHAP no iSCSI

## Pré-requisitos específicos

1. **Nutanix CSI Driver** instalado (addon "Nutanix CSI Storage" via Kommander, ou manual):
   ```bash
   kubectl get pods -n ntnx-system | grep csi
   ```
2. **Secret de credenciais do CSI** (padrão: `nutanix-csi-credentials` no namespace `ntnx-system`):
   ```bash
   kubectl get secret -n ntnx-system nutanix-csi-credentials
   ```
3. **Storage Container** já criado no Prism (Storage Console) — edite `storageContainer` em `storageclass.yaml` com o nome exato.
4. Se o cluster usar autenticação CHAP no iSCSI, troque `chapAuth: "false"` para `"true"` em `storageclass.yaml`.

## Quick Start

```bash
kubectl apply -f k8s-iobenchmark-nkp-iscsi/namespace.yaml
kubectl apply -f k8s-iobenchmark-nkp-iscsi/storageclass.yaml   # confirme storageContainer antes
kubectl apply -f k8s-iobenchmark-nkp-iscsi/secret.yaml          # só necessário se for usar Nutanix Objects
kubectl apply -f k8s-iobenchmark-nkp-iscsi/configmap.yaml
kubectl apply -f k8s-iobenchmark-nkp-iscsi/service.yaml
kubectl apply -f k8s-iobenchmark-nkp-iscsi/statefulset.yaml
kubectl apply -f k8s-iobenchmark-nkp-iscsi/deployment.yaml
kubectl apply -f k8s-iobenchmark-nkp-iscsi/hpa.yaml
```

## Trocar para Nutanix Files (NFS) em vez de Volumes

1. Em `storageclass.yaml`, descomente o bloco `nutanix-files` e edite `nfsServerName`
2. Em `statefulset.yaml`, troque `storageClassName: nutanix-volumes` por `nutanix-files`

## Trocar para Nutanix Objects (S3) em vez de armazenamento em bloco

1. Crie um bucket no Prism Central (Objects) e gere Access/Secret Key
2. Preencha `secret.yaml` com os valores em base64
3. Em `statefulset.yaml`, descomente `STORAGE_TYPE: "s3"` e as 4 env vars de credenciais S3
4. **Não é necessário aplicar** `storageclass.yaml`/PVCs — o worker S3 do io-generator ignora o volume de dados

## Validação

Todos os manifests foram validados com `yamllint` e checagem cruzada de nomes (StorageClass, ConfigMap, Secret, Service, namespace) — sem erros.
