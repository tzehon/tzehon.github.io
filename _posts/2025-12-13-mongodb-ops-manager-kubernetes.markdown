---
layout: post
title:  "Deploying MongoDB Ops Manager on Kubernetes with cert-manager"
date:   2025-12-13 00:00:00 +0800
categories: mongodb kubernetes ops-manager tls
---

This post walks through deploying MongoDB Ops Manager on Kubernetes using the MongoDB Controllers for Kubernetes (MCK) operator. The setup includes automated TLS certificate management via cert-manager, backup infrastructure, and external access configuration.

## Why Kubernetes for Ops Manager?

Running Ops Manager on Kubernetes brings several advantages:

- **Declarative configuration** - Define your entire deployment as YAML manifests
- **Automated recovery** - Kubernetes restarts failed pods automatically
- **Consistent environments** - Same deployment process for dev, staging, production
- **Resource management** - CPU and memory limits enforced by the platform

The challenge is that MongoDB's Kubernetes operators have evolved significantly. The original MongoDB Enterprise Kubernetes Operator (MEKO) has been deprecated in favor of MongoDB Controllers for Kubernetes (MCK), which uses a Helm-based installation approach.

## Architecture Overview

```
                         Kubernetes Cluster
+------------------------------------------------------------------+
|                                                                  |
|  +------------------------------------------------------------+  |
|  |  MongoDB Controllers (MCK) - Helm deployed                 |  |
|  +------------------------------------------------------------+  |
|                              |                                   |
|         +--------------------+--------------------+              |
|         v                    v                    v              |
|  +-------------+      +-------------+      +-------------+       |
|  | Ops Manager |      | Backup      |      | Production  |       |
|  |-------------|      |-------------|      |-------------|       |
|  | OM Pod:8443 |      | Oplog (3)   |      | ReplicaSet  |       |
|  | AppDB (3)   |      | Blockstore  |      | Sharded     |       |
|  +-------------+      +-------------+      +-------------+       |
|                                                                  |
|  +------------------------------------------------------------+  |
|  |  cert-manager - TLS lifecycle management                   |  |
|  +------------------------------------------------------------+  |
|                                                                  |
+------------------------------------------------------------------+
```

The deployment includes:
- **Ops Manager** with a 3-node Application Database
- **Backup infrastructure** with oplog store and blockstore for point-in-time recovery
- **cert-manager** for automated TLS certificate provisioning and renewal
- **Production databases** (ReplicaSets or Sharded Clusters) managed by Ops Manager

## From MEKO to MCK

If you've worked with MongoDB on Kubernetes before, you might have used MEKO (MongoDB Enterprise Kubernetes Operator). The ecosystem has evolved:

| Aspect | MEKO (Deprecated) | MCK (Current) |
|--------|-------------------|---------------|
| Installation | kubectl apply CRDs + operator YAML | Helm chart |
| CRDs | MongoDBOpsManager, MongoDB | Same, but managed by Helm |
| Updates | Manual YAML updates | helm upgrade |
| Configuration | ConfigMaps + environment variables | Helm values |

The migration path is straightforward if you're using standard configurations. The CRDs remain compatible; only the operator deployment changes.

## TLS with cert-manager

