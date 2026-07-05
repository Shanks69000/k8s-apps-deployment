# Kubernetes Apps Deployment

Déploiement d'applications sur un cluster K3s avec Kustomize, network policies, quotas et limit ranges. Structure multi-environnement (base + overlays).

## Architecture cluster

```bash
K3s cluster (3 nodes)
├── k8s-master    "control plane"
├── k8s-worker1   "workload"
└── k8s-worker2   "workload"
Namespace: apps
├── homer         "dashboard interne"
└── it-tools      "boîte à outils IT"
```

## Sécurité et gouvernance

| Ressource | Rôle |
|---|---|
| `NetworkPolicy default-deny-all` | Bloque tout trafic ingress/egress par défaut |
| `NetworkPolicy allow-ingress-to-apps` | Autorise le trafic entrant sur les pods `tier: frontend` |
| `NetworkPolicy allow-dns-egress` | Autorise la résolution DNS (port 53) |
| `ResourceQuota` | Limite globale namespace : 2 CPU, 2Gi RAM requests, 10 pods max |
| `LimitRange` | Defaults par container : 100m/128Mi request, 250m/256Mi limit |

### Pourquoi ces ressources

**Requests et Limits** :
> Le scheduler K8s utilise les `requests` pour décider sur quel node placer un pod (le node doit avoir assez de CPU/RAM libre). Les `limits` définissent le plafond : si un container dépasse sa limit mémoire, le kernel OOM-kill le process. Sans limits, un pod qui fuit sa mémoire peut crasher le node entier et entraîner tous les autres pods avec lui. Sans requests, le scheduler empile les pods sans garantie de ressources et la première charge provoque de la contention.

**NetworkPolicies** :
> Par défaut dans K8s, tous les pods peuvent communiquer entre eux sans restriction. Le `default-deny-all` coupe tout trafic entrant et sortant dans le namespace. Ensuite on ouvre uniquement ce qui est nécessaire : le trafic ingress vers les pods labellisés `tier: frontend` (pour que le reverse proxy/ingress controller puisse les atteindre) et le DNS en egress (sans résolution DNS, les pods ne peuvent plus résoudre les noms de services internes comme `homer.apps.svc.cluster.local`). C'est le principe zero-trust appliqué au réseau K8s : tout est interdit sauf ce qui est explicitement autorisé.

**ResourceQuota** :
> Sans quota, n'importe quel déploiement dans le namespace `apps` peut consommer toutes les ressources du cluster. Le quota plafonne le namespace à 2 CPU / 2Gi RAM en requests et 10 pods max. Si un `kubectl apply` tente de dépasser ces seuils, l'API server refuse la création du pod. En environnement partagé (plusieurs équipes, plusieurs namespaces), c'est ce qui empêche un namespace de monopoliser le cluster au détriment des autres.

## Structure Kustomize

```bash
argocd/
├── application.yml           # Manifest ArgoCD Application (GitOps)
base/                         # Manifests communs
├── namespace.yml
├── network-policy.yml
├── resource-quota.yml
├── limit-range.yml
├── homer/
│   ├── deployment.yml        # Probes, resource limits, labels
│   ├── service.yml
│   └── ingress.yml
├── it-tools/
│   ├── deployment.yml
│   ├── service.yml
│   └── ingress.yml
└── kustomization.yml
overlays/prod/                # Patches environnement prod
├── patches/
│   ├── homer-replicas.yml    # Scale à 2 replicas
│   └── it-tools-replicas.yml
└── kustomization.yml
```

## Déploiement

```bash
# Base (1 replica par app)
kubectl apply -k base/

# Overlay prod (2 replicas par app)
kubectl apply -k overlays/prod/

# Vérifier
kubectl get all -n apps
kubectl get netpol -n apps
kubectl get quota -n apps
```

## GitOps avec ArgoCD

Le repo est synchronisé automatiquement via ArgoCD. Chaque push sur `main` déclenche un sync vers le cluster.

- **Auto-sync** : activé (pas de `kubectl apply` manuel)
- **Prune** : les ressources supprimées du repo sont supprimées du cluster
- **Self-heal** : toute modification manuelle sur le cluster est revertée à l'état Git

Le manifest ArgoCD Application est dans `argocd/application.yml`.

```bash
# Installation ArgoCD
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml --server-side

# Déploiement de l'application
kubectl apply -f argocd/application.yml
```

## Ce que ce projet démontre

- Kustomize avec overlays multi-environnement (base/prod)
- Network Policies zero-trust (default deny + whitelist)
- Resource Quotas et LimitRanges (gouvernance cluster)
- Probes readiness/liveness sur chaque deployment
- Resource requests/limits explicites par container
- GitOps avec ArgoCD (auto-sync, prune, self-heal)