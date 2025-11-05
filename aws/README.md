# Déployer notre architecture principale sur AWS

## 1. Prérequis :
[Installation AWS CLI, eksctl et configuration d'un utilisateur AWS](../README-aws-install-config.md)

## 2. Création du cluster

```bash
# Création du cluster Kubernetes avec eksctl
eksctl create cluster -f cluster.yaml
```

```bash
# Affichage du manifeste YAML généré par Kustomize
kubectl kustomize ../k8s/overlays/aws
```

```bash
# Application des ressources Kubernetes définies par Kustomize dans le cluster
kubectl apply -k ../k8s/overlays/aws
```

```bash
# Suppression des ressources Kubernetes définies par Kustomize dans le cluster
kubectl delete -k ../k8s/overlays/aws
```

```bash
# Suppression complète du cluster Kubernetes créé via eksctl
eksctl delete cluster --name microservices-demo-cluster --region us-east-1
```
