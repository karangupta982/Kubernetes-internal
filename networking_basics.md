# Kubernetes Networking: Complete Reference
(Source: ChatGPT conversation + your own diagram, audited and deepened)

## Diagram reference (your own example, keep using it to self-test)
```text
cluster-gen
├── node-cpu
│   ├── ns-dev → pod-frontend(c1,c2), pod-db(c3)
│   └── ns-stg → pod-poc(c4)
└── node-gpu
    └── ns-ml → pod-ml(c5)
cluster-peak
└── node-global
    └── ns-global → pod-peak(c6,c7,c8)
```

Arrows in your diagram, confirmed correctly mapped by ChatGPT:
① c1↔c3 same namespace, different pod, same node
② c3↔c4 different namespace (ns-dev ↔ ns-stg), same node
③ pod-frontend↔pod-ml different node, SAME cluster
④ pod-ml↔pod-peak different CLUSTER entirely
⑤ c7↔c8 same pod

---

## 1) Entity hierarchy — top to bottom (confirmed correct, keeping as-is)

Physical Machine/VM → Cluster → Control Plane + Worker Nodes → Namespace
→ Deployment/StatefulSet/Job → Pod → Container → Process

---

## 2) THE MECHANISM LAYER — what ChatGPT mentioned but didn't fully connect

Ye section sabse important hai kyuki interview mein "Kubernetes networking
kaise kaam karta hai" pucha jata hai aur log sirf "Service/DNS" bol dete
hain, ACTUAL packet path nahi bata pate. Yahan exact, verified mechanism hai.

### 2.1 Same-node, different-pod — veth pair aur bridge ka exact role

```
Har Pod banne par, kubelet (CRI ke through) ek NETWORK NAMESPACE banata
hai us Pod ke liye. Phir CNI plugin ye steps karta hai:

1. Ek "veth pair" banaya jata hai — socho ye ek VIRTUAL CABLE hai jiske
   DO SIRE (ends) hain:
   - EK sira Pod ke network namespace ke andar jata hai (Pod ke andar
     "eth0" interface ban jata hai)
   - DOOSRA sira NODE ke main network namespace mein reh jata hai

2. Node ke side wala veth-end ek VIRTUAL BRIDGE se connect hota hai
   (naam: "cni0" Flannel mein, ya CNI-specific naam). Ye bridge ek
   VIRTUAL SWITCH ki tarah kaam karta hai — socho ek office mein switch
   jisse saare computers connect hote hain.

3. Jab Pod A (veth se cni0 bridge se connected) Pod B ko packet bhejta
   hai (DONO SAME NODE par), packet:
   Pod A → veth (Pod A ka) → cni0 bridge → veth (Pod B ka) → Pod B

   YE PACKET KABHI NODE SE BAHAR NAHI JATA — pura andar hi handle ho
   jata hai Linux kernel ke through. ISLIYE same-node Pod-to-Pod
   communication SABSE FAST hota hai.

Interview one-liner: "Each pod gets a virtual ethernet (veth) pair —
one end inside the pod's network namespace, the other attached to a
bridge on the node. For same-node pod-to-pod traffic, the bridge acts
as a virtual switch, so packets never leave the node's kernel."
```

### 2.2 Cross-node — the part most people get vague on

