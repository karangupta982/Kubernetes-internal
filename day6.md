- service is an another layer above deployments so that app could be available to user.
- If we try to pull image from any repository then we must specify tag, in case if tag is not specified then image would get downloaded with tag 'latest' and if there is not image with latest then you would get error.
- internal vs external ips: ## Internal vs. External IP Addresses
The difference between internal and external IP addresses comes down to scope and visibility:

* Internal IP Address (Private IP): This is an address used inside a private network (like your home Wi-Fi, an office building, or a Kubernetes cluster node network). These addresses are hidden from the outside world. Devices inside the network use them to talk to each other. They cannot be reached directly from the internet.
* External IP Address (Public IP): This is a globally unique address assigned to a device connected to the public internet (like a router, a cloud load balancer, or a public website). Anyone on the internet can attempt to route traffic to an external IP address.

------------------------------
## What is RFC 1918?
RFC 1918 (Request for Comments 1918) is the official global internet standard that reserves specific blocks of IPv4 addresses exclusively for private (internal) networks.
Before this standard was created in 1996, every single connected device needed a unique public IP address. As the internet grew rapidly, the world began running out of IPv4 addresses. RFC 1918 solved this by setting aside three distinct blocks of addresses that internet routers are strictly forbidden from routing publicly.
## The Three RFC 1918 Private Ranges
If you check your computer's IP address, your home router, or your Kubernetes node IPs, they will almost always fall into one of these three blocks:

| Block Range | Total IP Addresses | Common Use Case |
|---|---|---|
| 10.0.0.0 to 10.255.255.255 | ~16.7 Million (10.0.0.0/8) | Large enterprise networks, Cloud VPCs (AWS/GCP), and Kubernetes Pod/Service networks. |
| 172.16.0.0 to 172.31.255.255 | ~1.04 Million (172.16.0.0/12) | Medium-sized corporate networks, Docker default bridge networks, and local kind nodes. |
| 192.168.0.0 to 192.168.255.255 | ~65,536 (192.168.0.0/16) | Home Wi-Fi routers and small office networks (e.g., 192.168.1.1). |

------------------------------
## How they work together: The Analogy 🏢
Think of a large office building:

* Internal IPs (RFC 1918): These are like the internal desk extension numbers (e.g., Extension 402). Anyone inside the building can dial Extension 402 to talk to you. However, a person standing out on the street cannot pick up their mobile phone and dial "402" to reach your desk.
* External IP: This is the main corporate phone number of the building printed on the billboard outside.

To bridge the gap, routers use a technology called NAT (Network Address Translation). When your laptop (with a private IP like 192.168.1.5) requests a webpage, your home router swaps your private IP out for its own single public External IP, fetches the data from the internet, and hands it back to your laptop.

- Types of Services: ClusterIp, Load balancer, Node port, Headless, External Name
  
- ClusterIP (The Default): This exposes the Service on an internal IP address within the cluster. It makes the Service only reachable from inside the cluster.
  Best for: Internal backend microservices, databases, or caching layers that should never be exposed directly to users.
  How it works: Pod A talks to Pod B using the stable cluster-internal IP or DNS name.

- Nodeport: This opens a specific, identical port on every single Node (machine) in your cluster and forwards traffic arriving at that port to your Service. Every Kubernetes cluster is made up of one or more physical or virtual machines called Nodes.When you create a NodePort service, Kubernetes reserves a specific port number from a dedicated range (30000 to 32767) and opens that exact same port on every single Node in your cluster.Any network traffic that hits any node's IP address on that specific port is automatically caught and forwarded directly into your Pods.
  A user must be in the "same network" to access a NodePort service, it means the user's device must have direct network routing to the internal IP addresses of your Kubernetes cluster nodes. If there is a firewall, router, or public internet boundary separating the user from the nodes, the connection will fail.
  [ Public Internet User ] 
       │ 
       ▼ 
┌───────────────────────────────────────┐
│  VPC / Company Firewall Boundary      │
│                                       │
│  [ Internal VM ] ───(Can Access!)──┐  │
│                                    ▼  │
│                       [ K8s Node IP: 10.0.0.5 ]
│                          └── Port: 32123 (NodePort)
└───────────────────────────────────────┘

  Best for: Quick development testing, or when you want to route external traffic to your cluster without paying for a cloud provider's load balancer.
  How it works: Kubernetes allocates a random port in the default range 30000–32767. You can access your application by hitting http://<Any-Node-IP>:<NodePort>.
  apiVersion: v1
kind: Service
metadata:
  name: frontend-nodeport
spec:
  type: NodePort                # 🎯 Tells K8s to open a port on all nodes
  selector:
    app: webserver              # Finds your Pods with this label
  ports:
    - protocol: TCP
      port: 80                  # The port used INSIDE the cluster (ClusterIP)
      targetPort: 80            # The port your application is listening on inside the container
      nodePort: 31050           # 🚪 Optional: The exact external port to open (30000-32767)
      
[ Your Laptop Browser ]
         │
         ├───► Hits http://172.18.0.2:31050 (Node 1) ──┐
         │                                             ▼
         └───► Hits http://172.18.0.3:31050 (Node 2) ──┼──► [ Kubernetes Routing ]
                                                       │            │
                                                       │            ▼
                                                       └────► [ Your Pod ] (Port 80)


  
- Load Balancer: This integrates directly with a cloud provider's infrastructure (like AWS, GCP, or Azure) to automatically spin up a physical, external load balancer machine.
  When you set a service type to LoadBalancer, Kubernetes automatically reaches out to your cloud provider's API and rents a physical, hardware network router outside of your cluster.
  Best for: Exposing production-grade web applications or public APIs directly to the internet.
  How it works: nce applied in a cloud environment, run kubectl get svc. You will see:PORT: 80EXTERNAL-IP: 203.0.113.45 (A real, public external IP address)Now, a user sitting at home can type http://203.0.113.45 into their web browser. The traffic hits the cloud's external router, which safely tunnels the request straight into your cluster's private Pod network. The cloud provider provisions a dedicated external public IP address. It automatically routes internet traffic straight down into your cluster's NodePorts and Pods. O
  Note for kind: Because kind runs locally inside Docker, a LoadBalancer service will sit in a permanent <pending> state unless you install a local tool like MetalLB to simulate a cloud provider.

- External Name: This acts as a simple internal alias/shortcut that points your internal pods to an external, real-world domain name outside the cluster. an ExternalName service does not use selectors and does not point to Pods. Instead, it acts as an internal redirect or alias to an external website or third-party server outside your cluster.
  Best for: When code running inside your cluster needs to securely talk to an external third-party API or database (like an AWS RDS database or Twilio API).
  How it works: Instead of hardcoding ://amazonaws.com in your app's configuration, you map it to a local service name. It handles the redirection at the DNS level without proxying any network traffic.
  Suppose your backend Pod needs to talk to a managed database hosted outside your cluster on AWS RDS, with a long address like ://amazonaws.com.Instead of hardcoding that long address inside your application's source code, you map it to a short internal service name like local-db. If the database address changes in the future, you only update this one YAML file—you never have to rebuild or recompile your app code.
  apiVersion: v1
kind: Service
metadata:
  name: local-db                # 📦 The short internal nickname
spec:
  type: ExternalName            # 🎯 Tells K8s this points outside the cluster
  externalName: ://amazonaws.com # 🌍 The real external address
  Inside your container application, you tell your database client configuration to connect to http://local-db.Kubernetes network components immediately intercept that request and substitute the real destination, making the connection seamless.

