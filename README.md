# homelab-gitops

Everything running on the homelab Kubernetes cluster (Talos on Proxmox), managed by **Argo CD**. Git is the source of truth: change a file, push, Argo CD makes the cluster match.

```
bootstrap/        applied once by hand: Argo CD + the root app
argocd-apps/      one Application per thing in the cluster (watched by "root")
infrastructure/   cluster-wide: storage class (later ingress, cert-manager…)
apps/hello-api/   base manifests + dev/prod overlays
```

## Before the first push: fill in placeholders

```bash
grep -rn "YOUR_GITHUB_USER\|NAS_IP_ADDRESS" .
```

- `YOUR_GITHUB_USER` → your GitHub username (all files)
- `NAS_IP_ADDRESS` and `/volume1/k8s` → your Synology IP and NFS share (`infrastructure/storage/storageclass-nfs.yaml`)

Keep this repo **public** (it contains no secrets), so Argo CD can read it without credentials. If you prefer it private, add the repo in the Argo CD UI under Settings → Repositories.

## Bootstrap (one time)

```bash
# 1. Install Argo CD
kubectl apply --server-side --force-conflicts -k bootstrap/argocd
kubectl -n argocd rollout status deploy/argocd-server

# 2. Get the admin password and open the UI
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath='{.data.password}' | base64 -d; echo
kubectl -n argocd port-forward svc/argocd-server 8080:443
#    -> https://localhost:8080  (user: admin)

# 3. Hand everything else over to Git
kubectl apply -f bootstrap/root.yaml
```

After step 3, Argo CD creates the NFS driver, the StorageClass and both hello-api environments from `argocd-apps/`.

## Daily use

- **dev** syncs automatically. hello-api's CI writes each new image tag into `apps/hello-api/overlays/dev`.
- **prod** is manual: copy the tag from dev to `apps/hello-api/overlays/prod/kustomization.yaml`, commit, then press **Sync** on `hello-api-prod` in the UI.
- Try the app: `kubectl -n hello-api-dev port-forward svc/hello-api 8000:80` → http://localhost:8000/version

## Check your YAML before pushing

```bash
kubectl kustomize apps/hello-api/overlays/dev
kubectl kustomize apps/hello-api/overlays/prod
```

## Next steps when you're ready

- Ingress controller + MetalLB (or Talos/Cilium L2) so apps get a real URL instead of port-forward
- Sealed Secrets for anything secret
- Let Argo CD manage its own install (move `bootstrap/argocd` into an Application)