```
PROBLEM: Pod A (node-cpu, IP 10.244.1.2) ko Pod B (node-gpu, IP
10.244.2.2) se baat karni hai. Bridge sirf SAME NODE ke andar kaam
karta hai — node-cpu ka bridge node-gpu ke bridge se directly connected
nahi hai. Toh packet node ke BAHAR kaise jaye?

DO APPROACHES hain (CNI plugin decide karta hai konsa use karna hai):

APPROACH 1 — OVERLAY NETWORK (encapsulation, e.g. Flannel VXLAN mode):
  Packet ko ek DOOSRE packet ke ANDAR "wrap" kar diya jata hai
  (VXLAN encapsulation — UDP port 4789 use hota hai):

  Original packet: [Pod A IP → Pod B IP, actual data]
  Wrapped packet:  [Node-CPU IP → Node-GPU IP, [Original packet andar]]

  Ab physical network ko sirf NODE IPs pata hone chahiye (jo already
  routable hain), Pod IPs ka pata hona zaroori nahi. Destination node
  par pahunch kar, "decapsulate" hota hai (outer wrapper hata diya
  jata hai), aur original packet andar wale bridge se Pod B tak
  pahuncha diya jata hai.

  TRADE-OFF: Extra header add hone se thoda overhead hota hai (VXLAN
  ~50 bytes extra), aur MTU (Maximum Transmission Unit) thoda kam ho
  jata hai effectively — high-throughput workloads ke liye ye
  PERFORMANCE COST hai.

APPROACH 2 — DIRECT ROUTING / BGP (e.g. Calico's default mode, no
encapsulation):
  Yahan koi wrapping nahi hoti. Har node ke routing table mein directly
  ENTRY hoti hai: "Pod CIDR range X ke liye, node Y ke through jao."
  Ye entries BGP protocol se automatically propagate hoti hain saare
  nodes mein. Underlying network (jaise on-prem datacenter routers)
  ko Pod IPs route karne layak hona chahiye.

  ADVANTAGE: Koi encapsulation overhead nahi, FASTER hai.
  REQUIREMENT: Underlying network infrastructure ko BGP support karna
  chahiye (cloud mein ye thoda tricky ho sakta hai, on-prem datacenter
  mein zyada common hai).

Interview-ready summary: "Cross-node pod traffic relies on the CNI
plugin's routing strategy — either overlay networking, where the
original pod packet is encapsulated inside a node-to-node packet using
something like VXLAN, or direct routing, where the underlying network
is configured (often via BGP) to natively route pod CIDR ranges between
nodes without any encapsulation. Overlay is simpler to set up anywhere;
direct routing is faster but needs underlying network support."
```

### 2.3 Service — kya ye ek "running process" hai? (NO — clear this up explicitly)

```
ChatGPT ne sahi kaha: "Service is NOT a process." Yahan thoda zyada
precision:

Service banane par, Kubernetes:
1. Ek STABLE virtual IP allocate karta hai (ClusterIP) — ye IP kisi
   NETWORK INTERFACE par actually exist NAHI karti, ye purely VIRTUAL hai
2. kube-proxy (jo HAR node par chalta hai) is mapping ko IPTABLES ya
   IPVS rules mein convert kar deta hai

Jab koi packet Service IP (e.g. 10.96.0.15) par jata hai:
- iptables/IPVS rule check karte hain: "10.96.0.15 ka matlab hai
  EK IN-THESE-IPs choose karo: 10.244.1.3, 10.244.1.4, 10.244.2.7..."
  (jo bhi Pods is Service ke "Endpoints" hain)
- DNAT (Destination NAT) ho jata hai — packet ki destination IP
  CHANGE ho jati hai actual chosen Pod IP mein, BEFORE packet kahin
  bhejne se pehle hi (kernel level par hi ye decision ho jata hai)

Isliye Service "Iska apna koi process nahi hai" — ye sirf RULES ka
ek set hai jo HAR node par maintained hota hai.
```

---

## 3) NetworkPolicy — MISSING ENTIRELY from this conversation, must add

This is a major gap — your ChatGPT conversation correctly says "by default all pods can talk to all pods" but never covers HOW to restrict that, which is heavily tested (Services & Networking = 20% of CKA, and this is THE core security mechanism within that domain).

```
## NetworkPolicy — full explanation (CKA aur CKS dono ke liye critical)

DEFAULT BEHAVIOR (jo ChatGPT ne sahi bataya): Kubernetes mein by
default, FLAT NETWORK hai — koi bhi Pod kisi bhi doosre Pod se baat
kar sakta hai, chahe namespace alag ho. YE INSECURE hai production
ke liye (e.g. dev namespace ka pod accidentally production database
namespace se baat kar sakta hai).

NetworkPolicy is default behavior ko RESTRICT karne ka tarika hai.

### Important precondition (often missed):

NetworkPolicy SIRF tab kaam karti hai jab tumhara CNI plugin ise
SUPPORT karta ho. Flannel (basic mode) NetworkPolicy ENFORCE NAHI
karta — agar tum NetworkPolicy likh do Flannel cluster mein, wo
SILENTLY IGNORE ho jayegi, koi error nahi aayega (yeh DANGEROUS
gotcha hai). Calico, Cilium, Weave — ye sab NetworkPolicy support
karte hain.

### Basic structure:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-from-other-namespaces
  namespace: ns-dev
spec:
  podSelector: {}            # empty selector = ALL pods in ns-dev
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector: {}        # only allow from pods within SAME namespace
```

