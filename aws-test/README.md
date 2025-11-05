# Tester la création d'un cluster EKS + deploy nginx
Avant de déployer notre architecture principale sur AWS, j'ai réalisé une petite application Nginx de test.\
Si tout ça se passe bien pour vous ici, vous pouvez passer au déploiement de l'application principale.

* [cluster.yaml](cluster.yaml)
* [nginx-deployment.yaml](nginx-deployment.yaml)

## 1. Prérequis

### Compte AWS

Tu dois disposer de :
* D’un compte AWS actif
* De droits IAM suffisants (création de cluster EKS, VPC, EC2…)

### Sur ton poste local :
* Installer :
    * `awscli`
    * `eksctl`
    * `kubectl`

#### Installer AWS CLI

```bash
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install
aws --version
```

#### Installer `eksctl`

```bash
curl -sLO "https://github.com/eksctl-io/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz"
tar -xzf eksctl_$(uname -s)_amd64.tar.gz -C /tmp
sudo mv /tmp/eksctl /usr/local/bin
eksctl version
```

---

## 2. Créer un utilisateur IAM sécurisé (Access Key / Secret Key)

### Étape 1 : Créer un nouvel utilisateur IAM avec tous les droits EKS

> Console → **IAM** → **Users** → **Create user**

* Nom de l’utilisateur : `eks-user`

### Étape 2 : Donner les permissions
Crée un **groupe IAM “eks-user-group”** et attache-lui ce policies :
```
IAMFullAccess
```

Puis ajoute ton utilisateur `eks-user` à ce groupe.

### Étape 3 : Récupérer les clés

À la fin de la création, dans les informations de l'utilisateur `eks-user`, crée une clé d'accès :

* Télécharge le fichier CSV
* Ou note :
    * **Access key ID** (ex: `AKIA...`)
    * **Secret access key** (ex: `abcd...`)

=> Tu **ne pourras plus revoir la clé secrète** plus tard, garde-la bien.

### Étape 4 : Configurer AWS CLI

```bash
aws configure
```

Entre tes clés :

```
AWS Access Key ID [None]: AKIA...
AWS Secret Access Key [None]: xxxxxxx
Default region name [None]: us-east-1
Default output format [None]: json
```

Vérifie :

```bash
aws sts get-caller-identity
```

Tu dois voir ton utilisateur IAM :

```json
{
  "UserId": "AIDAEXAMPLE123",
  "Account": "123456789012",
  "Arn": "arn:aws:iam::123456789012:user/eks-user"
}
```

### Étape 5 : Attacher les policies AWS managées à notre groupe IAM "eks-user-group"

#### 1 — Attacher la policy **AmazonEKSClusterPolicy**
```bash
aws iam attach-group-policy \
  --group-name eks-user-group \
  --policy-arn arn:aws:iam::aws:policy/AmazonEKSClusterPolicy
```

#### 2 — Attacher la policy **AmazonEKSServicePolicy**
```bash
aws iam attach-group-policy \
  --group-name eks-user-group \
  --policy-arn arn:aws:iam::aws:policy/AmazonEKSServicePolicy
```

#### 3 — Attacher la policy **AmazonEKSWorkerNodePolicy**
```bash
aws iam attach-group-policy \
  --group-name eks-user-group \
  --policy-arn arn:aws:iam::aws:policy/AmazonEKSWorkerNodePolicy
```

#### 4 — Attacher la policy **AmazonEC2FullAccess**
```bash
aws iam attach-group-policy \
  --group-name eks-user-group \
  --policy-arn arn:aws:iam::aws:policy/AmazonEC2FullAccess
```

#### 5 — Attacher la policy **AmazonVPCFullAccess**
```bash
aws iam attach-group-policy \
  --group-name eks-user-group \
  --policy-arn arn:aws:iam::aws:policy/AmazonVPCFullAccess
```

#### 6 — Attacher la policy **AWSCloudFormationFullAccess**
```bash
aws iam attach-group-policy \
  --group-name eks-user-group \
  --policy-arn arn:aws:iam::aws:policy/AWSCloudFormationFullAccess
```

