# TP GitOps — Continuous Deployment avec ArgoCD

Pipeline GitOps de bout en bout : une application NGINX décrite avec Kustomize dans ce dépôt, déployée et maintenue par ArgoCD sur un cluster kind.

**Compte rendu complet : [docs/CR.md](docs/CR.md)**

## Contenu

| Chemin | Rôle |
|---|---|
| `app/base/` | Deployment, Service et page HTML (description commune) |
| `app/overlays/dev/` | Namespace, nombre de répliques et tag d'image ; fichier modifié pendant les démonstrations |
| `argocd/install/` | Installation d'ArgoCD v3.5.4 + polling Git réduit à 30 s |
| `argocd/application.yaml` | Application ArgoCD : auto-sync, prune, self-heal |
| `kind/cluster.yaml` | Cluster local, app exposée sur `localhost:8081` |

## Installation

Prérequis : Docker Desktop démarré, `kind`, `kubectl`, `gh`.

```bash
# 1. Cluster
kind create cluster --config kind/cluster.yaml

# 2. ArgoCD
kubectl create namespace argocd
kubectl apply --server-side -k argocd/install
kubectl -n argocd wait --for=condition=Available deployment --all --timeout=300s

# 3. Accès au dépôt privé (clé de déploiement en lecture seule, jamais commitée)
mkdir -p .secrets
ssh-keygen -t ed25519 -N "" -f .secrets/argocd-deploy-key
gh repo deploy-key add .secrets/argocd-deploy-key.pub --title argocd-gitops-lab
kubectl -n argocd create secret generic repo-gitops-argocd-lab \
  --from-literal=type=git \
  --from-literal=url=git@github.com:HanineKhemir/gitops-argocd-lab.git \
  --from-file=sshPrivateKey=.secrets/argocd-deploy-key
kubectl -n argocd label secret repo-gitops-argocd-lab argocd.argoproj.io/secret-type=repository

# 4. Application (dernier kubectl apply manuel ; la suite vient de Git)
kubectl apply -f argocd/application.yaml
```

## Accès

- Application : http://localhost:8081
- ArgoCD : `kubectl -n argocd port-forward svc/argocd-server 8080:443`, puis https://localhost:8080
  - utilisateur `admin`
  - mot de passe : `kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath='{.data.password}' | base64 -d`

## Démonstrations

| # | Action | Résultat attendu |
|---|---|---|
| 1 | Modifier `count` dans `app/overlays/dev/kustomization.yaml`, commit, push | Changement appliqué en ≤ 30 s, sans intervention |
| 2 | `kubectl -n demo delete deployment web` ou `kubectl -n demo scale deployment web --replicas=1` | Drift détecté et corrigé en quelques secondes |
| 3 | `newTag: does-not-exist`, commit, push | **Synced** mais **Degraded** (`ErrImagePull`) ; les anciens pods restent en service |
| 4 | UI : History and Rollback, revenir à une révision stable | **Healthy** / OutOfSync ; puis `git revert` et `kubectl apply -f argocd/application.yaml` pour revenir à **Synced** / Healthy avec auto-sync |
