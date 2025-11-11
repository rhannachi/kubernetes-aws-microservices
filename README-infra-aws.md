# Déployer notre architecture principale sur AWS

## 1. Prérequis

Avant de commencer, assure-toi d’avoir installé et configuré correctement :

* **AWS CLI** (avec tes credentials configurés via `aws configure`)
* **eksctl** pour créer et gérer des clusters EKS
* Un utilisateur AWS avec les permissions nécessaires pour créer des ressources EKS, VPC, LoadBalancer, IAM, etc.

> Voir le guide complet : [Installation AWS CLI, eksctl et configuration](README-aws-install-config.md)

Pour installer **Helm**, le gestionnaire de packages pour Kubernetes, il est recommandé de suivre les instructions officielles fournies par la documentation Helm.
> Tu peux consulter la page officielle ici : [https://helm.sh/docs/intro/install/](https://helm.sh/docs/intro/install/)


---

## 2. Création du cluster Kubernetes

Créer un cluster EKS en utilisant un fichier de configuration `cluster.yaml` :

```bash
eksctl create cluster -f ./infra/cluster.yaml
```

> Ce fichier définit le nombre de nœuds, le type d’instance, le VPC, les sous-réseaux et d’autres paramètres du cluster.

Vérifie que les nœuds sont opérationnels :

```bash
kubectl get nodes
```

> Tu devrais voir tous les nœuds du cluster en `Ready`.

---

## 3. Déployer les ressources de base avec Kustomize

Avant de déployer le monitoring et le logging, applique les ressources de base (workloads, services, ConfigMaps…) :

```bash
kubectl apply -k ./k8s/overlays/aws
```

**Vérification :**

```bash
kubectl get all
```

> Cette étape déploie tous les composants nécessaires pour que ton application fonctionne sans la partie monitoring/logging.\
> Astuce : Kustomize permet de gérer des overlays pour différents environnements (AWS, staging, production…).

---

## 4. Déployer le logging (Elasticsearch, Fluent Bit, Kibana)

```bash
kubectl apply -k ./k8s/overlays/aws/logging
```

**Vérification :**

```bash
kubectl get all -n logging
```

> Fluent Bit collecte les logs des pods Kubernetes et les envoie vers Elasticsearch.\
> Kibana permet de visualiser ces logs et sera exposé via un LoadBalancer AWS.

Exemple de service Kibana exposé :

```text
NAME      TYPE           CLUSTER-IP       EXTERNAL-IP                                                               PORT(S)   AGE
kibana    LoadBalancer   172.20.186.47    ae7ed72eb20f24f2a87d51933a43-1074990582.us-east-1.elb.amazonaws.com   5601:31158/TCP   119m
```

> Tu peux accéder à Kibana via l’EXTERNAL-IP pour tester l’affichage des logs.

---

## 5. Installer kube-prometheus-stack avec Helm

Avant l’installation, crée le namespace `monitoring` :

```bash
kubectl create namespace monitoring
```

Ajoute le repo Helm de Prometheus et mets-le à jour :

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo update
```

Installe le chart avec ton fichier de valeurs personnalisé :

```bash
helm install kube-prometheus-stack prometheus-community/kube-prometheus-stack \
  -n monitoring \
  -f ./k8s/overlays/aws/monitoring/kube-prometheus-values.yaml
```

**Ce déploiement inclut :**

* **Prometheus** pour collecter les métriques Kubernetes
* **Grafana** pour visualiser les métriques et logs
* **Alertmanager** pour gérer les alertes
* **NodeExporter** et **kube-state-metrics** pour exposer les métriques systèmes et Kubernetes
* **ServiceMonitor** pour scrapper Fluent Bit et autres composants

**Vérification :**

```bash
kubectl get all -n monitoring
```

> Grafana et Prometheus seront accessibles via LoadBalancer et Grafana aura déjà **Elasticsearch** comme datasource pour les logs.
```text
NAME                                                     TYPE           CLUSTER-IP       EXTERNAL-IP                                                               PORT(S)                         AGE
service/kube-prometheus-stack-grafana                    LoadBalancer   172.20.182.208   a31873ea64e224baa925825bf-1857922259.us-east-1.elb.amazonaws.com   80:32267/TCP                    82s
service/kube-prometheus-stack-prometheus                 LoadBalancer   172.20.107.133   a8c0a9cc60c5046f44517a322be-24384938.us-east-1.elb.amazonaws.com     9090:31427/TCP,8080:32462/TCP   82s
```

---

## 6. Vérification et tests

1. **Logs Fluent Bit :**
   Dans Grafana → Explore → sélectionne la datasource Elasticsearch → utilise une query type :

   ```
   kubernetes.namespace_name:"logging" AND kubernetes.container_name:"fluent-bit"
   ```

   > Tu devrais voir les logs collectés par Fluent Bit.

2. **Metrics Kubernetes :**
   Dans **Grafana** → Explore ou Dashboards → datasource Prometheus → vérifie les métriques CPU, mémoire, pods, nodes, etc.

3. **Fluent Bit metrics :**
   Dans **Prometheus** → Status → Targets → tu dois voir les endpoints Fluent Bit en `UP`.
   Prometheus scrappe automatiquement Fluent Bit via le `ServiceMonitor`.

---

## 7. Mise à jour de la stack

* **Kustomize (workloads, logging) :**

```bash
kubectl apply -k ./k8s/overlays/aws
kubectl apply -k ./k8s/overlays/aws/logging
```

* **Helm (kube-prometheus-stack) :**
  Si tu modifies `kube-prometheus-values.yaml` (ex: rétention, LoadBalancer, datasources) :

```bash
helm upgrade kube-prometheus-stack prometheus-community/kube-prometheus-stack \
  -n monitoring \
  -f ./k8s/overlays/aws/monitoring/kube-prometheus-values.yaml
```

> Helm upgrade met à jour la stack sans downtime significatif.

---

## 8. Supprimer toute la stack

Pour nettoyer entièrement le cluster et les ressources :

```bash
helm uninstall kube-prometheus-stack -n monitoring

kubectl delete -k ./k8s/overlays/aws/logging
kubectl delete -k ./k8s/overlays/aws

kubectl delete namespace monitoring logging
```

Supprimer le cluster EKS :

```bash
eksctl delete cluster --name microservices-demo-cluster --region us-east-1
```

> Toutes les ressources AWS (VPC, LoadBalancers, IAM, etc.) créées par EKS seront également supprimées.

