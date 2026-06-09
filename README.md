# IO Synthetic Benchmark Platform

Plataforma de geração de IO sintético para Kubernetes com dashboard web em tempo real.

Deploy de **3 pods geradores de IO** distribuídos em workers diferentes + **1 dashboard** que agrega e exibe métricas ao vivo, tudo em um único arquivo YAML autocontido.

---

## Visão Geral

```
┌─────────────────────────────────────────────────────────┐
│                 NAMESPACE: io-benchmark                  │
│                                                          │
│  io-generator-0 ──► PVC (NFS/iSCSI/COSI)  worker-1     │
│  io-generator-1 ──► PVC (NFS/iSCSI/COSI)  worker-2     │
│  io-generator-2 ──► PVC (NFS/iSCSI/COSI)  worker-3     │
│         │                                                │
│         │ Headless Service (DNS estável)                 │
│         ▼                                                │
│  io-dashboard ──► NodePort :3000 ──► BROWSER            │
└─────────────────────────────────────────────────────────┘
```

Cada pod gerador expõe uma REST API (`/start`, `/stop`, `/metrics`). O dashboard consulta os 3 pods a cada segundo, agrega os dados e serve uma UI web com gráficos ao vivo via Chart.js.

---

## Funcionalidades

| Parâmetro | Opções |
|---|---|
| **Tamanho de bloco** | 2K, 4K, 8K, 16K, 32K, 64K, 128K, 256K, 512K, 1M, 2M, 4M, 8M |
| **Ratio leitura/escrita** | 0–100% por slider (ex: 70% read / 30% write) |
| **Padrão de acesso** | Sequencial, Aleatório, Burst |
| **Burst** | Duração do burst (1–30s) + tempo idle (1–60s) configuráveis |
| **Tipo de storage** | NFS, iSCSI, COSI (object storage S3-compatible) |

**Métricas monitoradas em tempo real:**
- IOPS de leitura e escrita (por pod e total)
- Largura de banda MB/s (por pod e total)
- Latência média em ms (por pod e média geral)
- Janela deslizante de 60 segundos nos gráficos

---

## Arquivos

| Arquivo | Descrição |
|---|---|
| `io-benchmark.yaml` | Deploy completo — aplique com `kubectl apply` |
| `dashboard-preview.html` | Preview do dashboard no browser sem cluster |

---

## Pré-requisitos