### Key concept: DEFAULT DENY vs SELECTIVE ALLOW

Jaise hi EK NetworkPolicy kisi Pod ko "select" karti hai (via
podSelector), us Pod ke liye DEFAULT BEHAVIOR badal jata hai —
ab SIRF wahi traffic allowed hai jo explicitly specified hai, baaki
SAB DENY ho jata hai (for that traffic direction — Ingress/Egress).

Agar Pod KISI BHI NetworkPolicy se selected nahi hai, wo Pod abhi
bhi OLD DEFAULT (sab allow) behavior follow karta hai.

### Example — exactly matching your diagram's scenario:

Maan lo tum chahte ho ns-stg (pod-poc) SIRF apne hi namespace ke
andar baat kare, ns-dev (pod-db) se NA baat kar sake:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: isolate-stg
  namespace: ns-stg
spec:
  podSelector: {}
  policyTypes:
  - Egress
  egress:
  - to:
    - podSelector: {}    # sirf ns-stg ke andar wale pods allowed
```

Ye apply hone ke baad, tumhare diagram ka arrow ② (c3↔c4 cross-namespace)
BLOCKED ho jayega.

Interview answer: "By default Kubernetes has no network isolation —
any pod can reach any other pod across namespaces. NetworkPolicy
objects implement an allow-list model: once a pod is selected by any
policy, all traffic not explicitly permitted is denied for that
direction. It's enforced by the CNI plugin, not the API server itself
— so the policy is silently ineffective if your CNI doesn't support it,
which is a common production gotcha."
```

---

## 4) DNS Resolution — depth on what ChatGPT mentioned briefly

```
## CoreDNS — exact resolution path, full FQDN breakdown

Service ka full DNS name:
<service-name>.<namespace>.svc.cluster.local

Breaking it down (yaad karo har part ka matlab):
- <service-name>     : jo Service banaya
- <namespace>        : kis namespace mein
- svc                : "service" type record (Pods ka bhi DNS record
                        ban sakta hai, "pod" subdomain ke through,
                        rarely use hota hai)
- cluster.local       : DEFAULT cluster domain (configurable, but
                        cluster.local sabse common default hai)

### Shortcuts (important — exam mein time bachata hai):

Agar SAME namespace se call kar rahe ho:
  curl db-service                          # sirf naam, kaam karega

Agar DIFFERENT namespace se:
  curl db-service.ns-dev                   # namespace add karna padega
  curl db-service.ns-dev.svc.cluster.local  # full form, hamesha kaam karega

YE KYU KAAM KARTA HAI: Pod ke /etc/resolv.conf mein "search domains"
configured hote hain (jaise "ns-dev.svc.cluster.local", "svc.cluster.local",
"cluster.local") — isliye short naam likhne par bhi resolver automatically
try kar leta hai pura FQDN bana ke.

Verify karne ka command:
kubectl exec -it <pod> -- cat /etc/resolv.conf

### CoreDNS kaise kaam karta hai (Pod-level se):

1. Pod DNS query bhejta hai → Pod ke resolv.conf mein nameserver
   CoreDNS Service ki IP point karta hai (typically kube-system
   namespace mein "kube-dns" Service)
2. CoreDNS apne internal records check karta hai (jo automatically
   update hote hain jab Services create/delete hoti hain — ye watch
   mechanism se hota hai, Lecture 2 wala concept yaad karo)
3. CoreDNS Service ki ClusterIP return karta hai
4. Pod ab us IP par actual request bhejta hai (jo upar discuss kiya
   gaya iptables/IPVS mechanism se Pod tak pahunchti hai)

Troubleshooting command (CKA mein common):
kubectl run debug --image=busybox -it --rm -- nslookup db-service.ns-dev
```

