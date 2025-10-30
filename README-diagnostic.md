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
NAME                                  READY   STATUS    RESTARTS   AGE    IP            NODE       NOMINATED NODE   READINESS GATES
api-gateway-cf5578c8c-7rrrl           1/1     Running   0          104m   10.244.0.31   minikube   <none>           <none>
position-simulator-64f5cd6557-hls9g   1/1     Running   0          114m   10.244.0.24   minikube   <none>           <none>
position-tracker-7db5c5bd6d-xpksm     1/1     Running   0          114m   10.244.0.25   minikube   <none>           <none>
queue-5c7df485c6-kpnsl                1/1     Running   0          114m   10.244.0.23   minikube   <none>           <none>
webapp-68b5dfbdc7-q6ftk               1/1     Running   0          106m   10.244.0.30   minikube   <none>           <none>

kubectl get svc -o wide
NAME                        TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)                          AGE    SELECTOR
fleetman-api-gateway        NodePort    10.98.8.110      <none>        8080:30020/TCP                   114m   app=api-gateway
fleetman-position-tracker   ClusterIP   10.106.24.181    <none>        8080/TCP                         114m   app=position-tracker
fleetman-queue              NodePort    10.111.188.189   <none>        8161:30010/TCP,61616:31843/TCP   114m   app=queue
fleetman-webapp             NodePort    10.106.220.175   <none>        80:30080/TCP                     114m   app=webapp
kubernetes                  ClusterIP   10.96.0.1        <none>        443/TCP                          129m   <none>
```

Tu dois voir :

```
fleetman-queue
fleetman-position-tracker
fleetman-api-gateway
fleetman-webapp
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

### Test 1 — Position Simulator → ActiveMQ

Le simulateur publie vers la queue (`fleetman-queue:61616`).

```bash
nc -zv fleetman-queue 61616
```
Attendu :
```
Connection to fleetman-queue 61616 port [tcp/*] succeeded!
```

### Test 2 — Position Tracker → ActiveMQ

Le tracker consomme aussi les messages :
```bash
nc -zv fleetman-queue 61616
```

### Test 3 — API Gateway → Position Tracker

Le gateway appelle le tracker sur port `8080` :
```bash
curl -v fleetman-position-tracker:8080/vehicles/City%20Truck
```
Attendu : réponse HTTP 200 / contenu JSON.

### Test 4 — WebApp → API Gateway

```bash
curl -v fleetman-api-gateway:8080
```
Attendu : HTTP 200 / contenu JSON.

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

