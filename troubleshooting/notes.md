# Kubernetes Troubleshooting Guide – Master Index

A comprehensive, example-driven reference for common Kubernetes issues.  
Each topic has its own detailed file with **symptoms, causes, exact YAML examples, and step-by-step resolutions**.

---

## Topic Files

| # | Topic | File | What's Covered |
|---|-------|------|----------------|
| 1 | **Pod Issues** | [pod-issues.md](./pod-issues.md) | CrashLoopBackOff, ImagePullBackOff, Pending, OOMKilled, Stuck Terminating, CreateContainerConfigError, RunContainerError, Init container failures |
| 2 | **Service & Networking** | [service-networking.md](./service-networking.md) | No Endpoints, ClusterIP access, pod-to-pod connectivity, port mismatches, NetworkPolicy debugging |
| 3 | **Storage & Volumes** | [storage-volumes.md](./storage-volumes.md) | PVC Pending, Permission Denied, Multi-Attach errors, dynamic provisioning failures |
| 4 | **Node Issues** | [node-issues.md](./node-issues.md) | NotReady, Disk/Memory Pressure, Taints, node draining, kubelet crashes |
| 5 | **Deployment & Scaling** | [deployment-scaling.md](./deployment-scaling.md) | Rollout stuck, HPA not scaling, rolling update downtime, replica mismatches |
| 6 | **RBAC & Security** | [rbac-security.md](./rbac-security.md) | Forbidden errors, ServiceAccount tokens, PodSecurityPolicy/Standards, SecurityContext |
| 7 | **DNS Issues** | [dns-issues.md](./dns-issues.md) | CoreDNS failures, internal/external resolution, ndots tuning, custom DNS |
| 8 | **Cluster-Level Issues** | [cluster-issues.md](./cluster-issues.md) | etcd problems, API server down, certificate expiry, control plane recovery |
| 9 | **Helm Issues** | [helm-issues.md](./helm-issues.md) | Install/Upgrade failures, stuck releases, values debugging, chart templating |
| 10 | **Ingress Issues** | [ingress-issues.md](./ingress-issues.md) | 404/502 errors, TLS/HTTPS, path routing, ingress controller debugging |
| 11 | **Logging & Monitoring** | [logging-monitoring.md](./logging-monitoring.md) | Empty logs, Metrics Server, Prometheus/Grafana setup issues |
| 12 | **Debugging Commands** | [debugging-commands.md](./debugging-commands.md) | Cheat sheet of all essential kubectl debugging commands with examples |

---

## Quick Troubleshooting Flowchart

```
Pod not running?
│
├─ Status: Pending
│  ├─ "Insufficient cpu/memory"     → See pod-issues.md § 1.3
│  ├─ "no PV to bind"              → See storage-volumes.md § 3.1
│  └─ "node selector mismatch"     → See node-issues.md § 4.3
│
├─ Status: CrashLoopBackOff
│  ├─ App error in logs             → See pod-issues.md § 1.1
│  ├─ Missing ConfigMap/Secret      → See pod-issues.md § 1.6
│  └─ Probe failing                 → See pod-issues.md § 1.1 (probes)
│
├─ Status: ImagePullBackOff
│  ├─ Wrong image/tag               → See pod-issues.md § 1.2
│  └─ Private registry auth         → See pod-issues.md § 1.2
│
├─ Status: Running but unreachable
│  ├─ Service has no endpoints       → See service-networking.md § 2.1
│  ├─ NetworkPolicy blocking         → See service-networking.md § 2.3
│  └─ DNS not resolving              → See dns-issues.md § 7.1
│
└─ Cluster-wide problems
   ├─ API server unreachable         → See cluster-issues.md § 8.2
   ├─ Certificate expired            → See cluster-issues.md § 8.3
   └─ Node NotReady                  → See node-issues.md § 4.1
```

---

*Last updated: 2026-10-31*