- Headless Service (The Custom Routing Service): A standard Service acts as a single proxy. It gets an internal cluster IP, and traffic hits that IP before being distributed to random Pods.
  A Headless Service is a service where you explicitly set clusterIP: None. It has no single IP address. Instead of routing traffic, it simply returns the exact, direct internal IP addresses of all matching Pods simultaneously. 
  When do you use it?You use it for stateful applications like databases (MongoDB, PostgreSQL, or Kafka). In a database cluster, you cannot talk to just a random Pod; you need to write data specifically to the Primary/Master node and read data from the Secondary/Replica nodes. Your application needs to know exactly who is who.
apiVersion: v1
kind: Service
metadata:
  name: database-headless
spec:
  clusterIP: None               # 🎯 THIS makes it a Headless Service!
  selector:
    app: mysql
  ports:
    - port: 3306

 How it behaves in action: When a backend application looks up database-headless in the cluster's network, instead of getting one service IP, the internal DNS returns a list:
 10.244.0.5 (MySQL-0 / Master)
 10.244.1.9 (MySQL-1 / Replica)
  The application can now choose to connect directly to the Master node to perform database queries.


------

## Endpoints and EndpointSlices

### Pehle problem samjho, phir solution

Service ka kaam hai ek **stable IP** provide karna jo traffic ko **actual Pods** tak pahunchaye. Lekin Service ko kaise pata chalta hai ki "abhi is waqt konse Pods healthy hain aur traffic receive kar sakte hain?"

Yahi kaam karta hai **Endpoints object**.

---

## 1. Endpoints — kya hai, exactly

```
DEFINITION: Endpoints ek Kubernetes object hai jo ek Service ke
liye CURRENT, HEALTHY, READY Pod IPs aur ports ki LIST maintain
karta hai.

Har Service ke saath AUTOMATICALLY ek same-naam ka Endpoints object
banta hai.
```

### Concrete example

```yaml
# Tumhara Service:
apiVersion: v1
kind: Service
metadata:
  name: db-service
  namespace: ns-dev
spec:
  selector:
    app: postgres
  ports:
  - port: 5432
    targetPort: 5432
```

Jaise hi ye Service apply hui, Kubernetes automatically ek Endpoints
object banata hai:

```yaml
apiVersion: v1
kind: Endpoints
metadata:
  name: db-service        # ← Service ke SAME naam ka, automatically
  namespace: ns-dev
subsets:
- addresses:
  - ip: 10.244.1.3        # Pod 1 ka IP (app=postgres label wala, READY)
  - ip: 10.244.1.7        # Pod 2 ka IP (app=postgres label wala, READY)
  - ip: 10.244.2.2        # Pod 3 ka IP (app=postgres label wala, READY)
  notReadyAddresses:
  - ip: 10.244.1.9        # Pod 4 — label match karta hai lekin
                          # readinessProbe FAIL hai, isliye READY nahi
  ports:
  - port: 5432
```

```
CRITICAL POINT: notReadyAddresses mein jo pods hain, unhe
kube-proxy TRAFFIC NAHI BHEJTA — ye Endpoints object ka sabse
important role hai. Service sirf "addresses" wale pods ko traffic
deta hai, "notReadyAddresses" wale ko nahi.
```

### Who manages this Endpoints object?

```
CONTROLLER: "Endpoints Controller" (kube-controller-manager ka hissa —
  Lecture 2 yaad karo)

YE CONTROLLER KYA KARTA HAI:
1. Har Service ko watch karta hai
2. Har Service ke selector ke according matching Pods dhundhta hai
3. Un Pods ka readinessProbe status check karta hai
4. Ready Pods ki IPs → "addresses" mein dalta hai
5. NotReady Pods ki IPs → "notReadyAddresses" mein dalta hai
6. Pod add/delete/crash/recover hone par AUTOMATICALLY update karta hai

ISLIYE: Jab tum ek Pod delete karte ho, kuch milliseconds mein wo
Endpoints se hat jata hai → traffic us dead pod ko nahi milta.
```

### Verify karna

```bash
kubectl get endpoints                      # saare endpoints
kubectl get endpoints db-service           # specific service ka
kubectl describe endpoints db-service      # detail mein
kubectl get endpoints db-service -o yaml   # full YAML
```

### Manually endpoints check karo — real debugging use case

```bash
# Agar Service kaam nahi kar raha, PEHLE ye check karo:
kubectl get endpoints <service-name>

# Output mein:
NAME         ENDPOINTS                                AGE
db-service   10.244.1.3:5432,10.244.1.7:5432         5m
             ← Agar ye "<none>" dikhe, koi READY pod nahi hai
               Service ke selector se match karne wala

# "<none>" ke common reasons:
# 1. Label mismatch — Service selector aur Pod labels alag hain
# 2. Pod exists but readinessProbe fail ho raha hai
# 3. Koi pod hi exist nahi karta us selector ke saath
# 4. Pod wrong namespace mein hai
```

---

## 2. Problem with Endpoints — scale par kya hota hai

```
Maan lo tumhara Service 1000 Pods ko serve kar raha hai.

Endpoints object:
subsets:
- addresses:
  - ip: 10.244.0.1
  - ip: 10.244.0.2
  - ip: 10.244.0.3
  ... (997 more entries)

Ab 1 Pod crash hua. Kya hoga?

Endpoints controller POORA Endpoints object UPDATE karta hai —
ek IP nikali aur updated object API server ko bheja. API server
ne updated object ETcd mein likha. Phir har node par kube-proxy
ne is updated object ko DEKHA aur apne iptables rules update kiye.

YE HAR NODE PAR HOTA HAI.

1000 nodes wale cluster mein, ek Pod change hone par:
- 1 full Endpoints object (1000 IPs wala) → 1000 nodes par
  distribute kiya gaya
- Har node ne poora object receive kiya aur process kiya
- NETWORK BANDWIDTH WASTE: 1 IP change ke liye poora 1000-IP
  object bheja gaya

Is problem ko kehte hain: "THUNDERING HERD PROBLEM" ya
"O(n²) SCALING PROBLEM" jahan n = pods ka count.

10,000 pods wale cluster mein ek single pod change:
= 10,000 nodes × 10,000 IP list = 10^8 operations
= CLUSTER MELTDOWN
```

---

## 3. EndpointSlice — solution

```
KUBERNETES 1.19 mein STABLE hua (ab default hai).

CORE IDEA: Ek bada Endpoints object banane ke bajaye, Pods ko
CHHOTE CHHOTE GROUPS mein divide karo — har group ek "slice" hai.

DEFAULT SIZE: Har EndpointSlice mein maximum 100 endpoints.

Same 1000 pods wala example:
1000 pods ÷ 100 per slice = 10 EndpointSlices
```

### EndpointSlice YAML example

```yaml
apiVersion: discovery.k8s.io/v1
kind: EndpointSlice
metadata:
  name: db-service-xv93k      # auto-generated naam (Service naam + random)
  namespace: ns-dev
  labels:
    kubernetes.io/service-name: db-service  # ← kis Service ka hai
addressType: IPv4
ports:
- name: ''
  port: 5432
  protocol: TCP
endpoints:
- addresses:
  - 10.244.1.3                 # Pod 1
  conditions:
    ready: true                # readinessProbe pass
    serving: true
    terminating: false
- addresses:
  - 10.244.1.7                 # Pod 2
  conditions:
    ready: true
    serving: true
    terminating: false
- addresses:
  - 10.244.1.9                 # Pod 3 — terminating ho raha hai
  conditions:
    ready: false
    serving: true              # abhi bhi traffic serve kar sakta hai
    terminating: true          # lekin delete ho raha hai
```

