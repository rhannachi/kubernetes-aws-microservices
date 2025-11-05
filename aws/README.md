
```bash
eksctl create cluster -f cluster.yaml
```

```bash
kubectl kustomize ./k8s/overlays/aws
```

```bash
kubectl apply -k ./k8s/overlays/aws
```

```bash
eksctl delete cluster --name microservices-demo-cluster --region us-east-1 
```