#### 7 — Vérifie que tout est bien attaché
```bash
aws iam list-attached-group-policies --group-name eks-user-group --output table
------------------------------------------------------------------------------------------
|                                ListAttachedGroupPolicies                               |
+----------------------------------------------------------------------------------------+
||                                   AttachedPolicies                                   ||
|+------------------------------------------------------+-------------------------------+|
||                       PolicyArn                      |          PolicyName           ||
|+------------------------------------------------------+-------------------------------+|
||  arn:aws:iam::aws:policy/AmazonEC2FullAccess         |  AmazonEC2FullAccess          ||
||  arn:aws:iam::aws:policy/IAMFullAccess               |  IAMFullAccess                ||
||  arn:aws:iam::aws:policy/AmazonEKSClusterPolicy      |  AmazonEKSClusterPolicy       ||
||  arn:aws:iam::aws:policy/AmazonVPCFullAccess         |  AmazonVPCFullAccess          ||
||  arn:aws:iam::aws:policy/AmazonEKSWorkerNodePolicy   |  AmazonEKSWorkerNodePolicy    ||
||  arn:aws:iam::aws:policy/AmazonEKSServicePolicy      |  AmazonEKSServicePolicy       ||
||  arn:aws:iam::aws:policy/AWSCloudFormationFullAccess |  AWSCloudFormationFullAccess  ||
|+------------------------------------------------------+-------------------------------+|

```

#### 8 — Ajouter la policy "AllowEksFullAccess" à notre groupe

Cette policy accorde à ton groupe (et donc à ton utilisateur `eks-user`) toutes les permissions EKS.
C’est équivalent à `AmazonEKSFullAccess`, qui n’existe pas comme policy managée mais que nous recréons ici.

```bash
$ aws iam put-group-policy \
  --group-name eks-user-group \
  --policy-name AllowEksFullAccess \
  --policy-document '{
    "Version": "2012-10-17",
    "Statement": [
      {
        "Effect": "Allow",
        "Action": [
          "eks:*"
        ],
        "Resource": "*"
      }
    ]
  }'
```

Vérifie que la policy a bien été ajoutée :

```bash
aws iam list-group-policies --group-name eks-user-group
```

Résultat attendu :

```
{
    "PolicyNames": [
        "AllowEksFullAccess"
    ]
}
```

Teste l’accès EKS :

```bash
aws eks describe-addon-versions
```

Si tout est bon, la commande doit retourner une longue liste d’add-ons EKS (versions de vpc-cni, kube-proxy, coredns, etc.).

---

## 3. Créer le cluster EKS

Crée un fichier `cluster.yaml` :

```yaml
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig

metadata:
  name: demo-cluster
  region: us-east-1
  version: "1.34"

# Désactivation de l'Auto Mode (important avec eksctl >= 0.215)
autoModeConfig:
  enabled: false

vpc:
  cidr: "10.0.0.0/16"

managedNodeGroups:
  - name: ng-1
    instanceType: t3.small     # plus économique que t3.medium, suffisant pour 4 microservices
    desiredCapacity: 2
    minSize: 1
    maxSize: 2
    volumeSize: 20             # taille du disque en Go
    ssh:
      allow: true
      publicKeyName: <your-ssh-keypair-name>
    iam:
      withAddonPolicies:
        autoScaler: true       # pour utiliser le Cluster Autoscaler
        cloudWatch: true       # pour les métriques/logs
        albIngress: true       # si tu veux un Ingress Controller plus tard
    tags:
      environment: dev
      project: microservices-demo
```

### À propos du `publicKeyName`

Dans la section :

```yaml
ssh:
  allow: true
  publicKeyName: <your-ssh-keypair-name>
```

Cela indique à `eksctl` :

> “Associe cette clé SSH à mes instances EC2 (les worker nodes) pour que je puisse m’y connecter si besoin.”

#### Concrètement :

* Les worker nodes sont des EC2 managés par EKS.
* Si `allow: true`, `eksctl` ajoute la clé publique AWS EC2 Key Pair à ces instances.
* Tu pourras ensuite te connecter en SSH :

```bash
ssh -i my-key.pem ec2-user@<public-ip>
```

#### Comment obtenir `publicKeyName` ?

Tu dois avoir un **Key Pair** déjà existant dans ta région AWS (`us-east-1` ici).

Pour le vérifier :

```bash
aws ec2 describe-key-pairs --region us-east-1
```

Tu verras une sortie du type :

```json
{
  "KeyPairs": [
    {
      "KeyName": "my-ec2-key",
      "KeyPairId": "key-0123456789abcdef0",
      "KeyType": "rsa"
    }
  ]
}
```
Ici `KeyName` = `my-ec2-key`\
=> donc dans ton YAML tu mets :
```yaml
publicKeyName: my-ec2-key
```

#### Si tu n’as pas encore de clé :
Crée-la avec :
```bash
aws ec2 create-key-pair --key-name my-ec2-key --query "KeyMaterial" --output text > my-ec2-key.pem
chmod 400 my-ec2-key.pem
```
Garde le fichier `.pem` précieusement :
c’est ta clé privée pour te connecter en SSH.

### Lancer la création

```bash
eksctl create cluster -f cluster.yaml
```
Durée : 15–20 minutes
`eksctl` va :