### Ab same problem ka solution:

```
1000 pods, 10 slices (100 pods each)
1 Pod crash hua — wo pod slice number 7 mein tha.

ENDPOINTS (purana tarika):
  Poora 1000-IP object update → 1000 nodes par bheja
  Data sent: 1000 nodes × 1000 entries = BAHUT ZYADA

ENDPOINTSLICE (naya tarika):
  Sirf SLICE 7 update hua (100 IPs wala object) → 1000 nodes par bheja
  Data sent: 1000 nodes × 100 entries = 10x CHHOTA
  
Jaise jaise cluster bada hota hai, difference aur bada hota jaata hai.
```

### Commands

```bash
kubectl get endpointslices                    # saare slices
kubectl get endpointslices -l kubernetes.io/service-name=db-service
kubectl describe endpointslice db-service-xv93k
```

---

## 4. EndpointSlice ke extra conditions — Endpoints se better

```
Endpoints sirf 2 states jaanta tha:
  addresses         → ready (traffic do)
  notReadyAddresses → not ready (traffic mat do)

EndpointSlice 3 conditions track karta hai:

ready:       → readinessProbe pass? Normal traffic ke liye.

serving:     → Kya pod ABHI traffic serve KAR SAKTA HAI?
               (ready=false hone ke bawajood serve=true ho sakta hai
               jab pod graceful shutdown mein hai — ongoing connections
               complete karne de)

terminating: → Pod ko delete kiya ja raha hai?
               (SIGTERM mil gaya lekin abhi terminationGracePeriod
               khatam nahi hua — in-flight requests finish karne de)

YE COMBINATION "connection draining" enable karta hai — deployment
update ke waqt already connected clients ke requests gracefully
complete hoti hain, nahi tode jaate beech mein.
```

---

## 5. Headless Service — endpoints ka ek special case

```
Normal Service → ek single stable ClusterIP deta hai, kube-proxy
  traffic distribute karta hai pods mein.

Headless Service → clusterIP: None, koi stable IP NAHI milti.
  DNS query par DIRECTLY Pod IPs return hoti hain (Endpoints/
  EndpointSlice se liye jaate hain).

```yaml
spec:
  clusterIP: None       # ← yahi headless banata hai
  selector:
    app: postgres
```

DNS query result:
  Normal Service:   db-service.ns-dev.svc.cluster.local → 10.96.0.15 (single IP)
  Headless Service: db-service.ns-dev.svc.cluster.local → 10.244.1.3,
                                                           10.244.1.7,
                                                           10.244.2.2 (all Pod IPs)

USE CASE: StatefulSets (jaise databases) jahan CLIENT khud decide
karna chahta hai ki konsa specific pod se baat kare (primary ya
replica) — load balancer ke through nahi.

StatefulSet individual pod DNS:
  postgres-0.db-service.ns-dev.svc.cluster.local → 10.244.1.3 (pod 0)
  postgres-1.db-service.ns-dev.svc.cluster.local → 10.244.1.7 (pod 1)
```

---

## 6. Interview answers

```
Q: "What is an Endpoint in Kubernetes?"

A: "An Endpoints object is automatically created by the Endpoints
controller for every Service with a selector. It maintains the live
list of ready Pod IPs and ports that back that Service — kube-proxy
reads this list to program iptables or IPVS rules on each node.
Pods that fail their readiness probe appear in notReadyAddresses and
receive no traffic, which is the actual mechanism behind readiness-
based traffic control."

Q: "What problem does EndpointSlice solve?"

A: "The original Endpoints object stores all backing pod IPs in a
single object. In large clusters — say 1000+ pods — a single pod
change forces the entire object to be rewritten and distributed to
every node in the cluster. EndpointSlice shards this into groups of
100 endpoints per slice, so a single pod change only triggers an
update to one small slice rather than the entire list. This changes
the update cost from O(n²) to roughly O(n), which is what made
Kubernetes scale to clusters with tens of thousands of pods."

Q: "Service kaam nahi kar raha, kaise debug karoge?"

A: "Pehle kubectl get endpoints <service-name> — agar output mein
<none> hai, Service ko koi ready pod nahi mil raha. Then check:
kubectl get pods -l <service-selector> to see if pods exist, then
kubectl describe pod <pod-name> to see if readinessProbe is failing.
Agar endpoints populated hain lekin traffic nahi ja raha, then
kube-proxy rules check karo — kubectl get endpoints -o yaml se
actual IPs verify karo aur us pod par kubectl exec se directly
test karo."
```

---

## Summary — ek line mein dono

```
Endpoints  = "Service ke peeche kaunse Pods ready hain" ki list
             (ek bada object, scale par slow)

EndpointSlice = Same kaam, lekin 100-pod ke chhote chunks mein
                (fast, efficient, production standard now)

kube-proxy dono ko read karta hai aur iptables/IPVS rules banata
hai — actual packet forwarding ka mechanism yahi hai.
```

---

Good question — and it has a precise answer, not a vague one. Let me explain both parts exactly.

---

## Part 1: Endpoints vs EndpointSlice — who controls what

```
Tumhara confusion valid hai: "agar dono automatically bante hain,
to Kubernetes kaise decide karta hai konsa use karna hai?"

SHORT ANSWER: Dono ABHI BHI bante hain by default. EndpointSlice
ne Endpoints ko REPLACE nahi kiya — kube-proxy ne sirf apna DEFAULT
behavior switch kar liya hai "main ENDPOINTSLICE se padhta hoon"
par.
```

### What actually happens when you create a Service today

```
kubectl apply -f service.yaml
          ↓
API Server accepts it
          ↓
          ├── Endpoints Controller runs
          │     → Creates/updates Endpoints object (still happens)
          │     → Same name as Service
          │
          └── EndpointSlice Controller runs (separate controller)
                → Creates/updates EndpointSlice object(s)
                → Name = service-name + random suffix

DONO SIMULTANEOUSLY EXIST KARTE HAIN.
```

Verify karo khud:

```bash
kubectl get endpoints
kubectl get endpointslices

# Dono mein same Service ka data dikhega
```

### Toh "switch" kahan hua?

```
KUBE-PROXY ne apna READING SOURCE switch kiya.

Kubernetes 1.19 se pehle:
  kube-proxy → Endpoints object padhta tha → iptables rules banata tha

Kubernetes 1.19+ (EndpointSlice stable):
  kube-proxy → EndpointSlice padhta hai → iptables rules banata hai
  (Endpoints object abhi bhi banta hai, lekin kube-proxy use IGNORE
  karta hai by default)

Matlab: dono objects exist karte hain, lekin kube-proxy ne source
switch kar liya. Endpoints object ab mostly BACKWARD COMPATIBILITY
ke liye exist karta hai — purane tools jo Endpoints padhte the
(third-party controllers, service meshes) wo abhi bhi kaam karein.
```

### Kab Endpoints object actually use hota hai ab?

