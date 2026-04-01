---
trigger: model_decision
description:  Gemini said  Best practices & anti-patterns for production K8s manifests. Covers workloads, security, networking, and GitOps. Sourced from official docs (v1.35), CIS Benchmarks, and CNCF guidance to ensure reliability and resource efficiency. Applie
---

---
activation_mode: glob
glob: "**/*.{yaml,yml}"
---

# Kubernetes Best Practices

Read fully before producing or reviewing any Kubernetes manifest. These rules are non-negotiable unless explicitly overridden.

---

## ❌ Anti-Patterns

### Images
- **Never use `latest` tag** — mutable, non-reproducible, triggers `imagePullPolicy: Always` by default causing `ErrImagePull` in air-gapped/rate-limited registries. Pin to immutable tags (git SHA, semver) or digests.
- **Never set `imagePullPolicy: Always`** without explicit justification. Default (`IfNotPresent`) is correct for pinned tags.
- **Never bake environment config into images** — no hardcoded IPs, URLs, passwords, or env-specific tags (`:dev`, `:prod`). One image must run in any environment via external config injection.

### Workloads
- **Never run bare Pods** — always use a controller (`Deployment`, `StatefulSet`, `DaemonSet`, `Job`). Bare pods are not rescheduled on node failure.
- **Never run a single replica** in production — minimum 2 to survive eviction and node drain without downtime.
- **Never concentrate all replicas on one node** — use pod anti-affinity or `topologySpreadConstraints`.
- **Never apply ad-hoc mutations** (`kubectl edit`, `kubectl patch`) in production — creates drift invisible to version control.

### Resources
- **Never omit `resources.requests`** — scheduler treats missing requests as near-zero, causing overcommit and node instability.
- **Never omit `resources.limits` for memory** — an unbounded container can exhaust node memory and invoke the OOM killer, taking down other workloads.
- **Never set `requests > limits`** — the pod will never be scheduled.
- **Avoid tight CPU limits** — CPU throttling causes latency spikes with no visible errors. Set accurate CPU requests; let pods burst. Memory limits are mandatory; CPU limits are optional.

### Health Probes
- **Never couple liveness probes to external dependencies** (DB, APIs, queues) — a slow downstream call kills healthy pods, cascading into a full restart loop.
- **Never use identical endpoints for liveness and readiness** — a not-ready pod gets killed instead of just removed from Service endpoints.
- **Never omit a readiness probe** — without it, traffic hits the pod the instant the container starts, before the app is ready.
- **Never check external service health inside a readiness probe** — downstream failure propagates upstream and takes down your entire frontend stack.
- **Don't use liveness probes for crash recovery** — apps should exit non-zero on fatal errors. Liveness is only for deadlock/hang detection.
- **Add a startup probe for slow-starting apps** — without it, liveness kills the pod during initialization and creates a restart loop.

### Security
- **Never run as root** unless explicitly required. Set `runAsNonRoot: true` + non-zero `runAsUser`.
- **Never use `privileged: true`** — violates PSS baseline; grants full host access equivalent to root on the node.
- **Never use wildcard `*` verbs/resources in RBAC** — grants cluster-admin equivalent. Enumerate exact verbs and resources.
- **Never bind to `cluster-admin`** unless required. Prefer namespace-scoped `Role`/`RoleBinding`.
- **Never use the `default` ServiceAccount** for workloads — it accumulates permissions over time.
- **Never store secrets in ConfigMaps or as plaintext env vars in manifests** — use External Secrets Operator or CSI Secrets Store Driver backed by AWS Secrets Manager or Vault.
- **Never use `hostPath` volumes** for application workloads — acceptable only for system DaemonSets (e.g. node-exporter) where the path is guaranteed.
- **Never use PodSecurityPolicy** — removed in Kubernetes 1.25. Use Pod Security Admission (PSA).

### Networking
- **Never leave namespaces without a NetworkPolicy** — default is fully open pod-to-pod communication across all namespaces.
- **Never hardcode IPs or CIDRs in manifests** — they overlap with cluster CIDRs and break across environments. Use Service DNS names.
- **Never use underscores in names/hostnames** — RFC-1123 allows only lowercase alphanumeric and hyphens; underscores silently break DNS.

### Operations
- **Never mix infra and app deployment in one pipeline** — different cadences, unclear failure ownership, and wasted time on infra for every app deploy.
- **Never co-locate production and non-production** in the same cluster — blast radius, noisy-neighbor contention, and RBAC cross-contamination.
- **Don't run stateful databases on K8s** without a mature operator (CloudNativePG, Vitess, Strimzi) and robust runbooks. Default to managed services (RDS, ElastiCache).
- **Don't use `ingress-nginx` for new deployments** — officially retiring (Nov 2025). Use Gateway API with a conformant controller.

---

## ✅ Best Practices

### Images & Manifests
- Pin all images to immutable tags or digests. Build once, promote across environments.
- Apply standard recommended labels to all resources: `app.kubernetes.io/name`, `version`, `component`, `part-of`, `managed-by`.
- Always set an explicit `namespace` on every resource — even `default` must be explicit.

