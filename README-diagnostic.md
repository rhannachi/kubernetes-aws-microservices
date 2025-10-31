# Objectif global

Vérifier que :

1. Tous les pods peuvent se **résoudre par DNS** via leur `Service`.
2. Tous les pods peuvent **communiquer sur les bons ports**.
3. Les flux principaux (simulator → queue → tracker → api-gateway → webapp) **fonctionnent réellement**.
4. Les services de type `NodePort` sont **accessibles depuis l’extérieur**.

---

# Étape 1 — Identifier les pods et services

```bash
kubectl get pods -o wide
NAME                                  READY   STATUS    RESTARTS   AGE     IP            NODE       NOMINATED NODE   READINESS GATES
api-gateway-69f87647b4-l48v6          1/1     Running   0          7m45s   10.244.0.12   minikube   <none>           <none>
mongodb-84d458c5dc-gwnsv              1/1     Running   0          7m45s   10.244.0.17   minikube   <none>           <none>
position-simulator-764c5b5c4b-qdj78   1/1     Running   0          6m59s   10.244.0.18   minikube   <none>           <none>
position-tracker-5c8db98b74-d8b88     1/1     Running   0          5m26s   10.244.0.19   minikube   <none>           <none>
queue-98f6cb787-2rdw7                 1/1     Running   0          7m45s   10.244.0.15   minikube   <none>           <none>
webapp-85f9ff9b5b-7m2l7               1/1     Running   0          7m45s   10.244.0.16   minikube   <none>           <none>

kubectl get svc -o wide
NAME                        TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)                          AGE     SELECTOR
fleetman-api-gateway        NodePort    10.104.60.11     <none>        8080:30020/TCP                   8m24s   app=api-gateway
fleetman-mongodb            ClusterIP   10.97.215.160    <none>        27017/TCP                        8m24s   app=mongodb
fleetman-position-tracker   ClusterIP   10.109.51.157    <none>        8080/TCP                         8m24s   app=position-tracker
fleetman-queue              NodePort    10.96.199.66     <none>        8161:30010/TCP,61616:31103/TCP   8m24s   app=queue
fleetman-webapp             NodePort    10.102.129.144   <none>        80:30080/TCP                     8m24s   app=webapp
kubernetes                  ClusterIP   10.96.0.1        <none>        443/TCP                          63m     <none>
```

Tu dois voir :

```
fleetman-queue
fleetman-position-tracker
fleetman-api-gateway
fleetman-webapp
fleetman-mongodb
```

Les adresses `CLUSTER-IP` seront utilisées dans les tests DNS/port.

---

# Étape 2 — Tester la **résolution DNS interne** (CoreDNS)

Exécute un shell dans un pod, par exemple dans le `api-gateway` :

```bash
kubectl exec -it deploy/api-gateway -- sh
```

Puis teste la résolution :

```bash
# Test DNS interne pour chaque service
nslookup fleetman-queue
nslookup fleetman-position-tracker
nslookup fleetman-api-gateway
nslookup fleetman-webapp
nslookup fleetman-mongodb
```
**Résultat attendu :**
Chaque service doit résoudre vers un `CLUSTER-IP` interne (ex: `10.96.x.x`).

---

# Étape 3 — Vérifier la **connectivité TCP** entre les microservices

On utilise `curl` (pour HTTP) et `nc` (netcat) pour les ports non HTTP.
Si ton image ne contient pas `curl` ou `nc`, tu peux installer temporairement un pod de debug :

```bash
kubectl run -it network-test --image=alpine:3.18 -- sh
apk add curl netcat-openbsd
```

### Test 1 — Position Simulator/Tracker → ActiveMQ

Le simulateur publie vers la queue (`fleetman-queue:61616`).

```bash
nc -zv fleetman-queue 61616
```
Attendu :
```
Connection to fleetman-queue 61616 port [tcp/*] succeeded!
```

### Test 2 — API Gateway → Position Tracker

Le gateway appelle le tracker sur port `8080` :
```bash
curl -v fleetman-position-tracker:8080/vehicles/City%20Truck
```
Attendu : réponse HTTP 200 / contenu JSON.

### Test 3 — WebApp → API Gateway

```bash
curl -v fleetman-api-gateway:8080
```
Attendu : HTTP 200 / contenu JSON.

### Test 4 — Position Tracker → mongodb

```bash
nc -zv fleetman-mongodb 27017
```
Attendu :
```
Connection to fleetman-queue 61616 port [tcp/*] succeeded!
```

---

# Étape 4 — Vérifier la communication externe (NodePort)

Teste depuis ta machine (ou un `curl` hors cluster) :

```bash
# WebApp
curl http://<NODE_IP>:30080
# API Gateway
curl http://<NODE_IP>:30020/api/vehicles # => 404 (TODO)
```

Si le cluster est minikube :
```bash
minikube service fleetman-webapp --url
minikube service fleetman-api-gateway --url
```

Tu devrais obtenir des URLs accessibles depuis ton navigateur.

---

# Étape 5 — (Optionnel) Pod “network-debug” permanent

Ajoute un pod utilitaire pour refaire les tests à tout moment :

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: network-debug
spec:
  containers:
  - name: net-debug
    image: nicolaka/netshoot
    command: ["sleep", "infinity"]
```

```bash
kubectl exec -it network-debug -- bash
```
Tu auras accès à `dig`, `curl`, `ping`, `nc`, `tcpdump`, etc.

---

# Résumé des flux à tester

| Source Pod         | Destination Service       | Port  | Protocole | Commande de test                                      |
| ------------------ | ------------------------- | ----- | --------- | ----------------------------------------------------- |
| position-simulator | fleetman-queue            | 61616 | TCP       | `nc -zv fleetman-queue 61616`                         |
| position-tracker   | fleetman-queue            | 61616 | TCP       | `nc -zv fleetman-queue 61616`                         |
| api-gateway        | fleetman-position-tracker | 8080  | HTTP      | `curl fleetman-position-tracker:8080/actuator/health` |
| webapp             | fleetman-api-gateway      | 8080  | HTTP      | `curl fleetman-api-gateway:8080`                      |
| external (host)    | fleetman-webapp           | 30080 | HTTP      | `curl http://<nodeip>:30080`                          |