```
1. PURANE TOOLS: Koi custom controller ya service mesh jo
   EndpointSlice ko support nahi karta — wo abhi bhi Endpoints
   padhta hai.

2. MANUAL ENDPOINTS (no selector): Ye important case hai.
   Jab Service mein koi selector nahi hota (external service
   ko Kubernetes Service ki tarah wrap karna ho), to tum
   MANUALLY Endpoints object banate ho:

   apiVersion: v1
   kind: Service
   metadata:
     name: external-db
   spec:
     ports:
     - port: 5432
     # NO selector — manually manage karenge

   apiVersion: v1
   kind: Endpoints
   metadata:
     name: external-db    # same naam ZAROORI hai
   subsets:
   - addresses:
     - ip: 203.0.113.5    # external DB server ka IP
     ports:
     - port: 5432

   USE CASE: Tumhara PostgreSQL AWS RDS par hai (Kubernetes ke
   bahar), lekin application code mein "db-service" naam se
   access karna chahte ho. Service + manual Endpoints se ye
   possible hota hai bina code change kiye.

3. HEADLESS SERVICE + STATEFULSET: Specific pod DNS entries
   ke liye Endpoints abhi bhi relevant hai.
```

### "Endpoints completely disable karo" — kya possible hai?

```
Haan — EndpointSlice Only Mode:

kube-controller-manager par ye flag set karo:
--controllers=-endpointcontroller

Ye Endpoints controller ko DISABLE kar deta hai — Endpoints
objects tab bhi nahi banenge. Sirf EndpointSlice banenge.

LEKIN: Ye rarely karte hain kyunki:
- Purane tools break ho sakte hain jo Endpoints padhte hain
- Backward compatibility khatam ho jati hai
- Default behavior (dono banao, kube-proxy EndpointSlice padhe)
  already optimal hai

PRACTICAL ANSWER: Production mein hum disable nahi karte.
Hum bas jaante hain ki kube-proxy EndpointSlice use kar raha
hai — Endpoints object exist karta hai lekin routing decisions
mein use nahi hota.
```

---

## Part 2: Service actually kya kaam karta hai — poora mechanism

```
Tumne poocha: "Service ka ACTUAL kaam kya hai?"

Seedha answer: Service khud KUCH NAHI KARTA runtime par.
Service sirf ek CONFIGURATION OBJECT hai jisse kube-proxy
padhta hai aur ACTUAL RULES banata hai.

Service = Blueprint
kube-proxy = Jo blueprint ko implement karta hai
```

### Step by step — packet Service se pod tak kaise pahunchta hai

```
Scenario:
  frontend pod (10.244.1.2) → db-service (ClusterIP: 10.96.0.15) → 
  db pods (10.244.2.3, 10.244.2.4, 10.244.2.5)

STEP 1: Service create hua
  kubectl apply -f db-service.yaml
  → API server ne Service object etcd mein store kiya
  → EndpointSlice controller ne ready pod IPs collect kiye
  → EndpointSlice object bana: {10.244.2.3, 10.244.2.4, 10.244.2.5}

STEP 2: kube-proxy ne change detect kiya
  kube-proxy har node par WATCH karta hai API server ko —
  Service aur EndpointSlice dono changes ke liye.
  Naya Service aur EndpointSlice dikh gaya.

STEP 3: kube-proxy ne IPTABLES RULES banaye (iptables mode mein)
  Har node par (EVERY node par, frontend node aur db nodes dono par):

  RULE 1 (DNAT rule):
  "Koi bhi packet jo 10.96.0.15:5432 par ja raha hai,
   uski destination IP randomly BADAL DO in mein se ek:
   10.244.2.3 ya 10.244.2.4 ya 10.244.2.5"

  Ye kernel level par hota hai (netfilter) — packet
  APPLICATION tak pahuncha hi nahi, PEHLE HI redirect
  ho gaya.

STEP 4: frontend pod ne request bheji
  curl db-service:5432
          ↓
  DNS resolve hua: db-service → 10.96.0.15 (CoreDNS se)
          ↓
  Packet create hua: [src: 10.244.1.2, dst: 10.96.0.15:5432]
          ↓
  Packet frontend pod ke network namespace se nikla
          ↓
  NODE KE KERNEL NE PACKET INTERCEPT KIYA (veth se bridge tak)
          ↓
  IPTABLES RULE MATCH HUA — "10.96.0.15 wala packet? DNAT karo"
          ↓
  Kernel ne destination IP CHANGE KI:
  [src: 10.244.1.2, dst: 10.96.0.15:5432]
           ↓
  [src: 10.244.1.2, dst: 10.244.2.4:5432]  (randomly pod 2 choose hua)
          ↓
  Ab yeh normal pod-to-pod packet ban gaya
  (Lecture 3 ka mechanism — CNI routing, veth, bridge)
          ↓
  10.244.2.4 (db pod 2) tak pahuncha

STEP 5: Response wapas
  db pod → [src: 10.244.2.4, dst: 10.244.1.2]
          ↓
  Kernel ne SNAT kiya response par:
  [src: 10.96.0.15, dst: 10.244.1.2]
  (frontend ko lagta hai Service ne reply kiya, actual pod
   ne nahi — connection tracking se ye hota hai)
          ↓
  frontend pod ko response mila
```

### IPVS mode mein kya fark hai

```
iptables mode:
  Rules ek LINKED LIST ki tarah hain — packet har rule se
  sequentially match karta hai. 10,000 Services = 10,000
  rules check = SLOW for large clusters.

IPVS mode:
  Hash table use karta hai — O(1) lookup, size se independent.
  10,000 Services = same speed as 10 Services.

Check karo tumhara cluster kaunsa use kar raha hai:
kubectl -n kube-system get configmap kube-proxy -o yaml | grep mode
```

### Service ka actual kaam — ek line mein

```
SERVICE OBJECT = "Ye configuration hai jo batata hai ki
  kaunse pods select karo aur kaunsa port listen karo"

KUBE-PROXY = "Main ye configuration padh kar ACTUAL IPTABLES/
  IPVS RULES banata hoon har node par jisse packets redirect
  hote hain"

SERVICE KHUD KUCH NAHI KARTA — ye sirf instructions ka ek
set hai jo kube-proxy implement karta hai.
```

### Complete picture — sab ek saath

```
kubectl apply -f service.yaml
        │
        ▼
   API Server (etcd mein store)
        │
        ├──────────────────────────────────┐
        ▼                                  ▼
Endpoints Controller              EndpointSlice Controller
(kube-controller-manager)         (kube-controller-manager)
        │                                  │
        ▼                                  ▼
  Endpoints object               EndpointSlice object(s)
  (backward compat)              (actual source of truth)
                                           │
                                           ▼
                                      kube-proxy
                                   (har node par watch)
                                           │
                                           ▼
                              iptables/IPVS rules update
                              (har node par, har bar jab
                               pods change hote hain)
                                           │
                                           ▼
                              Packet routing actually
                              YAHAN HOTI HAI — kernel mein
```

### Interview answer

```
Q: "Service kya hai aur actual kaam kya karta hai?"

A: "A Service is a Kubernetes API object that defines a stable
virtual IP and a selector for matching pods. The Service itself
has no runtime presence — it's just configuration. The actual
work is done by kube-proxy, which watches Service and
EndpointSlice objects on every node and programs iptables or
IPVS rules in the kernel. When a packet hits the Service's
ClusterIP, the kernel intercepts it before it reaches any
process and rewrites the destination to one of the backing
pod IPs — this DNAT happens entirely in kernel space, which
is why it's fast. kube-proxy updates these rules whenever
the EndpointSlice changes, which happens whenever pods
become ready or unready."

Q: "Endpoints aur EndpointSlice dono bante hain to
   difference kya hua practically?"

A: "Both objects are still created today for backward
   compatibility. The difference is in who reads them —
   kube-proxy now reads EndpointSlice by default, ignoring
   the Endpoints object for routing decisions. Endpoints
   still exists for legacy tools and for manually-managed
   Services without selectors. You can disable Endpoints
   controller entirely with --controllers=-endpointcontroller
   on the controller manager, but this is rarely done in
   production since backward compatibility matters."
```
---

