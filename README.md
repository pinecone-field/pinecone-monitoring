# Pinecone Monitoring

Prometheus + Grafana monitoring for [Pinecone](https://www.pinecone.io) vector databases. Works with **SaaS (Serverless)**, **BYOC Public**, and **BYOC Private** deployments.

## What You Get

- **Pinecone Index Metrics** dashboard — record counts, storage, operation rates (query, upsert, fetch, update, delete), latency, read/write units. Works with all deployment types.
- **Kubernetes Cluster** dashboard (BYOC only) — node CPU/memory/disk, pod resources, PVC capacity.

### Dashboard Screenshots

![Index stats 1](%20screenshots/i-1.png)
![Index stats 2](%20screenshots/i-2.png)
![k8s stats 1](%20screenshots/k8-1.png)
![k8s stats 2](%20screenshots/k8-2.png)
![k8s stats 3](%20screenshots/k8-3.png)
![k8s stats 4](%20screenshots/k8-4.png)

## Quick Start — SaaS / Serverless

Monitor your Pinecone indexes with a single command using Docker Compose. No Kubernetes required.

### Prerequisites

- Docker and Docker Compose
- A Pinecone API key ([Standard or Enterprise plan](https://www.pinecone.io/pricing/))

### 1. Configure

```bash
cp .env.example .env
```

Edit `.env` with your values:

```
PINECONE_API_KEY=pcsk_...
PINECONE_PROJECT_ID=xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
```

You can find your project ID in the [Pinecone console](https://app.pinecone.io) URL: `app.pinecone.io/organizations/.../projects/<PROJECT_ID>/...`

### 2. Start

```bash
docker compose up -d
```

### 3. Open Grafana

Go to [http://localhost:3000](http://localhost:3000) and log in:

- **Username:** `admin`
- **Password:** `pinecone`

The **Pinecone Index Metrics** dashboard is pre-loaded and will start populating as soon as your indexes receive traffic.

### Stop

```bash
docker compose down        # keep data
docker compose down -v     # remove data volumes too
```

## Quick Start — BYOC

Deploy monitoring into your existing BYOC Kubernetes cluster. The script merges additional scrape jobs into the BYOC-managed Prometheus and deploys Grafana with pre-configured dashboards.

### Prerequisites

- `kubectl` configured with access to your BYOC cluster
- `helm` v3+
- `python3` with PyYAML (`pip install pyyaml`)
- A Pinecone API key
- SSH access to a bastion host in the BYOC VPC (for Grafana access)

### 1. Configure kubectl

```bash
# AWS
aws eks update-kubeconfig --region <REGION> --name <CLUSTER_NAME>

# GCP
gcloud container clusters get-credentials <CLUSTER_NAME> --region <REGION> --project <PROJECT>

# Azure
az aks get-credentials --resource-group <RG> --name <CLUSTER_NAME>
```

### 2. Set environment variables

```bash
export PINECONE_API_KEY="pcsk_..."
export PINECONE_PROJECT_ID="xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
export BYOC_METRICS_DOMAIN="preprod-aws-us-east-2-xxxx.byoc"
```

| Variable | Description | Example |
|---|---|---|
| `PINECONE_API_KEY` | Your Pinecone API key | `pcsk_abc123...` |
| `PINECONE_PROJECT_ID` | Project ID from the Pinecone console | `d2e983b7-8ead-...` |
| `BYOC_METRICS_DOMAIN` | BYOC domain prefix from your cluster | `preprod-aws-us-east-2-c97f.byoc` |

### 3. Deploy

```bash
./scripts/deploy-byoc.sh
```

The script will:

1. Auto-detect whether the cluster uses private or public access
2. Install `node-exporter` and `kube-state-metrics` via Helm
3. Create a Kubernetes secret for the Pinecone API key
4. Set up RBAC for Prometheus to scrape kubelet/cAdvisor metrics
5. Merge monitoring scrape jobs into the existing Prometheus config
6. Mount the API key into the Prometheus server pod
7. Deploy Grafana with pre-configured dashboards

### 4. Access Grafana

Grafana is exposed as a NodePort service on port **30300**. Since BYOC clusters run in a private VPC, use an SSH tunnel through your bastion host:

```bash
GRAFANA_NODE=$(kubectl get pods -n grafana -l app.kubernetes.io/name=grafana \
  -o jsonpath='{.items[0].status.hostIP}')

ssh -L 3000:$GRAFANA_NODE:30300 -A <USER>@<BASTION_IP>
```

Then open [http://localhost:3000](http://localhost:3000) and log in with:
- **Username:** `admin`
- **Password:** `pinecone-monitoring` (change in `grafana/grafana-values.yaml`)

## Dashboards

### Pinecone Index Metrics

Works with **all deployment types** (SaaS, BYOC Public, BYOC Private).

| Section | Panels |
|---|---|
| Index Overview | Total records, records per index, storage size per index |
| Operation Rates | Query, upsert, fetch, update, and delete rates per index; read/write units per second |
| Latency | Average query, upsert, and fetch latency per index |
| BYOC Infrastructure | Active index pods, CPU/memory usage per pod (collapsed by default, BYOC only) |

### Kubernetes Cluster (BYOC only)

| Section | Panels |
|---|---|
| Cluster Overview | Running pods, total nodes, CPU %, memory % |
| Node Resources | Per-node CPU, memory, disk usage; filesystem % bar gauge |
| Pod CPU | Top-10 pods by CPU, CPU by namespace (stacked), requests/limits/actual table |
| Pod Memory | Top-10 pods by memory, memory by namespace (stacked), requests/limits/actual table |
| Pod Storage | PVC usage bar gauge, capacity vs. used over time |

## How It Works

### SaaS Mode

Docker Compose runs Prometheus and Grafana locally. Prometheus scrapes the [Pinecone Metrics API](https://docs.pinecone.io/guides/production/monitoring) using HTTP service discovery to automatically find all indexes in your project.

```
┌─────────────────────────────────────────┐
│ Docker Compose                          │
│                                         │
│  ┌────────────────────┐                 │
│  │    Prometheus       │──── scrapes ──▶ Pinecone Metrics API
│  └────────┬───────────┘                 │  (api.pinecone.io)
│           │                             │
│  ┌────────▼───────────┐                 │
│  │     Grafana        │                 │
│  │  localhost:3000    │                 │
│  └────────────────────┘                 │
└─────────────────────────────────────────┘
```

### BYOC Mode

The deploy script adds scrape jobs to the existing BYOC-managed Prometheus. It auto-detects whether the cluster is private or public and configures the metrics endpoint accordingly (`metrics.private.*` vs `metrics.*`).

```
┌─────────────────────────────────────────────────────┐
│ BYOC Kubernetes Cluster                             │
│                                                     │
│  ┌──────────────┐  ┌──────────────┐                 │
│  │ node-exporter│  │kube-state-   │                 │
│  │ (DaemonSet)  │  │metrics       │                 │
│  └──────┬───────┘  └──────┬───────┘                 │
│         │                 │                         │
│  ┌──────▼─────────────────▼──────┐                  │
│  │       Prometheus Server       │◄── scrapes ──┐   │
│  │   (already in BYOC cluster)   │              │   │
│  └──────────────┬────────────────┘              │   │
│                 │                     ┌─────────┴─┐ │
│          ┌──────▼──────┐              │ Pinecone  │ │
│          │   Grafana   │              │ Metrics   │ │
│          │  (NodePort  │              │ API       │ │
│          │   :30300)   │              └───────────┘ │
│          └─────────────┘                            │
└─────────────────────────────────────────────────────┘
```

## Re-running After Pulumi Updates

If you run `pulumi up` or `pulumi up --refresh` on your BYOC cluster, Pulumi will reset the `prometheus-server` configmap to the BYOC defaults, removing the monitoring scrape jobs. Simply re-run the deploy script to restore them:

```bash
./scripts/deploy-byoc.sh
```

The script is idempotent and safe to run repeatedly.

## Multiple Projects

To monitor indexes across multiple Pinecone projects, add extra scrape jobs to `prometheus/prometheus-scrape-jobs.yaml`. For each additional project:

1. Create a separate secret:

```bash
kubectl create secret generic pinecone-api-key-2 \
  -n prometheus \
  --from-literal=api-key="<SECOND_API_KEY>"
```

2. Add a volume + volume mount to the Prometheus deployment patch for `/etc/pinecone-2/`

3. Duplicate the `pinecone-serverless-metrics` and `pinecone-byoc-metrics` jobs in the scrape config, changing:
   - `job_name` (must be unique)
   - `YOUR_PROJECT_ID` to the second project's ID
   - `credentials_file` to `/etc/pinecone-2/api-key`

## File Structure

```
pinecone-monitoring/
├── README.md
├── docker-compose.yaml                  # SaaS: Prometheus + Grafana
├── .env.example                         # Template for environment variables
├── grafana/
│   ├── grafana-values.yaml              # BYOC: Helm values for Grafana
│   ├── provisioning/                    # SaaS: Grafana provisioning configs
│   │   ├── datasources/
│   │   │   └── prometheus.yaml
│   │   └── dashboards/
│   │       └── default.yaml
│   └── dashboards/
│       ├── pinecone-index-metrics.json  # Index metrics (all modes)
│       └── kubernetes-cluster.json      # K8s cluster (BYOC only)
├── prometheus/
│   ├── prometheus.saas.yml              # SaaS: standalone Prometheus config
│   ├── prometheus-scrape-jobs.yaml      # BYOC: scrape jobs (merged into existing config)
│   ├── prometheus-secret-volume-patch.yaml  # BYOC: API key volume mount
│   └── node-exporter-values.yaml        # BYOC: node-exporter Helm values
└── scripts/
    ├── deploy-byoc.sh                   # BYOC: full deploy script
    └── uninstall-byoc.sh               # BYOC: cleanup script
```

## Uninstall

### SaaS

```bash
docker compose down -v
```

### BYOC

```bash
./scripts/uninstall-byoc.sh
```

This removes Grafana, node-exporter, kube-state-metrics, the API key secret, and monitoring RBAC. The Prometheus server itself is **not** removed, as it is managed by the BYOC cluster.

## Troubleshooting

### Pinecone API metrics showing "No data"

`pinecone_db_*` metrics only appear when your indexes are actively serving traffic. If indexes are idle, the metrics endpoint returns no data. Run some queries/upserts and check again.

**BYOC:** Verify targets in Prometheus:

```bash
kubectl port-forward -n prometheus svc/prometheus-server 9090:80
# Open http://localhost:9090/targets
```

**SaaS:** Check Prometheus directly at [http://localhost:9090/targets](http://localhost:9090/targets).

If targets show TLS certificate errors (e.g. `x509: certificate is valid for *.private.… not metrics.…`), the cluster is private-access but the deploy script did not detect it. Re-run `deploy-byoc.sh` -- it reads the `pc-pulumi-outputs` configmap to auto-detect.

### node-exporter pods not scheduling on all nodes (BYOC)

The BYOC cluster uses a Kyverno policy (`protect-dedicated-nodes`) that mutates pods with wildcard tolerations. The `node-exporter-values.yaml` uses a specific toleration key (`pinecone.io/dedicated`) to avoid this. If you see pod churn, check:

```bash
kubectl get pods -n prometheus -l app.kubernetes.io/name=prometheus-node-exporter
kubectl get clusterpolicy protect-dedicated-nodes -o yaml
```

### Grafana PVC multi-attach error on restart (BYOC)

The `grafana-values.yaml` sets `strategy.type: Recreate` to prevent this. If you still hit it:

```bash
kubectl patch deployment grafana -n grafana --type=json \
  -p '[{"op":"remove","path":"/spec/strategy/rollingUpdate"},{"op":"replace","path":"/spec/strategy/type","value":"Recreate"}]'
```

## References

- [Pinecone Monitoring Docs](https://docs.pinecone.io/guides/production/monitoring)
- [Pinecone BYOC Docs](https://docs.pinecone.io/guides/production/bring-your-own-cloud)
- [Prometheus Helm Chart](https://github.com/prometheus-community/helm-charts)
- [Grafana Helm Chart](https://github.com/grafana/helm-charts)

## License

MIT