---

## 5) Service Types — ChatGPT mentioned LoadBalancer/NodePort/Ingress in passing, full comparison needed

```
## Service Types — comparison table (exam-critical, often confused)

| Type         | Accessible From          | Use Case                          |
|--------------|---------------------------|-------------------------------------|
| ClusterIP    | Only inside cluster       | Default; internal microservice talk |
| NodePort     | External, via <NodeIP>:<port> (port range 30000-32767) | Quick external access, dev/test |
| LoadBalancer | External, via cloud LB    | Production external access (needs cloud provider / CCM) |
| ExternalName | DNS alias to external service | Pointing to a service OUTSIDE the cluster (e.g. external DB) |

NodePort ACTUALLY internally Service hi hai — bas iske upar ek EXTRA
port (every node par same port) bhi open hota hai jo directly node ki
IP se accessible hai. NodePort Service KHUD ek ClusterIP bhi rakhta hai
internally — NodePort sirf ADDITIONAL access path hai.

LoadBalancer Service, NodePort + ClusterIP DONO ka superset hai — plus
cloud provider se actual external Load Balancer provision karwata hai
(Lecture 2 mein Cloud Controller Manager wala concept yaad karo — yahi
karta hai ye provisioning).

Interview one-liner: "Each Service type builds on the previous one —
NodePort is a ClusterIP plus an exposed port on every node, and
LoadBalancer is a NodePort plus a cloud-provisioned external load
balancer pointing at those node ports."
```

---

## 6) Ingress — mentioned once in passing, needs its own section

```
## Ingress — kyu zaroori hai (ChatGPT ne sirf naam liya tha)

PROBLEM: Agar tumhare paas 10 different microservices hain, har ek
ke liye ALAG LoadBalancer Service banana EXPENSIVE hai (har LoadBalancer
= ek real cloud load balancer = real cost) aur messy hai (10 alag
external IPs manage karna).

SOLUTION: Ingress — EK SINGLE entry point banata hai jo URL PATH ya
HOSTNAME ke basis par traffic ko DIFFERENT internal Services tak
route karta hai.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-ingress
spec:
  rules:
  - host: myapp.example.com
    http:
      paths:
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: api-service
            port:
              number: 80
      - path: /web
        pathType: Prefix
        backend:
          service:
            name: web-service
            port:
              number: 80
```

IMPORTANT: Ingress OBJECT khud kuch nahi karta — ye sirf RULES define
karta hai. Actual traffic handle karne ke liye "Ingress Controller"
chalana ZAROORI hai (e.g. nginx-ingress-controller, Traefik). Bina
controller ke, Ingress object effectless hai — bas etcd mein baitha
rehta hai.

Newer alternative: GATEWAY API (Lecture 2 mein mention hua tha) — ye
Ingress se zyada flexible/extensible hai, naye clusters mein increasingly
common, lekin CKA abhi dono cover karta hai.

Path: Internet → Ingress Controller → match rule → Service → Pod
(Tumhare diagram ka CASE 12 yahi hai)
```

---

## 7) Egress — your diagram's "Pod → Internet" case, deepened

```
## Egress traffic — Pod se bahar jaana

Default: ALLOWED hota hai (jaise ChatGPT ne kaha). Pod directly
internet se baat kar sakta hai (e.g. external API call, package
download).

MECHANISM: Pod ka traffic node tak aata hai (veth/bridge), phir node
par SNAT (Source NAT) hota hai — Pod ki INTERNAL IP (10.244.x.x, jo
internet par routable NAHI hoti) ko NODE ki PUBLIC/routable IP mein
badal diya jata hai before internet par bhejne ke liye. Response wapas
aane par, reverse translation hoti hai.

Restriction: NetworkPolicy ke "Egress" rules se control kiya ja sakta
hai (upar example diya gaya hai).