Question: if we have check the reliability of any pod or application then how can it be done?

---
Explain these point to give clarity:
NodePort: Node ka port
TargetPort: Bhejna kahan h traffic or aapkha container kis port pe sun rha h traffic. If in dockerfile application is exposed at 8080 then pod pe bhi same hi port hona chahiye in target port. Container ka port. It is must to be defined.
Port: service ka port

by default port and targetPort are same but can be made different.

---
explain the concept hidden in this questions and also give correct understandable answer.
Question: Can you have services with no selectors? Yes (Create manual slice and manually add them)

---

- service ki external and internal ips have nothing to do with node ki external and internal ips.
- service ki external ips ho sakti h and node ki internal ip ho sakti h
- First list what all entities in kubernetes have external and internal ips and then also tell external and internal ips are used for what and where.

---

## L3, L4, L7 Load balancers

## Pehle OSI Model samjho — tabhi L4/L7 click karega

```
OSI Model = "Network communication ko 7 layers mein divide karo,
har layer ka ek specific kaam hai"

LAYER 7 — APPLICATION    → HTTP, HTTPS, gRPC, WebSocket, DNS
LAYER 6 — PRESENTATION   → Encryption/Decryption, SSL/TLS termination
LAYER 5 — SESSION        → Session management
LAYER 4 — TRANSPORT      → TCP, UDP (ports yahan hote hain)
LAYER 3 — NETWORK        → IP addresses, routing
LAYER 2 — DATA LINK      → MAC addresses
LAYER 1 — PHYSICAL       → Actual cables, signals

Exam/Interview mein sirf L3, L4, L7 matter karta hai baaki
almost kabhi nahi poocha jaata Kubernetes context mein.
```

---

## L3 Load Balancer

```
OPERATES AT: IP layer — sirf source IP aur destination IP dekhta hai

KAAM: IP address ke basis par traffic route karta hai

KUBERNETES RELEVANCE: Almost nahi use hota directly. Kuch BGP-based
solutions (like MetalLB in BGP mode) yahan operate karte hain.

WHAT IT SEES:
  [src IP: 203.0.113.1] → [dst IP: 10.96.0.15]
  Bas itna — koi port, koi protocol, koi content nahi.

LIMITATION: Port bhi nahi dekh sakta — matlab same IP par
  alag alag services differentiate nahi kar sakta.
```

---

## L4 Load Balancer — Kubernetes ka default

```
OPERATES AT: Transport layer — TCP/UDP + ports dekhta hai

KAAM: IP + Port combination ke basis par traffic route karta hai

WHAT IT SEES:
  [src IP: 203.0.113.1, src port: 54231]
  [dst IP: 10.96.0.15,  dst port: 443]

JO NAHI DEKHTA:
  - HTTP headers nahi
  - URL path nahi
  - Domain name nahi
  - Cookie nahi
  - Request content nahi — PACKET KA ANDAR NAHI DEKHA

KUBERNETES MEIN KAHAN USE HOTA HAI:
  - Service type: LoadBalancer → L4 load balancer provision
    karta hai (AWS NLB, GCP Network Load Balancer)
  - kube-proxy ka iptables/IPVS mechanism → L4 hai
  - NodePort → L4 hai

ANALOGY: Blindfolded delivery person — sirf address aur
  flat number dekh sakta hai, letter nahi padh sakta

PROS:
  - FAST — packet content parse nahi karna padta
  - Protocol agnostic — TCP ho ya UDP, koi bhi chalega
  - Koi SSL termination ki zarurat nahi (packet decrypt
    nahi karna — sirf forward karna hai)
  - Low latency

CONS:
  - Ek IP + port combination = ek Service hi handle kar sakta hai
  - URL based routing impossible (/api → service1, /web → service2)
  - Host header based routing impossible
  - Content based decisions impossible
```

### L4 ka real packet flow

```
Client → L4 LB → decides based on TCP port only → Pod

Example:
  Traffic on port 443 → db-service pods
  Traffic on port 80  → web-service pods

Alag alag ports pe alag routing — but SAME port par?
  L4 LB sirf round-robin ya random karta hai among pods.
  Content dekh ke smart routing — IMPOSSIBLE at L4.
```

---

## L7 Load Balancer — Kubernetes Ingress

```
OPERATES AT: Application layer — HTTP/HTTPS content dekhta hai

KAAM: HTTP headers, URL path, hostname, cookies, query params
  ke basis par traffic route karta hai

WHAT IT SEES (packet ka ANDAR):
  GET /api/users HTTP/1.1
  Host: myapp.example.com
  Cookie: session=abc123
  Content-Type: application/json

JO EXTRA DEKHTA HAI L4 ke upar:
  - URL path: /api, /web, /admin
  - Hostname: api.example.com vs web.example.com
  - HTTP method: GET, POST, PUT
  - Headers: Authorization, Cookie, Content-Type
  - Query parameters
  - Request body (rarely, performance cost high)

KUBERNETES MEIN KAHAN USE HOTA HAI:
  - Ingress resource → Ingress Controller (nginx, Traefik,
    HAProxy, AWS ALB) → L7 load balancer hai
  - Gateway API → L7 hai
  - Service mesh (Istio, Linkerd) → L7 hai

ANALOGY: Intelligent delivery person — envelope kholke content
  padhta hai aur tab decide karta hai kahan deliver karna hai
```

### L7 ka real routing example

```
EK SINGLE IP/PORT par multiple services handle karna:

myapp.example.com/api/users    → user-service pods
myapp.example.com/api/orders   → order-service pods
myapp.example.com/web          → frontend-service pods
admin.example.com              → admin-service pods (alag domain!)

YE L4 SE IMPOSSIBLE HAI — kyunki L4 sirf port dekh sakta hai,
/api aur /web same port 443 par aate hain, alag karna possible
nahi bina content dekhe.

Ingress YAML (L7 routing rules):
apiVersion: networking.k8s.io/v1
kind: Ingress
spec:
  rules:
  - host: myapp.example.com       # L7 — Host header dekh ke
    http:
      paths:
      - path: /api/users          # L7 — URL path dekh ke
        backend:
          service:
            name: user-service
      - path: /api/orders         # L7 — URL path dekh ke
        backend:
          service:
            name: order-service
  - host: admin.example.com       # L7 — alag domain
    http:
      paths:
      - path: /
        backend:
          service:
            name: admin-service
```

### SSL/TLS Termination — L7 ka important feature

```
L4 LB:
  Client → [ENCRYPTED packet] → L4 LB → [ENCRYPTED packet] → Pod
  L4 LB packet ko decrypt NAHI karta — sirf forward karta hai
  Har pod ko khud SSL handle karna padta hai
  YA: Pass-through SSL — encryption end-to-end, LB blind hai

L7 LB (SSL Termination):
  Client → [ENCRYPTED HTTPS] → L7 LB → [DECRYPTED HTTP] → Pod
                                  ↑
                            Yahan SSL
                         terminate hota hai
                         Certificate yahan
                         hota hai, pod par nahi

BENEFIT: Pods ko SSL certificates manage nahi karne padte,
  plain HTTP par kaam karte hain internally.
  Certificates ek jagah (Ingress Controller) manage hote hain.

Kubernetes mein:
  kubectl create secret tls myapp-tls --cert=cert.pem --key=key.pem
  
  Ingress mein reference karo:
  spec:
    tls:
    - hosts:
      - myapp.example.com
      secretName: myapp-tls    # L7 LB yahan se certificate padhega
```

