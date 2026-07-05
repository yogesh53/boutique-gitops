# Online Boutique — ArgoCD Practice (4–5 hour sprint)

10 microservices + Redis, split per-service for an ApplicationSet.
Loadgenerator deliberately excluded (causes resource pressure on small nodes).

## BEFORE ANYTHING: find & replace
Replace `YOUR_GITHUB_USERNAME` in these 5 files with your GitHub username:
- bootstrap/root-app.yaml, bootstrap/project.yaml, bootstrap/appset-app.yaml,
  bootstrap/redis-app.yaml, appsets/services-appset.yaml

```bash
grep -rl YOUR_GITHUB_USERNAME . | xargs sed -i 's/YOUR_GITHUB_USERNAME/<your-username>/g'
```

## Run sheet

### Hour 0:00–0:30 — cluster + repo (parallel)
```bash
eksctl create cluster -f cluster.yaml          # runs ~20 min; do the next steps meanwhile
```
While it creates: push this folder to a NEW public GitHub repo named `boutique-gitops`.
```bash
git init && git add . && git commit -m "initial gitops repo"
git branch -M main
git remote add origin https://github.com/<you>/boutique-gitops.git
git push -u origin main
```

### Hour 0:30–1:00 — install ArgoCD + bootstrap
```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl wait --for=condition=available -n argocd deploy/argocd-server --timeout=300s

# UI access + password
kubectl port-forward svc/argocd-server -n argocd 8080:443 &
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo

# THE ONLY MANUAL APPLY — everything else comes from Git:
kubectl apply -f bootstrap/root-app.yaml
```
Watch the UI: root → creates project/appset-app/redis → ApplicationSet stamps out 10 service apps.
All green = app-of-apps + ApplicationSet drill DONE.
Get the frontend URL: `kubectl get svc frontend-external -n boutique` (LoadBalancer).

### Hour 1:00–1:45 — Drill: rollback (git revert)
1. Edit apps/frontend/deployment.yaml → change image tag to `v99.99` → commit+push
2. Watch: Synced but Degraded (ImagePullBackOff). THIS is the two-axes lesson.
3. `git revert HEAD && git push` → watch ArgoCD heal prod. Never touch the cluster.

### Hour 1:45–2:15 — Drill: selfHeal + prune
1. In appsets/services-appset.yaml change `automated: {}` to:
   `automated: {prune: true, selfHeal: true}` → commit+push
2. selfHeal test: `kubectl scale deploy/frontend -n boutique --replicas=5`
   → watch it snap back to Git's count within minutes.
3. prune test: `git rm -r apps/adservice && git push`
   → watch the adservice APP disappear, then its workloads (cascade).
   Restore: `git revert HEAD && git push` → adservice returns. Magic.

### Hour 2:15–2:45 — Drill: PreSync hook
1. Copy hooks/db-init-job.yaml into apps/cartservice/, add `db-init-job.yaml`
   to apps/cartservice/kustomization.yaml resources list. Push. Watch Job run pre-sync.
2. Change `exit 0` → `exit 1` in the Job, also bump cartservice image tag in the
   same commit. Push. Watch: sync FAILS, new cartservice version NEVER applies.
3. Fix back to exit 0, push, watch it go green.

### Hour 2:45–3:30 — Drill: AppProject + RBAC
1. Uncomment the dev-readonly role in bootstrap/project.yaml, push.
2. Create a local user: edit argocd-cm ConfigMap (accounts.dev: login),
   set password via `argocd account update-password --account dev`.
3. Map in argocd-rbac-cm: `g, dev, proj:boutique:dev-readonly`
4. Log in as dev → confirm you can SEE apps but Sync/Delete buttons are denied.

### Hour 3:30–4:30 — STRETCH (only if on schedule): External Secrets
- helm install external-secrets, create an AWS Secrets Manager secret,
  IRSA service account, ClusterSecretStore, one ExternalSecret for paymentservice.
- If short on time: SKIP. Do it in a later session; don't rush IAM.

### FINAL — always
```bash
eksctl delete cluster --name boutique-practice --region ap-south-1
```
Control plane costs $0.10/hr — never leave it running overnight.

## Gotchas
- PVCs: nothing here needs one (Redis runs emptyDir), so no EBS CSI driver needed today.
- If apps stay Progressing: check `kubectl get events -n boutique --sort-by=.lastTimestamp`
- If ApplicationSet creates nothing: repo URL typo or repo is private
  (make it public, or add repo credentials in ArgoCD settings).