* Créer un **VPC**
* Créer le **control plane EKS**
* Lancer les **nœuds EC2**
* Mettre à jour ton fichier `~/.kube/config`

---

## 4. Vérifier la connexion Kubernetes

Quand c’est fini :
```bash
kubectl get nodes
```
Résultat attendu :

```
NAME                                           STATUS   ROLES    AGE   VERSION
ip-10-0-1-45.us-east-1.compute.internal       Ready    <none>   2m    v1.30.x
ip-10-0-2-12.us-east-1.compute.internal       Ready    <none>   2m    v1.30.x
```

---

## 5. Déployer un manifest simple

Crée un fichier `nginx-deployment.yaml` :

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:latest
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  type: LoadBalancer
  ports:
  - port: 80
    targetPort: 80
  selector:
    app: nginx
```

Applique :
```bash
kubectl apply -f nginx-deployment.yaml
```

Vérifie :
```bash
kubectl get all
kubectl get svc nginx-service
```

Quand le `EXTERNAL-IP` apparaît :
```
nginx-service   LoadBalancer   a1b2c3d4.us-east-1.elb.amazonaws.com   80:31234/TCP   2m
```
Ouvre cette URL dans ton navigateur → tu verras la page Nginx.

---

## 6. Activer les logs du cluster

Ajouter la policy managée pour CloudWatchLogs :

```bash
aws iam attach-user-policy \
  --user-name eks-user \
  --policy-arn arn:aws:iam::aws:policy/CloudWatchLogsFullAccess
```

Vérification des permissions :

```bash
aws iam list-attached-user-policies --user-name eks-user
```

Activer les logs :

```bash
eksctl utils update-cluster-logging \
  --enable-types all \
  --region us-east-1 \
  --cluster demo-cluster \
  --approve
```

Vérification :

```bash
aws eks describe-cluster \
  --name demo-cluster \
  --region us-east-1 \
  --query "cluster.logging.clusterLogging" \
  --output table
```

Voir les logs :

```bash
aws logs tail /aws/eks/demo-cluster/cluster --follow
```

---

## 7. Supprimer le cluster

Quand tu as fini :
```bash
eksctl delete cluster --name demo-cluster --region us-east-1
```

## 8. vérification

#### 1. Vérifier les stacks CloudFormation

C’est **le plus fiable** — `eksctl` crée toujours des stacks CloudFormation pour gérer ton cluster et tes nodes.

```bash
aws cloudformation list-stacks --region us-east-1 \
  --query "StackSummaries[?starts_with(StackName, 'eksctl-demo-cluster') && StackStatus!='DELETE_COMPLETE'].[StackName,StackStatus]" \
  --output table
```

Si tout est bien supprimé, cette commande ne doit **rien retourner** (ou bien toutes les stacks ont `DELETE_COMPLETE`).

#### 2. Vérifier le cluster EKS

```bash
aws eks list-clusters --region us-east-1
```

Ton cluster `demo-cluster` **ne doit plus apparaître**.

#### 3. Vérifier le VPC associé

`eksctl` crée un VPC du style `eksctl-demo-cluster-cluster/VPC`
Tu peux lister les VPC encore présents :

```bash
aws ec2 describe-vpcs --region us-east-1 \
  --query "Vpcs[?Tags[?Value=='eksctl-demo-cluster-cluster']].VpcId" \
  --output table
```
Si le résultat est vide, le VPC a bien été supprimé.

#### 4. Vérifier les groupes de sécurité restants (au cas où)

```bash
aws ec2 describe-security-groups --region us-east-1 \
  --query "SecurityGroups[?GroupName=='eksctl-demo-cluster-cluster'].GroupId" \
  --output table
```

#### 5. Vérifier les rôles IAM générés par EKSCTL

```bash
aws iam list-roles --query "Roles[?contains(RoleName, 'eksctl-demo-cluster')].RoleName" --output table
```
Si aucun rôle ne ressort, tout a été nettoyé.

#### 6. Vérifier les ELB (load balancers) éventuellement restants

```bash
aws elb describe-load-balancers --region us-east-1 \
  --query "LoadBalancerDescriptions[?contains(DNSName, 'demo-cluster')].LoadBalancerName" --output table
```

(et pour les nouveaux ALB/NLB — version v2 :)

```bash
aws elbv2 describe-load-balancers --region us-east-1 \
  --query "LoadBalancers[?contains(LoadBalancerName, 'demo-cluster')].LoadBalancerName" --output table
```

Tu es 100 % propre si :

* `aws eks list-clusters` → vide
* `aws cloudformation list-stacks` → plus de stack `eksctl-...`
* `aws ec2 describe-vpcs` → VPC supprimé
* `aws iam list-roles` → aucun rôle `eksctl-demo-cluster`