---

## L4 vs L7 — complete comparison table

```
| Feature              | L4                      | L7                        |
|----------------------|-------------------------|---------------------------|
| Sees                 | IP + Port only          | Full HTTP content         |
| Speed                | Faster                  | Slower (parsing overhead) |
| SSL Termination      | Pass-through only       | Can terminate SSL         |
| URL routing          | No                      | Yes (/api → service1)     |
| Host-based routing   | No                      | Yes (domain based)        |
| Cookie/Header based  | No                      | Yes                       |
| Protocol support     | Any TCP/UDP             | HTTP/HTTPS/gRPC/WebSocket |
| Kubernetes object    | Service: LoadBalancer   | Ingress / Gateway API     |
| Cloud equivalent     | AWS NLB, GCP Net LB     | AWS ALB, GCP HTTP LB      |
| kube-proxy           | Yes (L4)                | No                        |
| Service Mesh         | No                      | Yes (Istio/Linkerd)       |
```

---

## Kubernetes mein exactly kahan kaunsa use hota hai

```
                    TRAFFIC FLOW

Internet
    │
    ▼
[L7 Load Balancer]          ← Ingress Controller (nginx/Traefik)
    │                          AWS ALB Ingress Controller
    │ URL/Host based routing    Gateway API
    │
    ▼
[L4 Load Balancer]          ← Service type: LoadBalancer
    │                          AWS NLB, GCP Network LB
    │ IP + Port routing         kube-proxy (iptables/IPVS)
    │
    ▼
[Pod]
    │
    └── Application Process
```

### Real world example — e-commerce app

```
myshop.example.com/products  → product-service   (L7 routing)
myshop.example.com/cart      → cart-service       (L7 routing)
myshop.example.com/payment   → payment-service    (L7 routing)
admin.myshop.com             → admin-service      (L7 host routing)

SAB EK HI IP AUR PORT 443 par — L7 LB ne path/host dekhke
alag-alag services ko route kiya.

Phir L4 (kube-proxy) ne har service ke andar load balance kiya
among multiple pods:
  product-service → [pod1, pod2, pod3] mein se ek choose kiya
```

---

## Interview answers

```
Q: "L4 aur L7 load balancer mein kya fark hai?"

A: "L4 operates at the transport layer — it routes traffic
based only on IP address and TCP/UDP port. It never looks
inside the packet, so it's fast but can't make content-based
routing decisions. L7 operates at the application layer — it
terminates the connection, reads HTTP headers, URL paths, and
host names, and routes based on that content. In Kubernetes,
kube-proxy is L4, Ingress controllers are L7. The tradeoff is
speed vs intelligence — L4 is faster because it doesn't parse
content, L7 enables path-based and host-based routing but adds
latency from parsing."

Q: "Kubernetes mein L7 routing kaise implement hoti hai?"

A: "Through Ingress objects and an Ingress Controller like
nginx or Traefik. The Ingress object defines the routing rules
— which hostname and URL path maps to which Service. The
Ingress Controller is a running pod that reads these rules and
acts as an actual L7 reverse proxy, terminating SSL, parsing
HTTP headers, and forwarding requests to the correct upstream
Service based on the rules. Without an Ingress Controller,
the Ingress object does nothing."

Q: "Ek hi domain par multiple microservices kaise serve karte ho?"

A: "L7 Ingress with path-based routing — /api → api-service,
/web → web-service — all behind a single external IP and port
443. L4 alone cannot do this because at port 443 you can't
distinguish /api from /web without reading the HTTP content."
```

---

**One mental model jo hamesha kaam aata hai:**

```
L4 = Delivery person jo sirf ENVELOPE dekha — address aur
     flat number. Andar kya hai, pata nahi, parwah nahi.

L7 = Delivery person jo ENVELOPE KHOLA, content padha, aur
     tab decide kiya "yeh HR department ka letter hai, 3rd
     floor jaunga" — content-aware decision.

Jitna upar layer, utna zyada information, utna zyada smart
routing possible — lekin utna zyada processing overhead bhi.
```

---

# Ingress, Ingress Controller, IngressClass

## 1. Problem first — Service type LoadBalancer ki limitation

```
Maan lo tumhare paas 5 microservices hain:
  user-service
  order-service
  payment-service
  product-service
  admin-service

Har service ke liye ek LoadBalancer type Service banao:
  user-service    → External IP: 52.10.1.1
  order-service   → External IP: 52.10.1.2
  payment-service → External IP: 52.10.1.3
  product-service → External IP: 52.10.1.4
  admin-service   → External IP: 52.10.1.5

PROBLEMS:
1. COST: 5 cloud load balancers = 5x cost (AWS ALB ~$16/month each
   = $80/month sirf load balancers ke liye)
2. DNS MANAGEMENT: 5 alag IPs ko 5 alag DNS entries se map karo
3. SSL: Har service par alag certificate manage karo
4. NO SMART ROUTING: /api → user-service, /orders → order-service
   ek hi domain se possible nahi

SOLUTION: INGRESS
   Ek single entry point jo sab traffic receive kare aur
   URL/host ke basis par sahi service tak route kare.
```

---

## 2. DNS — A Record aur CNAME Record (tumne flag kiya tha)

Ingress samajhne se pehle DNS records samajhna zaroori hai — kyunki Ingress host-based routing ke liye DNS par depend karta hai.

```
## A RECORD (Address Record)

DEFINITION: Ek domain name ya subdomain ko DIRECTLY ek IPv4
address se map karta hai.

EXAMPLE:
  myapp.example.com     → 52.10.1.1  (IP address)
  api.example.com       → 52.10.1.1  (same IP, alag subdomain)
  admin.example.com     → 52.10.1.2  (alag IP)

FORMAT:
  NAME                  TYPE    VALUE
  myapp.example.com     A       52.10.1.1

YE KUBERNETES MEIN KAISE USE HOTA HAI:
  Tumhara Ingress Controller ek External IP leta hai
  (LoadBalancer Service se). Us IP ko A record se map karte ho.
  
  Ingress Controller IP: 52.10.1.1
  
  A records banao DNS provider (Route53/Cloudflare) mein:
    api.myshop.com      A  52.10.1.1
    web.myshop.com      A  52.10.1.1
    admin.myshop.com    A  52.10.1.1
  
  SAARE SUBDOMAINS SAME IP POINT KARTE HAIN — Ingress Controller
  phir Host header dekh ke decide karta hai kahan route karna hai.

## CNAME RECORD (Canonical Name Record)

DEFINITION: Ek domain name ko DOOSRE domain name se map karta
hai — IP se nahi, naam se naam.

EXAMPLE:
  shop.mycompany.com    CNAME   myshop.example.com
                        ↑
                        "shop.mycompany.com ka matlab
                         myshop.example.com hai — usko
                         resolve karo, wahi IP use karo"

RESOLUTION CHAIN:
  User: shop.mycompany.com lookup karo
  DNS: CNAME → myshop.example.com
  DNS: myshop.example.com → A record → 52.10.1.1
  Final answer: 52.10.1.1

USE CASE: Cloud providers CNAME dete hain LoadBalancer ke liye
  kyunki cloud LB ka IP change ho sakta hai, lekin DNS name
  stable rehta hai.

  AWS ALB example:
    Tumhara ALB: k8s-abc123.ap-south-1.elb.amazonaws.com
    Tum apna domain CNAME karo:
    myapp.com  CNAME  k8s-abc123.ap-south-1.elb.amazonaws.com

  Agar AWS ka ALB IP change bhi kare, tumhara CNAME still
  correct rahega kyunki chain mein naam-se-naam link hai.

## A vs CNAME — quick comparison

| | A Record | CNAME Record |
|---|---|---|
| Maps | Domain → IP | Domain → Domain |
| Use karo jab | IP directly pata ho | Cloud LB DNS name mile |
| Chain | Direct (1 lookup) | Indirect (2 lookups) |
| IP change hone par | Manually update karo | Automatically resolves |
| Root domain par | Yes | No (root domain CNAME nahi hota) |

## OTHER RECORDS (brief — interview mein kabhi kabhi)

AAAA Record: A record jaisa lekin IPv6 address ke liye
  api.myapp.com  AAAA  2001:db8::1

MX Record: Mail Exchange — email ke liye
  myapp.com  MX  mail.myapp.com

TXT Record: Arbitrary text — domain verification ke liye
  (SSL certificate providers, Google site verification)

NS Record: Name Server — kaun DNS authority hai is domain ke liye
```