One of the significant improvements in this project was replacing manual certificate generation (using cfssl) with [cert-manager](https://cert-manager.io/). This automates the entire TLS lifecycle:

### Setting Up the Certificate Chain

```yaml
# ClusterIssuer for self-signed CA
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: selfsigned-issuer
spec:
  selfSigned: {}

---
# CA Certificate
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: mongodb-ca
  namespace: mongodb
spec:
  isCA: true
  commonName: mongodb-ca
  secretName: mongodb-ca-secret
  issuerRef:
    name: selfsigned-issuer
    kind: ClusterIssuer

---
# Issuer using the CA
apiVersion: cert-manager.io/v1
kind: Issuer
metadata:
  name: mongodb-ca-issuer
  namespace: mongodb
spec:
  ca:
    secretName: mongodb-ca-secret
```

### Requesting Certificates for Ops Manager

```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: opsmanager-cert
  namespace: mongodb
spec:
  secretName: opsmanager-cert
  duration: 8760h  # 1 year
  renewBefore: 720h  # Renew 30 days before expiry
  subject:
    organizations:
      - MongoDB
  commonName: opsmanager-svc.mongodb.svc.cluster.local
  dnsNames:
    - opsmanager-svc.mongodb.svc.cluster.local
    - opsmanager-svc.mongodb.svc
    - opsmanager-svc
    - "*.mongodb.svc.cluster.local"
  issuerRef:
    name: mongodb-ca-issuer
    kind: Issuer
```

cert-manager handles renewal automatically. No more scrambling when certificates expire.

## External Access Solutions

Getting connectivity from outside the Kubernetes cluster is often the trickiest part. There are two main approaches:

### Split-Horizon DNS (ReplicaSets)

For ReplicaSets, split-horizon DNS lets clients inside and outside the cluster use different DNS names that resolve to the same MongoDB members:

```yaml
spec:
  connectivity:
    replicaSetHorizons:
      - internal: "rs0-0.mongodb.svc.cluster.local:27017"
        external: "rs0-0.mdb.example.com:27017"
      - internal: "rs0-1.mongodb.svc.cluster.local:27017"
        external: "rs0-1.mdb.example.com:27017"
      - internal: "rs0-2.mongodb.svc.cluster.local:27017"
        external: "rs0-2.mdb.example.com:27017"
```

You'll need to configure DNS records and firewall rules to route external traffic to the appropriate NodePorts or LoadBalancers.

### LoadBalancer Services (Sharded Clusters)

For sharded clusters, exposing mongos routers via LoadBalancer services is more practical:

```bash
# The deployment script automatically:
# 1. Creates LoadBalancer services for each mongos
# 2. Waits for external IPs to be assigned
# 3. Regenerates mongos TLS certificates with external DNS names
```

The certificate regeneration step is crucial - mongos must have certificates that include the external hostname, or TLS verification fails.

## Backup Infrastructure

Point-in-time recovery requires two components:

| Component | Purpose | Configuration |
|-----------|---------|---------------|
| **Oplog Store** | Continuous oplog tailing | 3-node ReplicaSet |
| **Blockstore** | Snapshot storage | 3-node ReplicaSet |

Default retention settings:
- Snapshots: Every 24 hours, retained for 2 days
- Weekly snapshots: Retained for 2 weeks
- Monthly snapshots: Retained for 1 month
- Point-in-time window: 1 day

These are conservative defaults suitable for demonstrations. Production deployments typically need longer retention.

## Resource Requirements

This setup is not lightweight. Minimum recommended resources:

| Component | CPU | Memory | Storage |
|-----------|-----|--------|---------|
| Ops Manager | 8 cores | 32 GB | 50 GB |
| AppDB (3 nodes) | 12 cores | 48 GB | 150 GB |
| Oplog Store | 6 cores | 24 GB | 200 GB |
| Blockstore | 6 cores | 24 GB | 500 GB |
| **Total** | **32+ cores** | **128+ GB** | **900+ GB** |

For demonstration purposes, you can reduce these significantly, but performance will suffer.

## Common Pitfalls

### Pods Stuck in Pending

Usually a resource issue. Check node capacity:

```bash
kubectl describe nodes | grep -A 5 "Allocated resources"
```

### Ops Manager Not Becoming Ready

The Application Database must be healthy first:

```bash
kubectl get mongodb -n mongodb -w
# Wait for appdb to show Running
```

### TLS Certificate Errors

Regenerate certificates if they don't include the correct hostnames:

```bash
kubectl delete certificate opsmanager-cert -n mongodb
# cert-manager will recreate it
```

### External Clients Can't Connect

Check that:
1. Certificates include external hostnames
2. Firewall rules allow traffic to NodePorts/LoadBalancer IPs
3. DNS records resolve correctly

## Lessons Learned

1. **Start with cert-manager** - Manual certificate management doesn't scale
2. **Plan external access early** - Retrofitting split-horizon DNS is painful
3. **Monitor resource usage** - Ops Manager and backup infrastructure are resource-hungry
4. **Use Helm values** - Customize MCK via values.yaml, not by editing generated manifests

The full deployment scripts and templates are available at [research/mongodb-ops-manager-kubernetes](https://github.com/tzehon/research/tree/main/mongodb-ops-manager-kubernetes).
