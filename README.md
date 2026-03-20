# Demo ArgoCD — DevOps News

Application de démonstration Flask + Redis + Nginx deployee via ArgoCD sur GKE.

## Pre-requis

- Un cluster GKE operationnel avec `kubectl` configure
- La CLI `argocd` installée ([installation](https://argo-cd.readthedocs.io/en/stable/cli_installation/))

## 1. Installer Argo CD sur le cluster

```bash
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/v3.3.4/manifests/install.yaml
```

Attendre que tous les pods soient prêts :

```bash
kubectl wait --for=condition=ready pod --all -n argocd --timeout=120s
```

## 2. Exposer l'interface ArgoCD

```bash
kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "LoadBalancer"}}'
```

Récuperer l'IP externe (peut prendre 1-2 minutes sur GKE) :

```bash
kubectl get svc argocd-server -n argocd -w
```

Une fois l'`EXTERNAL-IP` attribuee, l'interface est accessible sur `https://<EXTERNAL-IP>`.

## 3. Recuperer le mot de passe admin

```bash
kubectl get secret argocd-initial-admin-secret -n argocd -o jsonpath='{.data.password}' | base64 -d; echo
```

Identifiants de connexion :
- **Utilisateur** : `admin`
- **Mot de passe** : celui affiche par la commande ci-dessus

## 4. Deployer l'application

```bash
kubectl apply -f environments/dev/argocd-application.yaml
```

ArgoCD va automatiquement :
1. Cloner le repo `https://github.com/akoudri/news-argocd.git` (branche `main`)
2. Rendre le chart Helm `charts/devops-news` avec les valeurs `environments/dev/values-dev.yaml`
3. Deployer les ressources dans le namespace `ali`

Suivre la progression :

```bash
kubectl get application devops-news-dev -n argocd -w
```

L'application est prete quand le statut passe a `Synced` / `Healthy`.

## 5. Verifier le deploiement

```bash
kubectl get all -n ali -l app.kubernetes.io/instance=devops-news-dev
```

Composants deployes :

| Composant | Type | Role |
|-----------|------|------|
| **backend** | Deployment | API Flask qui collecte des news |
| **frontend** | Deployment | Reverse-proxy Nginx |
| **redis** | StatefulSet | Stockage des news |
| **cleaner** | CronJob | Nettoyage periodique des anciennes news |

## 6. Nettoyage

Supprimer l'application (et toutes ses ressources Kubernetes) :

```bash
kubectl delete -f environments/dev/argocd-application.yaml
```

Desinstaller ArgoCD :

```bash
kubectl delete namespace argocd
```

## Structure du projet

```
.
├── charts/devops-news/          # Chart Helm de l'application
│   ├── templates/               # Manifests Kubernetes templatises
│   └── values.yaml              # Valeurs par defaut
└── environments/
    ├── dev/
    │   ├── argocd-application.yaml  # Manifest ArgoCD (Application CRD)
    │   └── values-dev.yaml          # Surcharges pour l'environnement dev
    └── prod/
        └── values-prod.yaml         # Surcharges pour l'environnement prod
```