---

## 3. Ingress — kya hai exactly

```
DEFINITION: Ingress ek Kubernetes API object hai jo cluster ke
BAHAR se aane wale HTTP/HTTPS traffic ke liye ROUTING RULES
define karta hai.

IMPORTANT REPEAT: Ingress object KHUD KUCH NAHI KARTA.
Ye sirf rules ka declaration hai — "agar Host header ye hai
aur path ye hai, to is Service pe bhejo."

ACTUAL KAAM karta hai: INGRESS CONTROLLER (alag pod jo chal
raha hai cluster mein)
```

### Basic Ingress YAML — har field explain

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myshop-ingress
  namespace: production
  annotations:                          # Controller-specific config
    nginx.ingress.kubernetes.io/rewrite-target: /
    nginx.ingress.kubernetes.io/ssl-redirect: "true"

spec:
  ingressClassName: nginx               # KONSA controller handle karega

  tls:                                  # SSL/HTTPS config
  - hosts:
    - myshop.com
    - api.myshop.com
    secretName: myshop-tls-secret      # Certificate yahan stored hai

  rules:                                # ROUTING RULES
  - host: myshop.com                   # Host header match
    http:
      paths:
      - path: /products                 # URL path match
        pathType: Prefix               # Prefix ya Exact
        backend:
          service:
            name: product-service      # Kahan bhejo
            port:
              number: 80
      - path: /cart
        pathType: Prefix
        backend:
          service:
            name: cart-service
            port:
              number: 80

  - host: api.myshop.com               # Alag subdomain
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 8080

  defaultBackend:                       # Koi rule match nahi hua to
    service:
      name: fallback-service           # yahan bhejo (404 page, etc.)
      port:
        number: 80
```

### pathType — 3 types

```
Exact:  Path EXACTLY match karna chahiye
  path: /products
  /products     → MATCH ✅
  /products/    → NO MATCH ❌
  /products/123 → NO MATCH ❌

Prefix: Path prefix match karo
  path: /products
  /products     → MATCH ✅
  /products/    → MATCH ✅
  /products/123 → MATCH ✅
  /product      → NO MATCH ❌ (exact prefix, not substring)

ImplementationSpecific: Controller decide karta hai — avoid karo,
  non-portable hai
```

---

## 4. Ingress Controller — actual worker

```
DEFINITION: Ingress Controller ek pod (ya set of pods) hai jo
cluster mein run karta hai, Ingress objects ko WATCH karta hai,
aur actual L7 load balancing/routing implement karta hai.

Ye ek REVERSE PROXY hai jo:
  1. Ingress rules padhta hai
  2. Traffic receive karta hai (external se)
  3. Host + Path match karta hai
  4. Sahi Service ko forward karta hai

POPULAR CONTROLLERS:
  nginx-ingress-controller  → Most common, nginx based
  Traefik                   → Auto-discovery, Let's Encrypt built-in
  HAProxy                   → High performance
  AWS ALB Controller        → AWS native, directly ALB use karta hai
  GKE Ingress Controller    → GCP native
  Kong                      → API Gateway features + Ingress
  Istio Gateway             → Service mesh based
```

### Ingress Controller deploy hone ke baad ka flow

```
STEP 1: Ingress Controller install karo (helm se mostly)
  helm install ingress-nginx ingress-nginx/ingress-nginx

STEP 2: Controller ek Service (type: LoadBalancer) banata hai
  → Cloud provider se external IP milti hai
  → YE IP HAI TUMHARA SINGLE ENTRY POINT

STEP 3: DNS mein A record banao
  myshop.com      A  <controller-external-ip>
  api.myshop.com  A  <controller-external-ip>

STEP 4: Ingress object apply karo
  kubectl apply -f ingress.yaml

STEP 5: Controller Ingress object ko WATCH karta hai, apna
  nginx.conf update karta hai internally

STEP 6: Traffic flow:
  User → myshop.com/products
       → DNS resolve → controller-external-ip
       → Ingress Controller pod receive karta hai
       → Host header: "myshop.com" check kiya
       → Path: "/products" check kiya
       → Rule match: product-service
       → Forward to product-service:80
       → product-service → product pod
```

---

## 5. IngressClass — kya hai, kyu zaroori hai

### Problem — multiple controllers

```
Scenario: Tumhare cluster mein 2 Ingress Controllers hain:
  1. nginx controller — internal apps ke liye
  2. AWS ALB controller — external customer traffic ke liye

Tum ek Ingress banate ho — KONSA controller ise handle karega?
DONO? Sirf ek? Kaunsa?

Chaos hoga agar koi mechanism nahi ho decide karne ka.

SOLUTION: IngressClass
```

### IngressClass object

```yaml
# IngressClass 1 — nginx ke liye
apiVersion: networking.k8s.io/v1
kind: IngressClass
metadata:
  name: nginx
  annotations:
    ingressclass.kubernetes.io/is-default-class: "true"  # DEFAULT
spec:
  controller: k8s.io/ingress-nginx   # konsa controller implement
                                      # karega ye class

---
# IngressClass 2 — AWS ALB ke liye
apiVersion: networking.k8s.io/v1
kind: IngressClass
metadata:
  name: alb
spec:
  controller: ingress.k8s.aws/alb
  parameters:
    apiGroup: elbv2.k8s.aws
    kind: IngressClassParams
    name: alb-params
```

### Ingress mein IngressClass reference karo

```yaml
spec:
  ingressClassName: nginx    # nginx controller handle karega
  rules: ...

---
spec:
  ingressClassName: alb      # AWS ALB controller handle karega
  rules: ...
```

### Default IngressClass

```
Agar Ingress mein ingressClassName specify nahi kiya:
  → DEFAULT IngressClass wala controller handle karega

Default kaise mark karo:
  IngressClass par annotation:
  ingressclass.kubernetes.io/is-default-class: "true"

IMPORTANT: Sirf EK IngressClass default honi chahiye.
Agar multiple default hain:
  → Admission controller nayi Ingress creation BLOCK kar
    dega jab tak ingressClassName explicitly specify na karo.
```

### Old annotation vs new ingressClassName

```
OLD WAY (deprecated, kubernetes 1.18 se pehle):
  metadata:
    annotations:
      kubernetes.io/ingress.class: "nginx"   ← DEPRECATED

NEW WAY (current, use karo):
  spec:
    ingressClassName: nginx                   ← CORRECT

