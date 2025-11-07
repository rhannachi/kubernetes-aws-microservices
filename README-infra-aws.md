# Déployer notre architecture principale sur AWS

## 1. Prérequis :
[Installation AWS CLI, eksctl et configuration d'un utilisateur AWS](README-aws-install-config.md)

## 2. Création du cluster

```bash
# Création du cluster Kubernetes avec eksctl
eksctl create cluster -f ./infra/cluster.yaml
```

```bash
# Afficher le manifeste de l'application principale
kubectl kustomize ./k8s/overlays/aws
```

```bash
# Déployer l'application principale
kubectl apply -k ./k8s/overlays/aws

# Déployer la stack ELK pour la journalisation (logging)
kubectl apply -k ./k8s/overlays/aws/logging
```

```bash
# Supprimer les ressources déployées
kubectl delete -k ./k8s/overlays/aws
kubectl delete -k ./k8s/overlays/aws/logging
```

```bash
# Supprimer complètement le cluster Kubernetes créé avec eksctl
eksctl delete cluster --name microservices-demo-cluster --region us-east-1
```