Production security practice: Egress ko bhi restrict karna chahiye
("default deny egress, sirf zaroori external endpoints allow karo")
— CKS mein ye heavily tested hai.
```

---

## 8) Cross-Cluster Communication — depth on what your diagram's arrow ④ represents

```
## Cluster-to-Cluster — full options comparison (ChatGPT listed, depth add karte hain)

Your diagram: pod-ml (cluster-gen) ↔ pod-peak (cluster-peak) — BY
DEFAULT IMPOSSIBLE, confirmed correct by ChatGPT.

| Method | How it works | Complexity |
|---|---|---|
| Expose via LoadBalancer + public internet | Service externally exposed, other cluster calls it like any external API | Low, but traffic goes over public internet (needs TLS/auth) |
| VPN between clusters | Private network tunnel between cluster networks | Medium |
| Service Mesh (Istio, Linkerd) multi-cluster mode | Mesh manages cross-cluster service discovery + mTLS automatically | High setup, but most feature-rich |
| Multi-cluster CNI (Cilium Cluster Mesh) | CNI-level — pod IPs become routable across clusters directly | High, but most "native"-feeling |
| Submariner | Purpose-built tool for connecting Kubernetes cluster networks | Medium |

Interview context: "Cross-cluster pod communication isn't a single
standard mechanism — it depends on whether you want simple exposed
endpoints, a full mesh with mutual TLS, or near-native pod-to-pod
routing. Tools like Cilium Cluster Mesh or Submariner aim for the
latter; simpler setups just expose a Service externally."
```

---

## 9) The Complete Communication Matrix — kept from ChatGPT, confirmed accurate, one addition

```
| Source | Destination | Possible? |
|---|---|---|
| Container | Same Pod Container | ✅ localhost |
| Container | Container in Other Pod | ✅ via Pod IP/Service (NEVER container-to-container directly) |
| Pod | Pod Same Namespace | ✅ |
| Pod | Pod Different Namespace | ✅ (default) — ❌ if NetworkPolicy restricts |
| Pod | Pod Different Node | ✅ via CNI routing/overlay |
| Pod | Service | ✅ |
| Pod | Internet | ✅ (default) — ❌ if Egress NetworkPolicy restricts |
| Internet | Pod | ❌ directly — ✅ via Service/Ingress |
| Pod | Pod Different Cluster | ❌ by default — ✅ via VPN/Mesh/external expose |
| Namespace | Namespace | ❌ — pods communicate, not namespaces |
| Node | Node | ✅ internal cluster networking |

ADDITION (missing row): NetworkPolicy ka effect explicitly add kiya
gaya hai upar — "default allow" ko "✅ default, ❌ if restricted" se
replace kiya, kyuki ye real-world production setups mein bahut common
hai aur exam mein bhi NetworkPolicy task aata hai.
```

---

## 10) Core principle — keeping ChatGPT's framing, it's exactly right

```
"Every Pod gets its own IP address, and every Pod can talk to every
other Pod in the cluster without NAT."

Ye Kubernetes networking model ki FOUNDATIONAL requirement hai
(officially called the "Kubernetes Networking Model" in the docs).
Teen guarantees jo har CNI plugin ko follow karni hoti hai:
1. Pods can communicate with all other pods without NAT
2. Nodes can communicate with all pods without NAT
3. The IP a pod sees itself as is the SAME IP others see it as
   (no address translation hiding identity)
```
```
---

A few honest closing notes:

**Your diagram and ChatGPT's case mapping were actually correct** — I went through it carefully expecting to find the kind of error these tools usually introduce, and didn't. That's a genuinely good diagram you built; keep using it as your self-test tool (cover the answer, trace each arrow yourself).

**NetworkPolicy was the single biggest gap** — it's not just "missing content," it's the actual *security control* layer for everything else this section describes, and it's directly testable on both CKA and CKS. If you only add one new section to your notes from this batch, make it that one.

**One thing worth doing before Lecture 4:** go back to your diagram, and for each of the 13 cases, try writing the *one-line interview answer* yourself from memory before checking the deep explanation. That active-recall step is what actually moves this from "I read it" to "I can say it under pressure" — which is the whole point given your job goal.