- Kubernetes 1.24+
- Pelo menos **3 worker nodes** (pods usam `podAntiAffinity` obrigatória)
- CSI driver instalado de acordo com o tipo de storage escolhido:
  - **NFS**: [`csi-driver-nfs`](https://github.com/kubernetes-csi/csi-driver-nfs)
  - **iSCSI**: [`democratic-csi`](https://github.com/democratic-csi/democratic-csi) ou similar
  - **COSI**: controller COSI + driver S3-compatible

---

## Quick Start

### 1. Configure o storage

Edite o `io-benchmark.yaml` de acordo com o tipo de storage desejado.

**NFS** (padrão):
```yaml
# StorageClass NFS — linhas 59–76
parameters:
  server: "192.168.1.100"        # ← IP do servidor NFS
  share: "/exports/io-benchmark" # ← path exportado
```

**iSCSI** (opcional):
```yaml
# StorageClass iSCSI — linhas 39–53
parameters:
  targetPortal: "192.168.1.100:3260"
  iqn: "iqn.2024-01.com.example:storage"
```
Depois troque no `volumeClaimTemplates` do StatefulSet:
```yaml
storageClassName: iscsi-storage   # linha 779
```

**COSI / Object Storage** (opcional):
1. Descomente as seções `BucketClass` e `BucketClaim` (linhas 83–104)
2. No StatefulSet, descomente `STORAGE_TYPE: "s3"` e as variáveis do Secret (linhas 716–737)

### 2. Aplique

```bash
kubectl apply -f io-benchmark.yaml
```

### 3. Aguarde os pods ficarem prontos

```bash
kubectl get pods -n io-benchmark -w
```

Os pods levam ~30–60s para subir (instalam dependências Python e criam o arquivo de teste de 10 GB no PVC).

```
NAME                   READY   STATUS    NODE
io-generator-0         1/1     Running   worker-1
io-generator-1         1/1     Running   worker-2
io-generator-2         1/1     Running   worker-3
io-dashboard-xxx-yyy   1/1     Running   worker-1
```

### 4. Acesse o dashboard

**Port-forward (recomendado para testes):**
```bash
kubectl port-forward -n io-benchmark svc/io-dashboard 3000:3000
```
Acesse: [http://localhost:3000](http://localhost:3000)

**NodePort (bare-metal / on-prem):**
```bash
NODE_IP=$(kubectl get nodes -o jsonpath='{.items[0].status.addresses[0].address}')
NODE_PORT=$(kubectl get svc io-dashboard -n io-benchmark \
  -o jsonpath='{.spec.ports[0].nodePort}')
echo "http://$NODE_IP:$NODE_PORT"
```

**LoadBalancer (cloud):**

Altere o tipo do Service no YAML e reaaplique:
```yaml
# linha ~880
type: LoadBalancer   # era NodePort
```
```bash
kubectl get svc io-dashboard -n io-benchmark   # aguarde EXTERNAL-IP
```

---

## Usando o Dashboard

1. Selecione o **Tipo de Armazenamento** (informativo — reflete o que foi configurado no YAML)
2. Escolha o **Tamanho de Bloco**
3. Ajuste o slider **Leitura / Escrita**
4. Selecione o **Padrão de Acesso**: Sequencial, Aleatório ou Burst
   - Em modo **Burst**: configure a duração ativa e o tempo de idle
5. Clique **▶ Iniciar** — o comando é enviado simultaneamente aos 3 pods
6. Os gráficos e cards atualizam em tempo real a cada **1 segundo**
7. Clique **■ Parar** para encerrar o IO em todos os pods

---

## API dos Geradores

Cada pod expõe uma REST API na porta `8080`. Você pode controlá-los diretamente:

```bash
# Iniciar IO em um pod específico
kubectl exec -n io-benchmark io-generator-0 -- \
  curl -s -X POST http://localhost:8080/start \
  -H 'Content-Type: application/json' \
  -d '{"block_size":"64k","read_pct":30,"pattern":"random"}'

# Ver métricas em JSON
kubectl exec -n io-benchmark io-generator-0 -- \
  curl -s http://localhost:8080/metrics | python3 -m json.tool

# Parar
kubectl exec -n io-benchmark io-generator-0 -- \
  curl -s -X POST http://localhost:8080/stop
```

### Parâmetros do `/start`

| Campo | Tipo | Padrão | Descrição |
|---|---|---|---|
| `block_size` | string | `"4k"` | Tamanho do bloco: `2k`…`8m` |
| `read_pct` | int | `50` | Percentual de leitura (0–100) |
| `pattern` | string | `"sequential"` | `sequential`, `random` ou `burst` |
| `burst_on` | int | `5` | Segundos em modo ativo (burst) |
| `burst_off` | int | `10` | Segundos em modo idle (burst) |

### Resposta do `/metrics`

```json
{
  "pod": "io-generator-0",
  "running": true,
  "storage_type": "file",
  "config": { "block_size": 65536, "read_pct": 70, "pattern": "random" },
  "metrics": {
    "read_iops": 842.3,
    "write_iops": 361.0,
    "read_bw_mb": 53.9,
    "write_bw_mb": 23.1,
    "read_lat_ms": 0.48,
    "write_lat_ms": 0.61,
    "phase": "active",
    "timestamp": 1718000000.0
  }
}
```

---

## Recursos Kubernetes Criados

| Recurso | Nome | Descrição |
|---|---|---|
| `Namespace` | `io-benchmark` | Namespace dedicado |
| `StorageClass` | `nfs-storage` | NFS via csi-driver-nfs |
| `StorageClass` | `iscsi-storage` | iSCSI via democratic-csi |
| `BucketClass` *(comentado)* | `cosi-benchmark-class` | COSI object storage |
| `BucketClaim` *(comentado)* | `io-benchmark-bucket` | Bucket COSI |
| `ConfigMap` | `io-generator-app` | Código Python do gerador |
| `ConfigMap` | `io-dashboard-app` | Código Python + HTML do dashboard |
| `StatefulSet` | `io-generator` | 3 pods geradores (anti-affinity) |
| `Service` (headless) | `io-generator-headless` | DNS estável por pod |
| `Deployment` | `io-dashboard` | Dashboard web |
| `Service` | `io-dashboard` | NodePort porta 3000 |

### PVCs criados automaticamente (via `volumeClaimTemplates`)

| Nome | Pod | Tamanho |
|---|---|---|
| `io-data-io-generator-0` | `io-generator-0` | 50Gi |
| `io-data-io-generator-1` | `io-generator-1` | 50Gi |
| `io-data-io-generator-2` | `io-generator-2` | 50Gi |

---

## Configurações Avançadas

### Alterar tamanho do arquivo de teste

Por padrão cada pod cria um arquivo de **10 GB** no PVC. Ajuste no StatefulSet:
```yaml
- name: TEST_FILE_SIZE_GB
  value: "20"   # GB
```

### Fixar porta NodePort

No Service `io-dashboard`:
```yaml
ports:
  - port: 3000
    targetPort: 3000
    nodePort: 30300   # descomente e ajuste
```

### Aumentar número de pods

Ajuste `replicas` no StatefulSet e `NUM_PODS` no Deployment do dashboard:
```yaml
# StatefulSet
spec:
  replicas: 5

# Deployment io-dashboard env
- name: NUM_PODS
  value: "5"
```
> Requer o mesmo número de worker nodes disponíveis (anti-affinity obrigatória).

---

## Limpeza

```bash
kubectl delete namespace io-benchmark
```

> Os PVCs e dados nos volumes são removidos junto com o namespace (política `reclaimPolicy: Delete`).

---

## Arquitetura Interna

```
Browser
  │  poll /api/metrics  (1s)
  │  POST /api/start
  ▼
io-dashboard :3000  (Flask)
  │  background thread: poll /metrics de cada pod (1s)
  │  POST /start → encaminha para todos os pods
  │
  │  DNS: io-generator-{0,1,2}.io-generator-headless.io-benchmark.svc.cluster.local
  ▼
io-generator-{0,1,2} :8080  (Flask)
  │  _file_worker: open(testfile), seek, read/write em loop
  │  _s3_worker:   boto3 PUT/GET (modo COSI)
  │  report a cada 1s → IOPS, BW, latência
  ▼
PVC (NFS / iSCSI / COSI)
```
