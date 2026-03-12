# Azure AKS Cost Estimate for LibreChat

## Context

LibreChat's Helm chart (`helm/librechat/`) defines a Kubernetes deployment with these services:

| Service | Purpose | Helm Default |
|---|---|---|
| **LibreChat API** | Node.js app (port 3080) | 1 replica, 6 GB Node heap |
| **MongoDB** | Primary datastore | Bitnami chart, no auth |
| **Meilisearch** | Full-text search | Bitnami chart, v1.7.3 |
| **Redis** | Caching (optional) | Disabled by default |
| **RAG API** | Embeddings/retrieval (optional) | Disabled by default |
| **PostgreSQL + pgvector** | Vector DB for RAG (optional) | Bitnami chart, required if RAG enabled |

Storage: 10 GB PVC for images (ReadWriteOnce).

---

## Cost Estimate: Core Setup (no RAG, no Redis)

**Region: West Europe (Netherlands) | Prices are monthly estimates in EUR**

> **Note:** West Europe is typically 10-15% more expensive than East US. EUR prices are
> based on Azure's published rates for the West Europe region. Verify exact current prices
> via the [Azure Pricing Calculator](https://azure.microsoft.com/en-us/pricing/calculator/)
> (set currency to EUR, region to West Europe).

### 1. AKS Control Plane

| Tier | Cost/month | Notes |
|---|---|---|
| Free | €0 | No SLA, fine for dev/staging |
| Standard | ~€70 | 99.95% SLA, recommended for production (~€0.10/hr) |

### 2. Compute (Node Pool VMs)

LibreChat + MongoDB + Meilisearch need ~2 nodes minimum for resilience.

| Option | VM Size | vCPUs | RAM | €/month each | Nodes | Total |
|---|---|---|---|---|---|---|
| **Small (dev/test)** | Standard_B2s (burstable) | 2 | 4 GB | ~€34 | 2 | **~€68** |
| **Recommended (prod)** | Standard_D2s_v5 | 2 | 8 GB | ~€80 | 2 | **~€160** |
| **Comfortable (prod)** | Standard_D2s_v5 | 2 | 8 GB | ~€80 | 3 | **~€240** |

### 3. Managed Databases (alternative to in-cluster)

Running MongoDB and PostgreSQL inside the cluster is cheapest but lacks managed backups/HA. Managed alternatives:

| Service | SKU | Cost/month |
|---|---|---|
| Azure Cosmos DB for MongoDB (RU) | 400 RU/s + 25 GB (free tier) | **€0** (free tier) |
| Azure Cosmos DB for MongoDB (RU) | 1000 RU/s + 50 GB | **~€55** |
| Azure Database for PostgreSQL | Burstable B1ms (1 vCore, 2 GB) | **~€28** |
| Azure Database for PostgreSQL | GP D2s_v3 (2 vCores, 8 GB) | **~€140** |

### 4. Storage

| Resource | Size | Type | Cost/month |
|---|---|---|---|
| Image PVC | 10 GB | Standard SSD (E1) | **~€1** |
| MongoDB data (in-cluster) | 8 GB | Standard SSD | **~€1** |
| Meilisearch data (in-cluster) | 8 GB | Standard SSD | **~€1** |
| OS disks (per node) | 128 GB x 2-3 | Standard SSD | **~€11-17** |

### 5. Networking

| Resource | Cost/month |
|---|---|
| Standard Load Balancer | **~€17** + €0.005/GB data |
| Public IP (static) | **~€4** |
| Egress bandwidth (first 100 GB free) | **~€0-9** |

---

## Scenario Summaries

### Scenario A: Dev/Test (Minimal)

| Component | Cost/month |
|---|---|
| AKS Free tier | €0 |
| 2x Standard_B2s nodes | €68 |
| In-cluster MongoDB + Meilisearch | €0 (runs on nodes) |
| Storage (PVCs + OS disks) | €14 |
| Load Balancer + IP | €21 |
| **Total** | **~€103/month** |

### Scenario B: Production (Recommended)

| Component | Cost/month |
|---|---|
| AKS Standard tier | €70 |
| 3x Standard_D2s_v5 nodes | €240 |
| In-cluster MongoDB + Meilisearch | €0 (runs on nodes) |
| Storage (PVCs + OS disks) | €20 |
| Load Balancer + IP | €21 |
| **Total** | **~€351/month** |

### Scenario C: Production with Managed DBs + RAG

| Component | Cost/month |
|---|---|
| AKS Standard tier | €70 |
| 3x Standard_D2s_v5 nodes | €240 |
| Azure Cosmos DB for MongoDB (1000 RU/s) | €55 |
| Azure Database for PostgreSQL (Burstable B1ms) | €28 |
| Redis (in-cluster or Azure Cache Basic C0) | €0-€50 |
| Storage (PVCs + OS disks) | €20 |
| Load Balancer + IP | €21 |
| **Total** | **~€434-€484/month** |

---

## Cost Optimization Tips

1. **Reserved Instances**: 1-year commitment saves ~35%, 3-year saves ~55% on VMs
2. **Spot nodes**: Use for non-critical workloads (up to 90% savings)
3. **Autoscaler**: Scale down to 1 node during off-hours
4. **Burstable VMs**: B-series for dev/test (significantly cheaper)
5. **Cosmos DB Free Tier**: 1000 RU/s + 25 GB free per subscription
6. **Start/Stop AKS**: Stop clusters during off-hours (no compute charges)

---

## Sources

- [Azure AKS Pricing](https://azure.microsoft.com/en-us/pricing/details/kubernetes-service/)
- [D2s v5 Pricing and Specs](https://instances.vantage.sh/azure/vm/d2s-v5)
- [Azure Cosmos DB for MongoDB Pricing](https://azure.microsoft.com/en-us/pricing/details/cosmos-db/mongodb/)
- [Azure Database for PostgreSQL Flexible Server Pricing](https://azure.microsoft.com/en-us/pricing/details/postgresql/flexible-server/)
- [Azure Managed Disks Pricing](https://azure.microsoft.com/en-us/pricing/details/managed-disks/)
- [Azure Load Balancer Pricing](https://azure.microsoft.com/en-us/pricing/details/load-balancer/)
- [Azure Pricing Calculator](https://azure.microsoft.com/en-us/pricing/calculator/)