### Workload Design
- Use `Deployment` for stateless, `StatefulSet` for stable identity/storage, `DaemonSet` for node agents, `Job`/`CronJob` for batch.
- Configure rolling update explicitly: `maxUnavailable: 0`, `maxSurge: 1` for zero-downtime deploys.
- Set `minReadySeconds` to prevent premature traffic routing during rollout.
- Use `topologySpreadConstraints` for multi-AZ distribution (preferred over pod anti-affinity for new manifests):
  ```yaml
  topologySpreadConstraints:
  - maxSkew: 1
    topologyKey: topology.kubernetes.io/zone
    whenUnsatisfiable: DoNotSchedule
    labelSelector:
      matchLabels:
        app.kubernetes.io/name: my-app
  ```
- Set `PodDisruptionBudgets` for all production workloads. Avoid `minAvailable: 1` on single-replica deployments — blocks node drain.

### Resources
- Right-size requests from Prometheus p95 actual usage. Use VPA in recommendation mode.
- Use HPA for horizontal scaling (CPU/memory), KEDA for event-driven scaling.
- Enforce defaults with `LimitRange`; enforce aggregate ceilings with `ResourceQuota` per namespace.

### Health Probes
- Readiness: HTTP check on a local `/ready` endpoint, `periodSeconds: 10`, `failureThreshold: 3`.
- Liveness: TCP socket or simple `/healthz` (zero dependencies), `periodSeconds: 30`, conservative `initialDelaySeconds`.
- Startup probe for slow apps: `failureThreshold: 30`, `periodSeconds: 10` — buys 5 min before liveness engages.

### Graceful Shutdown
- Handle `SIGTERM`: stop accepting connections, drain in-flight, then exit. Tune `terminationGracePeriodSeconds` accordingly.
- Add `preStop` sleep to absorb kube-proxy propagation lag:
  ```yaml
  lifecycle:
    preStop:
      exec:
        command: ["sh", "-c", "sleep 5"]
  ```
- Ensure PID 1 forwards signals — use `exec` CMD form, or wrap with `tini`/`dumb-init`.
- Close keep-alive connections during shutdown to prevent dropped requests.

### Security
- Set comprehensive `securityContext` on every container:
  ```yaml
  securityContext:
    runAsNonRoot: true
    runAsUser: 1000
    readOnlyRootFilesystem: true
    allowPrivilegeEscalation: false
    capabilities:
      drop: ["ALL"]
    seccompProfile:
      type: RuntimeDefault
  ```
- Enforce Pod Security Admission at namespace level (replaces PSP):
  ```yaml
  pod-security.kubernetes.io/enforce: restricted
  pod-security.kubernetes.io/warn: restricted
  ```
  Use `baseline` for workloads that legitimately can't meet `restricted`.
- Dedicated `ServiceAccount` per workload. Set `automountServiceAccountToken: false` for pods that don't call the K8s API.
- Scope RBAC: `Role`/`RoleBinding` over `ClusterRole`/`ClusterRoleBinding`. Audit regularly for `*` verbs, `*` resources, `cluster-admin` bindings.
- EKS: use IRSA for AWS API access — never mount static credentials. Enable control plane logging (api, audit, authenticator).
- Enable etcd encryption at rest. Forward audit logs to SIEM or S3.

### Networking
- Apply default-deny NetworkPolicy to every namespace, then explicitly whitelist required flows:
  ```yaml
  spec:
    podSelector: {}
    policyTypes: [Ingress, Egress]
  ```
- Use `ClusterIP` by default. `LoadBalancer` only for external exposure.
- New ingress requirements: use Gateway API (`HTTPRoute`) with a conformant controller (Envoy Gateway, Cilium, Istio).
- All inter-pod communication via DNS: `<svc>.<namespace>.svc.cluster.local`.

### Configuration & Secrets
- Externalize all env-specific config. `ConfigMap` for non-sensitive data; ESO/CSI driver for secrets.
- Never commit secrets to Git. Use Sealed Secrets, SOPS, or ESO for GitOps-safe secret management.
- Use IRSA on EKS for AWS API access. Rotate all static credentials regularly.

### Operations & GitOps
- All cluster state declared in Git. Helm with per-environment values files, or Kustomize overlays. No `kubectl edit` in production.
- Run `kubectl diff` before `kubectl apply`. Run `helm diff` before `helm upgrade`.
- Keep cluster version within N-2 of latest stable. Security patches end-of-life quickly.
- Use `kube-bench` (CIS Benchmark), `Falco` (runtime threats), `Trivy`/`Grype` (image CVE scanning in CI).
- Label all resources with `owner`, `team`, `repo`, `cost-center` for operability and cost attribution.

---

## 🚫 Removed / Deprecated APIs — Do Not Use

| Removed | Replacement | Since |
|---|---|---|
| `PodSecurityPolicy` | Pod Security Admission (PSA) | Removed k8s 1.25 |
| `ingress-nginx` | Gateway API (`HTTPRoute`) | Retiring Nov 2025 |
| `extensions/v1beta1` Ingress | `networking.k8s.io/v1` or Gateway API | Removed k8s 1.22 |
| `autoscaling/v2beta2` HPA | `autoscaling/v2` | Removed k8s 1.26 |
| `batch/v1beta1` CronJob | `batch/v1` | Removed k8s 1.25 |
| `policy/v1beta1` PDB | `policy/v1` | Removed k8s 1.25 |

Always validate API versions against the target cluster version before applying manifests.