DIFFERENCE:
  Old annotation: sirf controller ka naam tha, koi object
    nahi tha — unstructured, each controller ne apna
    interpretation rakha
  
  New ingressClassName: ek actual IngressClass OBJECT ko
    REFERENCE karta hai — structured, additional parameters
    support karta hai, validated hai

CURRENT STATE:
  Old annotation abhi bhi KAAM KARTA HAI (backward
  compatibility ke liye) — lekin logs mein warning aata hai.
  Production mein new field use karo.
```

---

## 6. Annotations — controller-specific configuration

```
PROBLEM: Ingress spec bahut basic hai — path routing aur TLS
bas. Lekin real world mein chahiye:
  - Rate limiting
  - Auth (basic auth, OAuth)
  - CORS headers
  - Custom timeouts
  - Rewrite rules
  - Redirect HTTP → HTTPS
  - Body size limits

SOLUTION: Annotations — controller apne specific annotations
define karta hai jo iske behavior ko configure karte hain.

YE ANNOTATIONS STANDARD NAHI HAIN — nginx ke annotations
Traefik mein kaam nahi karenge aur vice versa.
```

### nginx Ingress Controller annotations — common ones

```yaml
metadata:
  annotations:
    # REWRITE
    nginx.ingress.kubernetes.io/rewrite-target: /
    # Incoming path /api/v1/users se /api prefix hatao,
    # sirf /users forward karo service ko

    # SSL
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    # HTTP automatically HTTPS redirect karo

    nginx.ingress.kubernetes.io/force-ssl-redirect: "true"
    # Even agar SSL terminate hua pehle (proxy ke through)
    # force redirect karo

    # RATE LIMITING
    nginx.ingress.kubernetes.io/limit-rps: "10"
    # 10 requests per second per IP se zyada allow mat karo

    nginx.ingress.kubernetes.io/limit-connections: "5"
    # Ek IP se max 5 simultaneous connections

    # TIMEOUTS
    nginx.ingress.kubernetes.io/proxy-connect-timeout: "30"
    nginx.ingress.kubernetes.io/proxy-send-timeout: "60"
    nginx.ingress.kubernetes.io/proxy-read-timeout: "60"

    # BODY SIZE
    nginx.ingress.kubernetes.io/proxy-body-size: "10m"
    # Max 10MB request body (file upload ke liye badhao)

    # BASIC AUTH
    nginx.ingress.kubernetes.io/auth-type: basic
    nginx.ingress.kubernetes.io/auth-secret: basic-auth-secret
    nginx.ingress.kubernetes.io/auth-realm: "Authentication Required"

    # CORS
    nginx.ingress.kubernetes.io/enable-cors: "true"
    nginx.ingress.kubernetes.io/cors-allow-origin: "https://myapp.com"
    nginx.ingress.kubernetes.io/cors-allow-methods: "GET, POST, OPTIONS"

    # CUSTOM HEADERS
    nginx.ingress.kubernetes.io/configuration-snippet: |
      add_header X-Frame-Options "SAMEORIGIN";
      add_header X-Content-Type-Options "nosniff";

    # STICKY SESSIONS (same user same pod par jaye)
    nginx.ingress.kubernetes.io/affinity: "cookie"
    nginx.ingress.kubernetes.io/session-cookie-name: "INGRESSCOOKIE"
    nginx.ingress.kubernetes.io/session-cookie-expires: "172800"

    # WHITELIST — sirf ye IPs allow karo
    nginx.ingress.kubernetes.io/whitelist-source-range: "10.0.0.0/8,192.168.0.0/16"
```

### Rewrite annotation — detail mein (confusing hota hai)

```
Scenario:
  External URL: myapp.com/api/users
  Internal Service: user-service port 80, expects: /users (not /api/users)

Bina rewrite:
  Request: GET /api/users
  Service receive karta hai: GET /api/users
  → Service ka route /api/users pe nahi hai → 404

Rewrite ke saath:
  nginx.ingress.kubernetes.io/rewrite-target: /$2

  path: /api(/|$)(.*)
  
  Incoming: GET /api/users
  nginx strips /api → forwards: GET /users
  → Service ka route /users pe hai → 200 OK

REGEX CAPTURE GROUPS:
  (/|$)  → group 1: slash ya end
  (.*)   → group 2: baaki sab (users, orders, etc.)
  $2     → group 2 ka value rewrite target mein use karo
```

---

## 7. Complete picture — sab ek saath

```
DNS SETUP:
  myshop.com      A  → 52.10.1.1 (Ingress Controller IP)
  api.myshop.com  A  → 52.10.1.1 (same IP)

CLUSTER:
  IngressClass: nginx (default)
  Ingress Controller: nginx pods (LoadBalancer IP: 52.10.1.1)
  
  Ingress Rules:
    myshop.com/products → product-service:80
    myshop.com/cart     → cart-service:80
    api.myshop.com/     → api-service:8080

TRAFFIC FLOW:
  User types: https://myshop.com/products
      ↓
  DNS: myshop.com → 52.10.1.1
      ↓
  Packet: [dst: 52.10.1.1:443]
      ↓
  Ingress Controller receives
      ↓
  SSL Terminate (TLS secret se certificate use karke)
      ↓
  Decrypted HTTP request:
    Host: myshop.com
    Path: /products
      ↓
  Rule match: myshop.com + /products → product-service:80
      ↓
  Forward to product-service (L4 kube-proxy)
      ↓
  product-service → product pod
      ↓
  Response wapas (SSL re-encrypt if needed)
      ↓
  User ko response
```

---

## 8. Interview answers

```
Q: "Ingress aur Ingress Controller mein kya fark hai?"

A: "Ingress is a Kubernetes API object — it's just a declaration
of routing rules: which hostname and path should map to which
Service. It has no runtime behavior on its own. The Ingress
Controller is the actual implementation — a pod running in the
cluster that watches Ingress objects and acts as a real L7
reverse proxy, receiving external traffic and forwarding it
based on the rules. Without an Ingress Controller, Ingress
objects are completely inert."

Q: "IngressClass kyu zaroori hai?"

A: "When multiple Ingress Controllers exist in a cluster —
say nginx for internal apps and AWS ALB for external traffic —
every Ingress object needs to declare which controller should
handle it. IngressClass is a cluster-scoped resource that
represents a controller implementation. Ingress objects
reference it via spec.ingressClassName. Before 1.18, this
was done via the kubernetes.io/ingress.class annotation,
which is now deprecated — spec.ingressClassName is the
correct field today."

Q: "A record aur CNAME mein kya fark hai?"

A: "An A record maps a domain directly to an IPv4 address.
A CNAME maps a domain to another domain name, which then
resolves to an IP. In Kubernetes, when a cloud provider
gives you a LoadBalancer hostname instead of a stable IP —
like AWS does with ALBs — you CNAME your domain to that
hostname. This way, if the cloud provider's IP changes,
your DNS still resolves correctly through the chain. A
records are used when you have a stable IP, like a static
external IP on GCP."

Q: "Annotations Ingress mein kyu use hote hain?"

A: "The Ingress spec is intentionally minimal — just
routing rules and TLS config. Controller-specific features
like rate limiting, auth, rewrite rules, CORS, and session
affinity can't be standardized across all controllers, so
each controller defines its own annotations. nginx uses
nginx.ingress.kubernetes.io/* prefix, Traefik uses its own.
This is a known trade-off — Ingress objects become
controller-specific through annotations, which reduces
portability. Gateway API solves this by having typed
extension points instead of free-form annotations."
```
