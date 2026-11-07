# homelab-gitops — notes for Claude

## About this repo
- Desired state of my homelab Kubernetes cluster. Argo CD applies whatever is on `main`.
- Cluster: 2 Talos Linux nodes on Proxmox. Storage: Synology NAS over NFS (csi-driver-nfs).
- I'm learning. Explain Kubernetes concepts briefly when you touch them.

## Layout
- `bootstrap/` — applied by hand once: Argo CD install + the `root` app-of-apps
- `argocd-apps/` — one Argo CD `Application` per thing running in the cluster
- `infrastructure/` — cluster-wide pieces (StorageClass, later ingress, cert-manager...)
- `apps/<name>/base` + `apps/<name>/overlays/{dev,prod}` — kustomize per app

## How changes work
- Change YAML → commit → push → Argo CD syncs. dev syncs automatically; prod is manual.
- To add an app: create `apps/<name>/...` and an Application in `argocd-apps/`.
- hello-api's CI bumps the image tag in `apps/hello-api/overlays/dev` — don't fight it.

## Rules
- Validate before saying done: `kubectl kustomize apps/hello-api/overlays/dev` (and prod).
- Pods must meet the "restricted" pod security standard (non-root, no privilege escalation,
  drop ALL capabilities, seccomp RuntimeDefault). Talos enforces at least "baseline".
- NEVER commit plain Secrets, kubeconfig or talosconfig. Ask me how to handle secrets
  (the plan is Sealed Secrets when we need them).
- Don't run `kubectl apply`/`delete` against the cluster unless I ask. Read-only
  `kubectl get/describe/logs` is fine for debugging.
- Pin chart and manifest versions; never use `latest` for infrastructure.
