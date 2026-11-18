# Agentic Enterprise GitOps

Desired Kubernetes state for the Agentic Enterprise Lab.

## Layout

- `apps/agent-runtime/base`: reusable Deployment and Service.
- `apps/agent-runtime/overlays/k3s`: K3s deployment target watched by Argo CD.
- `platform/namespaces`: administrator-managed namespaces.
- `platform/security/agent-runtime`: administrator-managed ServiceAccount and
  read-only runtime RBAC.
- `platform/security/argocd-agent-runtime-sync`: dedicated ServiceAccount and
  least-privilege RBAC for Argo CD synchronization.
- `platform/argocd/overlays/k3s`: pinned, non-HA Argo CD v3.6.1 installation.
- `clusters/k3s/argocd`: restricted Agent Runtime project and application.

Image promotion updates the immutable `newTag` in the K3s overlay. The base
retains the current image reference during migration so its standalone output
is also pinned.

## Bootstrap order

This configuration targets Argo CD v3.6.1. Application sync impersonation is a
beta feature available in this version and is enabled with strict enforcement:
every project must map its destination to a ServiceAccount or synchronization
fails rather than falling back to the controller identity.

The K3s overlay replaces the upstream controller's unrestricted ClusterRole
with read-only access to Deployments, ReplicaSets, Pods, and Services plus
permission to create SelfSubjectAccessReviews. It removes the upstream server
ClusterRole and ClusterRoleBinding. It also removes the unnecessary
cluster-wide ApplicationSet binding; the namespaced ApplicationSet Role remains
available. UI operations use the existing exact ServiceAccount impersonation
grant instead. `resource.respectRBAC` is strict.

The overlay sets baseline requests of 25m CPU and 64Mi memory for small
components. The application controller requests 100m/256Mi, the repository
server 100m/256Mi, and the API server 50m/128Mi. Combined requests are about
350m CPU and 896Mi memory, which fits the confirmed four-core, eight-GiB lab.

Validate locally before installation:

```bash
kubectl kustomize platform/argocd/overlays/k3s >/tmp/argocd-install.yaml
kubectl kustomize platform >/tmp/platform.yaml
kubectl kustomize clusters/k3s/argocd >/tmp/agent-runtime-argocd.yaml
```

An administrator installs the control plane and security prerequisites in this
order:

```bash
kubectl apply --server-side -k platform/argocd/overlays/k3s
kubectl rollout status statefulset/argocd-application-controller -n argocd
kubectl rollout status deployment/argocd-server -n argocd
kubectl rollout status deployment/argocd-repo-server -n argocd
kubectl apply -k platform/
```

After materializing the read-only repository credential outside Git, register
the project and application:

```bash
kubectl apply -k clusters/k3s/argocd/
```

Verify the effective permissions before the first sync:

```bash
kubectl auth can-i list pods --all-namespaces \
  --as=system:serviceaccount:argocd:argocd-application-controller
kubectl auth can-i patch deployments --all-namespaces \
  --as=system:serviceaccount:argocd:argocd-application-controller
kubectl auth can-i list pods --all-namespaces \
  --as=system:serviceaccount:argocd:argocd-server
kubectl auth can-i impersonate users \
  --resource-name=system:serviceaccount:agentic-lab:argocd-agent-runtime-sync \
  --as=system:serviceaccount:argocd:argocd-server
kubectl auth can-i patch deployments -n agentic-lab \
  --as=system:serviceaccount:agentic-lab:argocd-agent-runtime-sync
kubectl auth can-i delete deployments -n agentic-lab \
  --as=system:serviceaccount:agentic-lab:argocd-agent-runtime-sync
kubectl auth can-i get pods/log -n agentic-lab \
  --as=system:serviceaccount:agentic-lab:argocd-agent-runtime-sync
```

Expected results are `yes`, `no`, `no`, `yes`, `yes`, `no`, and `yes`.
Kubernetes v1.37.2+k3s1 is within the Argo CD 3.6 tested range.

The Agent Runtime application uses manual synchronization. Automated sync is
omitted, so pruning and self-healing are disabled. The namespace is not created
by Argo CD and the platform prerequisites must exist before the first sync.

The restricted project accepts manifests only from this repository and targets
only `agentic-lab`. It permits `Deployment` and `Service` plus their read-only
`ReplicaSet` and `Pod` children so the resource tree remains visible. Argo CD
impersonates `argocd-agent-runtime-sync` for comparison, synchronization, and
UI/API operations. That ServiceAccount can write only Deployments and Services;
ReplicaSets, Pods, Pod logs, and Events are read-only. It cannot delete any
resource, so pruning remains unavailable. The existing manual `codex-deployer`
identity is not reused or modified.

## Private repository authentication

Add a dedicated SSH Deploy Key to `agentic-enterprise-gitops` with **Allow write
access** disabled. Store the private key in a secret manager, then materialize
an Argo CD repository Secret in `argocd`.
`clusters/k3s/argocd/repository-secret.example.yaml` documents the required
shape but is intentionally excluded from Kustomize and must never contain real
credentials in Git. The v3.6.1 installation already supplies verified
`github.com` host keys in `argocd-ssh-known-hosts-cm`; do not disable host-key
checking.

The `argocd-cm` settings are merged into the pinned installation's populated
ConfigMap by Kustomize, preserving upstream defaults. The standard
`argocd-application-controller` and `argocd-server` ServiceAccounts are granted
permission to impersonate only the exact
`system:serviceaccount:agentic-lab:argocd-agent-runtime-sync` username. This
grant is a resource-name-restricted ClusterRole because Kubernetes
impersonation authorization is cluster-scoped; the impersonated account's
effective Deployment and Service permissions remain namespace-scoped.
