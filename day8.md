# Lecture 8 — Namespaces Deep Dive, Multi-tenancy, CPU/Memory, QoS, Taints/Tolerations, Affinity, TopologySpreadConstraints

---

## 1. Kubernetes Hierarchy — Corrected aur Complete

```
Tumhare notes mein hierarchy slightly off thi. Correct version:

Cluster
  └── Nodes (physical machines — parallel concept, not inside namespace)
  └── Namespaces (logical grouping — parallel to nodes at API level)
        └── Services
        └── Deployments
              └── ReplicaSets
                    └── EndpointSlices
                    └── Pods
                          └── Containers
                                └── Linux processes
                                      └── cgroups (resource enforcement)

IMPORTANT CORRECTION:
Tumhare notes mein "nodes then namespaces" likha hai — jaise
nodes ke andar namespaces hain. YE GALAT HAI (humne already
lecture 3 notes mein clear kiya tha). Namespaces cluster-level
logical boundary hain, nodes physical infrastructure hain.
Dono parallel dimensions mein exist karte hain.

ALSO: "endpoint slice then pods" — ye bhi reverse hai. Pods
exist karte hain, aur unke IPs ko track karne ke liye
EndpointSlice banata hai controller. Pods → EndpointSlice
(nahi ki EndpointSlice → Pods).
```

---

## 2. Default Namespaces — Har Ek Ka Exact Role

```
4 default namespaces Kubernetes automatically banata hai:

## default
  Agar tum koi resource banate ho bina namespace specify kiye,
  ye yahan jaata hai. Ye "miscellaneous drawer" hai cluster ka.
  Production mein ye empty rakhna chahiye — apne apps ke liye
  separate namespaces banao.

## kube-system
  Kubernetes ke CORE COMPONENTS yahan run hote hain:
  - kube-dns (CoreDNS pods)
  - kube-proxy (DaemonSet — har node par)
  - metrics-server
  - Cloud provider ke injected components (GKE mein: fluentbit
    logging agent, GKE metrics agent, CSI storage driver,
    cluster autoscaler, konnectivity agent)
  - kube-controller-manager, kube-scheduler, kube-apiserver
    (kubeadm clusters mein ye static pods ke roop mein yahan
    hote hain — GKE mein ye managed/hidden hote hain)

  RULE: kube-system mein APNI cheezein mat daalo kabhi bhi.
  Agar kuch galat ho gaya internal Kubernetes pod ke saath,
  cluster stability risk mein pad jaata hai.

## kube-public
  Cluster-wide publicly readable data ke liye. Unauthenticated
  users bhi is namespace ka kuch data padh sakte hain.
  Practically: ek `cluster-info` ConfigMap yahan hota hai jo
  cluster ka bootstrap info rakhta hai. Mostly unused by humans.

## kube-node-lease
  YE IMPORTANT HAI — tumne flag kiya tha "lease objects present
  hain, number = number of pods" — YE GALAT HAI. Correction:

  Number of Lease objects = NUMBER OF NODES (nahi pods)

  Har NODE ke liye ek Lease object banta hai. Ye Lease object
  kubelet ka HEARTBEAT mechanism hai:

  MECHANISM:
    Kubelet har 10 seconds mein apne node ka Lease object
    update karta hai ("main zinda hoon")
    kube-controller-manager ye Lease objects watch karta hai
    Agar koi Lease ~40 seconds tak update nahi hua →
    Node "NotReady" mark ho jaata hai
    Agar ~5 minutes tak NotReady rahe → Node ke pods evict

  LEASE KYU ALAG NAMESPACE:
    Pehle Node status directly Node object par update hota tha
    etcd par har heartbeat = full Node object write = bahut
    expensive for large clusters (1000 nodes × every 10 seconds)
    Lease ek LIGHTWEIGHT object hai (sirf timestamp)
    Alag namespace mein rakhne se unka access control alag
    rakha ja sakta hai (performance isolation bhi)

  VERIFY:
    kubectl get leases -n kube-node-lease
    # Har node ke liye ek Lease milega, same name as node
    kubectl get lease <node-name> -n kube-node-lease -o yaml
    # renewTime field har 10 seconds mein update hoga
```

---

## 3. Namespaces — Deep Architecture

### 3.1 Namespaces nested nahi ho sakte

```
YE FUNDAMENTAL LIMITATION HAI:

INVALID (ye Kubernetes mein POSSIBLE NAHI):
cluster → namespace: "company" → namespace: "team-a" (nested)

VALID (sirf flat hain):
cluster → namespace: "company-team-a"
cluster → namespace: "company-team-b"

WHY NO NESTING:
  API design decision tha — nested namespaces se ownership,
  RBAC inheritance, aur resource quota computation bahut
  complex ho jaati. Kubernetes ne intentionally flat model
  choose kiya simplicity ke liye.

GKE WORKAROUND (instructor ne mention kiya):
  GKE Autopilot clusters mein Config Controller ke through
  hierarchical namespace controller (HNC) use kar sakte ho
  — ye ek CRD-based extension hai jo "namespace as directory"
  concept implement karta hai. Lekin ye Kubernetes core nahi
  hai, ek alag addon hai.
```

### 3.2 Cluster vs Namespace — exactly kab kaun sa use karo

```
INSTRUCTOR NE BILLING BILL DIKHAYA — ye point bahut real hai:
  GKE cluster: 6000-7000 rupay ka bill in a few days ka,
  networking sirf 000 (poore cluster ka — namespace-wise
  breakdown natively nahi milta)

WHY SINGLE CLUSTER + MULTIPLE NAMESPACES (default choice):

1. NETWORK COST:
   Cluster-to-cluster cross communication = expensive
   Same cluster ke andar = free (ya negligible)
   
   Cloud mein:
   Class A (same zone, same cluster): near-zero cost
   Class B (cross-zone): kuch paisa lagta hai
   Class C (cross-region, cross-cluster): bahut zyada
   
   REAL NUMBERS: Instructor ne mention kiya infrastructure
   cost ₹5 lakh ho to networking cost ₹15 lakh tak ja sakti
   hai agar properly controlled nahi ki cross-cluster calls

2. OBSERVABILITY COST:
   Har cluster mein apna: Prometheus, Grafana, Jaeger/Tempo
   (tracing), Fluent Bit (logging), metrics agents — sab ka
   alag instance = alag maintenance, alag cost, alag upgrades

3. OPERATIONAL OVERHEAD:
   Har cluster ka alag kubeconfig, alag access control,
   alag upgrade schedule, alag certificate management

LEKIN SINGLE CLUSTER + NAMESPACES KI LIMITS SAMJHO:
  Namespace sirf LOGICAL boundary hai — physical resources
  (node ka CPU/RAM) sab share karte hain. Sirf resource quotas
  aur RBAC se isolation aata hai, physical isolation nahi.
```

### 3.3 Multi-tenancy — Complete Understanding

```
TENANCY KA MATLAB (instructor ka analogy):
  PG mein ek building mein multiple tenants = multiple
  paying occupants, each with their own space
  
  Cloud providers (AWS, GCP, Azure) khud multi-tenancy
  par build hain — tumhara AWS account ek "tenant" hai
  jo shared physical infrastructure ka ek slice le raha hai

KUBERNETES MULTI-TENANCY = ek cluster mein multiple
  independent workloads/clients/teams run karna, jahan:
  - Har tenant apne namespace mein hai
  - Ek tenant doosre ka data/config nahi dekh sakta
  - Resources isolated hain (RBAC + ResourceQuota se)

TWO MODELS:

SOFT MULTI-TENANCY (most common):
  Same company ki different teams/environments:
  - dev, staging, prod namespaces
  - Different teams ko different namespaces
  - RBAC se access control
  - Resource quotas se fair usage
  - Network policies se optional traffic isolation
  - Koi bhi data breach risk nahi kyunki sab trusted internal

HARD MULTI-TENANCY (external clients/customers):
  Different companies/customers ek cluster share karein:
  - BAHUT HARDER to implement correctly
  - Namespace isolation alone ENOUGH NAHI hai
  - Kernel-level isolation bhi chahiye (gVisor, Kata Containers)
  - Bahut strict network policies chahiye
  - Pod security standards enforce karne padte hain
  - CKA ke liye: soft multi-tenancy samjho; hard ke liye CKS

RBAC MULTI-TENANCY MEIN KAISE WORK KARTA HAI:
  Har namespace ke liye:
  - ServiceAccount banate hain (namespace-scoped identity)
  - Role banate hain (namespace mein kya kya allowed)
  - RoleBinding banate hain (kaun use kar sakta hai)
  
  Result: Team A sirf apne namespace mein kaam kar sakti hai,
  Team B ka namespace tab bhi exist karta hai but accessible nahi
```

### 3.4 Namespace kahan use NAHI karna chahiye — deep reasoning

```
INSTRUCTOR NE 4 CASES MENTION KIE:

1. APPLICATION VERSION MANAGEMENT (v1 vs v2):
   Galat: namespace: app-v1, namespace: app-v2
   
   Sahi: Deployments aur labels use karo
   kubectl create deployment app-v1 --image=app:1.0
   kubectl create deployment app-v2 --image=app:2.0
   
   WHY: Version management Deployment/ReplicaSet ka kaam hai.
   Namespaces organizational boundaries ke liye hain, deployment
   strategies ke liye nahi. Exception: Bahut bada architectural
   difference (different DB schema, completely different infra
   requirements, side-by-side metrics comparison needed) —
   tab alag namespace reasonable hai.

2. FINE-GRAINED BILLING:
   Problem: kubectl get namespaces se cost breakdown nahi milta
   Namespace-wise billing natively support nahi karta Kubernetes
   
   WORKAROUND tools: OpenCost, KubeCost, Cloudability
   Ye Prometheus metrics + cloud billing API combine karke
   per-namespace cost estimate karte hain (estimate, exact nahi)
   
   WHY NATIVELY HARD: Ek pod ka resource consumption fluctuate
   karta hai, shared node overhead ka attribution hard hai,
   network egress per-pod track karna overhead hai

3. COMPLIANCE/REGULATED WORKLOADS (GDPR, HIPAA, COPA):
   Problem: Ek namespace dusre se truly isolated nahi hai
   kernel level par. Shared kernel = potential side-channel
   attacks possible in theory.
   
   GDPR example: EU citizen ka data EU mein hi store hona chahiye
   → Alag cluster (different region) use karo
   → Namespace se geographic isolation nahi aati
   
   HIPAA: Healthcare data → separate dedicated cluster
   PCI-DSS: Payment card data → separate cluster
   
   AUDIT BENEFIT: Ek cluster ko audit karna = ek audit scope
   2 tenants = 2x audit cost potentially
   → Alag clusters ka compliance boundary clear hota hai

4. GEO-DISTRIBUTED DEPLOYMENTS:
   Problem: ns: europe, ns: asia — ye sirf logical labels hain
   Physical location ek hi node pool hai jahan tumhara cluster hai
   
   Sahi approach:
   - Alag clusters alag regions mein (GKE cluster us-central1 vs
     europe-west1 vs asia-southeast1)
   - Ya ek cluster mein alag NODE POOLS jahan nodes alag zones mein
     hain (instructor ne yehi demo kiya)
   - Phir node affinity se pods specific zone ke nodes par schedule
```

---

## 4. Linux Namespace vs Kubernetes Namespace — Exact Difference

```
Ye interview mein specifically pucha jaata hai — tumhare notes mein
flag tha "kya difference hai?"

LINUX NAMESPACE (OS-level concept):
  Operating system ki feature jaise process ko isolated view
  deti hai system resources ka.
  Types:
  - Network namespace: isolated network stack (eth0, IP, etc.)
  - PID namespace: isolated process ID tree
  - Mount namespace: isolated filesystem view
  - UTS namespace: isolated hostname
  - User namespace: isolated UID/GID mapping
  - IPC namespace: isolated inter-process communication
  
  CONTAINER IS BUILT ON LINUX NAMESPACES:
  Jab Docker/containerd ek container banata hai, har container
  ko separate Linux namespaces milte hain. Isliye ek container
  doosre container ke processes nahi dekh sakta (PID namespace
  isolation), alag IP rakh sakta hai (network namespace), etc.

KUBERNETES NAMESPACE:
  Purely a LOGICAL/API-level concept.
  Koi Linux kernel feature use nahi karta.
  Sirf etcd mein ek label hai jo resources ko group karta hai.
  kube-apiserver is label ko use karta hai access control ke liye.
  
  Pod ke containers ki actual isolation LINUX NAMESPACES se
  aati hai (kernel level) — Kubernetes namespace se nahi.

KEY RELATIONSHIP:
  Ek Kubernetes pod ke andar saare containers EK Linux network
  namespace share karte hain (isliye localhost se communicate
  karte hain — ye Lecture 2/3 ka concept yaad karo).
  
  Alag Kubernetes namespaces ke pods DEFAULT mein ALAG Linux
  network namespaces mein hain (different pods hain) lekin
  iska matlab NetworkPolicy ke bina traffic blocked nahi hota.
  
  CONFUSION POINT: Kubernetes namespace isolation = logical
  Linux namespace isolation = physical/kernel
  Dono alag cheezon ke liye same word use hoti hai.

INTERVIEW ANSWER:
  Q: "Linux namespace aur Kubernetes namespace mein kya fark hai?"
  A: "Linux namespaces are kernel-level isolation primitives that
  give processes an isolated view of system resources — network,
  PIDs, filesystem mounts, etc. They're what actually make
  containers isolated from each other at the OS level. Kubernetes
  namespaces are an API-level organizational construct stored in
  etcd — they're just labels that the API server uses for scoping
  and RBAC enforcement. A Kubernetes namespace has no kernel
  footprint at all; it doesn't create any Linux namespaces.
  Actual container isolation comes from Linux namespaces created
  by the container runtime, regardless of which Kubernetes
  namespace the pod belongs to."
```

---

## 5. CPU vs Memory — Full Technical Depth

### 5.1 Compressible vs Incompressible — exact mechanism

```
COMPRESSIBLE RESOURCE = CPU:
  
  Kya hota hai jab limit cross ho:
  → Linux CFS (Completely Fair Scheduler) THROTTLE karta hai
  → Process ko CPU cycles milenga kam — slow hoga
  → Process alive rehta hai, kill nahi hota
  → "Compressible" = tum usKE allocated time ko compress/reduce
    kar sakte ho bina process kill kiye

  GAME ANALOGY (instructor ka — good one):
  PUBG lag karna = CPU throttle. Game slow ho jaata hai
  (frame drops), par game alive rehta hai.

INCOMPRESSIBLE RESOURCE = Memory:
  
  Kya hota hai jab limit cross ho:
  → Linux KERNEL ka OOM (Out of Memory) Killer activate hota hai
  → Kernel DIRECTLY process ko SIGKILL bhejta hai
  → Process immediately terminate hota hai
  → No warning, no graceful shutdown, no SIGTERM
  → "Incompressible" = ek baar process ne memory le li, tum usse
    wapas nahi le sakte jab tak process khud free na kare ya
    kill na ho

  GAME ANALOGY:
  PUBG CRASH karna = Memory OOM Kill. Game suddenly band,
  koi advance warning nahi.

WHY THE DIFFERENCE MATTERS IN PRODUCTION:
  CPU throttle → latency badh jaati hai (SLA impact)
    → Alert aata hai slow response time ka
    → App zinda hai, kuch trace karne ke liye available
  
  Memory OOM Kill → Pod crash hota hai (availability impact)
    → Pod CrashLoopBackOff mein jaata hai
    → App dead hai, real-time investigation impossible
    → kubectl logs --previous se last instance ke logs mile
```

### 5.2 CPU Units — exact numbers

```
CPU UNITS IN KUBERNETES:

1 CPU = 1 vCPU (cloud mein) = 1 core
  = 1 AWS vCPU
  = 1 GCP vCore  
  = 1 Azure vCore

PHYSICAL CORE vs CLOUD vCPU:
  1 physical core = 2 hyper-threads = 2 vCPUs (approximate)
  Isliye: 1 physical core ≈ 2 cloud vCPUs
  
  Reverse: 1 cloud vCPU ≈ 0.5 physical core

MILLICORES (m):
  m = milli = 10^-3 = 1/1000
  1000m = 1 CPU
  500m = 0.5 CPU
  100m = 0.1 CPU
  
  Reading convention:
  "100m" = "100 millicores" = 0.1 CPU core
  "0.5" = 500m = half CPU
  "2" = 2000m = 2 full CPUs

PRACTICAL EXAMPLES:
  A simple API server: 100m-500m request typical
  A ML inference pod: 2-4 CPU request typical
  A data processing job: 1-8 CPU depending on workload

VERIFY node allocatable CPU:
  kubectl describe node <name>
  # Allocatable: cpu: 3920m
  # Means 3.92 CPUs available for pods
  # (some reserved for system processes)
```

### 5.3 Memory Units — the Mebibyte vs Megabyte trap

```
MEMORY UNITS IN KUBERNETES:

Standard (base-10 — what marketers use):
  K  = Kilobyte  = 1000 bytes
  M  = Megabyte  = 1000² bytes = 1,000,000 bytes
  G  = Gigabyte  = 1000³ bytes
  T  = Terabyte  = 1000⁴ bytes

Binary (base-2 — what computers actually use):
  Ki = Kibibyte  = 1024 bytes
  Mi = Mebibyte  = 1024² bytes = 1,048,576 bytes
  Gi = Gibibyte  = 1024³ bytes
  Ti = Tebibyte  = 1024⁴ bytes

CRITICAL DIFFERENCE:
  1 Mi (Mebibyte) = 1.048576 MB (Megabyte)
  1000 Mi = 1048.576 MB ≠ 1000 MB

WHY THIS MATTERS AT SCALE:
  Small scale: 1000 Mi vs 1000 MB = 48.576 MB difference
  Large scale: 10 million Mi allocation:
    Mi: 10,485,760 MB
    MB: 10,000,000 MB
    Difference: 485,760 MB = ~475 GB — real money!

KUBERNETES DEFAULT:
  Kubernetes mein "Mi" use karo (Mebibytes):
  memory: "512Mi"   ← correct, clear
  memory: "512M"    ← also works but use Mi for precision
  
  kubectl describe node mein bhi Mebibytes mein dikhta hai

INTERVIEW TRICK:
  Q: "Kubernetes mein 1 Mi aur 1 M mein kya fark hai?"
  A: "1Mi is a mebibyte — 1024² or 1,048,576 bytes. 1M is a
  megabyte — 10⁶ or 1,000,000 bytes. In Kubernetes resource
  specs, using Mi is more precise and maps directly to how
  the kernel and container runtime actually measure memory.
  At scale, the difference compounds significantly."
```

### 5.4 Requests vs Limits — exact mechanism + PRODUCTION RULES

```
REQUESTS:
  "Scheduler ko batao — mujhe schedule karne ke liye MINIMUM
  itna chahiye"
  
  Node scheduling mein:
    Available = Allocatable - sum(all pod requests)
    Agar naya pod ka request fit nahi hota → PENDING state
  
  GUARANTEE: Request amount pod ko ALWAYS milega (reserved)
  Container request se ZYADA use kar sakta hai (up to limit)

LIMITS:
  "Container itne se ZYADA KABHI nahi le sakta"
  
  CPU limit: Throttle (slow) — container alive
  Memory limit: OOM Kill — container dead

YAML EXAMPLE:
  resources:
    requests:
      cpu: "100m"      # Schedule ke liye minimum
      memory: "128Mi"  # Schedule ke liye minimum
    limits:
      cpu: "500m"      # Maximum CPU (throttle if exceeded)
      memory: "512Mi"  # Maximum memory (OOM kill if exceeded)

PRODUCTION RULES (instructor ne explicitly kaha — yaad karo):

RULE 1 — CPU KI LIMIT KABHI DEFINE MAT KARO:
  
  WHY: CPU compressible hai — agar limit set ki to:
  - Agar node pe CPU available hai (idle CPU) lekin limit
    define hai, container us idle CPU use NAHI kar sakta
  - Idle CPU waste ho jaata hai
  - Application unnecessarily slow hoti hai
  - YE CPU THROTTLING PROBLEM bahut real hai production mein

  Better approach: CPU ke liye sirf REQUEST define karo.
  Container apni request guarantee paayega, aur agar extra
  CPU available hai to use kar sakta hai up to node capacity.
  
  CAVEAT: Agar tum GUARANTEED QoS chahte ho (neeche explain
  kiya), to CPU limit bhi request ke equal set karni padegi.
  Trade-off hai — most cases mein no-CPU-limit better hai.

RULE 2 — MEMORY REQUEST == MEMORY LIMIT (ALWAYS equal rakhna):
  
  WHY: Memory incompressible hai:
  - Agar request < limit hai (e.g., req: 128Mi, limit: 512Mi)
  - Pod ko 128Mi ke liye schedule kiya gaya
  - Runtime par pod 512Mi tak use kar sakta hai (le sakta hai
    kiसी bhi node se)
  - Lekin schedule hone ka basis sirf 128Mi tha
  - Multiple pods aisa karein → node ki actual memory exceed
    ho sakti hai physical capacity se
  - → NODE LEVEL OOM → kernal kisi bhi process ko kill kare
    (even outside Kubernetes)
  
  Request == Limit rakhne se:
  - Scheduling exact hoti hai (exactly 512Mi reserved)
  - Runtime surprise khatam (512Mi se zyada nahi lega)
  - Ye pod automatically GUARANTEED QoS class mein jaata hai
  - OOM kill risk minimum

INTERVIEW ANSWER:
  Q: "CPU aur memory ke liye recommended limits configuration
     kya hai production mein?"
  A: "For CPU, the recommendation is to set requests but not
  limits. Since CPU is compressible, throttling is preferable
  to wasting available CPU cycles on limits. Setting no CPU
  limit allows pods to burst into available capacity. For
  memory, the recommendation is to set requests equal to
  limits. Memory is incompressible — once allocated, it can't
  be reclaimed without killing the process. Equal request and
  limit prevents over-commitment at the scheduler level,
  avoids node-level OOM situations, and automatically places
  the pod in the Guaranteed QoS class."
```

---

## 6. QoS Classes — Complete

```
QUALITY OF SERVICE = Pod ki "priority class" jab eviction decide
karna ho. Automatically assign hoti hai based on resource config.

THREE CLASSES (eviction priority worst to best):

BEST EFFORT (pehle evict hoga):
  Condition: Pod mein koi bhi resource spec nahi (na request
             na limit — for ANY container in the pod)
  
  Example:
    resources: {}   ← kuch nahi = BestEffort
  
  WHEN EVICTED: Node pe memory pressure aate hi sabse PEHLE
  PRODUCTION MEIN: KABHI BHI ye class mat rehne do
  Iska matlab: node stressed hua, tumhara pod PEHLE MARA

BURSTABLE (middle priority):
  Condition: Request aur limit dono set hain lekin EQUAL NAHI,
             YA sirf request set hai,
             YA sirf kuch containers mein set hai
  
  Examples:
    cpu request set, no limit → Burstable
    memory req: 128Mi, limit: 512Mi → Burstable
    req == limit for cpu, but no memory spec → Burstable
  
  WHEN EVICTED: BestEffort sab evict hone ke baad

GUARANTEED (last mein evict hoga):
  Condition: ALL containers (including init containers) mein
             memory aur cpu DONO ke request AND limit SET HAIN
             aur request == limit (EXACTLY EQUAL)
  
  Example:
    resources:
      requests:
        cpu: "500m"
        memory: "512Mi"
      limits:
        cpu: "500m"    ← request ke EQUAL
        memory: "512Mi" ← request ke EQUAL
  
  WHEN EVICTED: Only as absolute last resort
  
  PRACTICAL IMPLICATION:
  "Memory request = limit" rule follow karo → automatically
  Guaranteed mein aa jaate ho (if CPU request bhi set ho —
  but note: CPU limit mat set karo for most cases unless you
  specifically want Guaranteed QoS)

VERIFY QoS class:
  kubectl describe pod <name>
  # Ye field automatically set hoga:
  # QOSClass: BestEffort / Burstable / Guaranteed

EVICTION ORDER SUMMARY:
  High memory pressure → BestEffort → Burstable → Guaranteed
  (last resort)

INTERVIEW:
  Q: "QoS classes kya hoti hain aur kaise determine hoti hain?"
  A: "QoS classes are automatically assigned by Kubernetes
  based on resource configuration. BestEffort pods have no
  resource specs and are evicted first. Burstable pods have
  some resources set but request doesn't equal limit. Guaranteed
  pods have requests equal to limits for both CPU and memory
  on all containers — they're last to be evicted. Setting
  memory request equal to limit (the production best practice
  for memory) automatically achieves Guaranteed class."
```

---

## 7. ResourceQuota — Full Depth

```
WHAT: Namespace ke total resource consumption ki HARD CAP.
WHERE: Namespace-level (cluster-wide ResourceQuota NAHI hoti)
WHO CREATES: Cluster admin / platform team
ENFORCED BY: Admission controller (resource quota admission controller)

IMPORTANT CLARITY FROM TRANSCRIPT:
  Instructor ne kaha "first come first serve" — ye SIRF
  namespace ke andar pods ke beech ke baare mein hai.
  Pods aapas mein compete nahi karte namespaces ke beech.
  ResourceQuota ensure karta hai ki ek namespace apna quota
  use kare, baaki namespace ko affect nahi kare.

RESOURCE QUOTA TYPES:

1. COMPUTE RESOURCES:
  requests.cpu: "4"          # Saare pods ke CPU requests ka max
  requests.memory: 8Gi       # Saare pods ke memory requests ka max  
  limits.cpu: "8"            # Saare pods ke CPU limits ka max
  limits.memory: 16Gi        # Saare pods ke memory limits ka max

2. OBJECT COUNT:
  pods: "50"                 # Max pods in namespace
  services: "20"             # Max Services
  services.loadbalancers: "5" # Max LB-type Services (expensive!)
  secrets: "50"              # Max Secrets
  configmaps: "50"           # Max ConfigMaps
  persistentvolumeclaims: "10" # Max PVCs
  deployments.apps: "20"    # Max Deployments

3. STORAGE:
  requests.storage: "100Gi"  # Total PVC storage requested
  # YE STORAGECLASS-wise bhi specify kar sakte ho:
  fast-ssd.storageclass.storage.k8s.io/requests.storage: 50Gi

FULL YAML:
  apiVersion: v1
  kind: ResourceQuota
  metadata:
    name: production-quota
    namespace: production
  spec:
    hard:
      pods: "50"
      requests.cpu: "10"
      requests.memory: 20Gi
      limits.cpu: "20"
      limits.memory: 40Gi
      services: "20"
      services.loadbalancers: "5"
      persistentvolumeclaims: "20"
      requests.storage: 100Gi
      secrets: "100"
      configmaps: "100"

WHAT HAPPENS WHEN QUOTA EXCEEDED:
  API server REJECT karta hai request:
  Error from server (Forbidden): error when creating "pod.yaml":
  pods "my-pod" is forbidden: exceeded quota: production-quota,
  requested: requests.cpu=500m, used: requests.cpu=9.95,
  limited: requests.cpu=10

CHECK quota usage:
  kubectl describe resourcequota -n production
  kubectl get resourcequota -n production
  # Used vs Hard columns dikhenge

IMPORTANT INTERACTION WITH LIMIT RANGE:
  Agar ResourceQuota set hai aur koi pod bina resource spec ke
  create kiya jaaye:
  - LimitRange defaults inject karega (request/limit add hoga)
  - ResourceQuota us injected value ke against check karega
  - Agar quota exceeded → pod create fail
  
  Agar LimitRange nahi hai aur ResourceQuota hai:
  - Pod bina resource spec ke create hogi → FAIL
  - ResourceQuota ke saath: "must specify requests.cpu"
  - Ye enforce karta hai ki resource specs mandatory hon
```

---

## 8. LimitRange — Full Depth

```
WHAT: Per-container/per-pod INDIVIDUAL limits ka enforcement
WHERE: Namespace-level (namespace ke andar har pod par apply)
DIFFERENCE FROM RESOURCEQUOTA:
  ResourceQuota = TOTAL namespace budget
  LimitRange    = PER POD/CONTAINER cap

TWO JOBS LimitRange karta hai:

JOB 1 — DEFAULT INJECT KARNA:
  Agar container mein resource spec define nahi hai, LimitRange
  AUTOMATICALLY default values inject karta hai (like admission
  webhook karta hai)
  
  Matlab: New intern bina resource spec ke pod banaye →
  LimitRange ne automatically default laga diya → pod safe hai

JOB 2 — MIN/MAX ENFORCE KARNA:
  Container itna kam ya itna zyada specify kare to REJECT karo

FULL YAML:
  apiVersion: v1
  kind: LimitRange
  metadata:
    name: namespace-limits
    namespace: production
  spec:
    limits:
    
    - type: Container           # Per container
      default:                  # Agar limit specify nahi
        cpu: "500m"
        memory: "512Mi"
      defaultRequest:           # Agar request specify nahi
        cpu: "100m"
        memory: "128Mi"
      max:                      # Maximum allowed
        cpu: "2"
        memory: "2Gi"
      min:                      # Minimum allowed
        cpu: "50m"
        memory: "64Mi"
    
    - type: Pod                 # Per pod (all containers combined)
      max:
        cpu: "4"
        memory: "4Gi"
    
    - type: PersistentVolumeClaim  # Per PVC
      max:
        storage: "10Gi"
      min:
        storage: "1Gi"

VERIFY:
  kubectl get limitrange -n production
  kubectl describe limitrange -n production

COMBINED USE IN PRODUCTION:
  ResourceQuota + LimitRange DONO hamesha production mein lagate hain:
  
  ResourceQuota: "Is namespace ne total 10 CPU aur 20Gi
                  memory se zyada nahi use karna"
  LimitRange:    "Har pod ne 2 CPU aur 2Gi memory se zyada
                  nahi use karna, aur 50m/64Mi se kam nahi"
  
  Together:
  - LimitRange → Individual greedy pod se protection
  - ResourceQuota → Namespace-level greedy (too many pods) se protection
```

---

## 9. Taints and Tolerations — Full Depth

### 9.1 Core Concept

```
TAINT = Node par lagaaya jaata hai
"Main special purpose ke liye hoon. Bina permission ke mat aao."

TOLERATION = Pod par lagaaya jaata hai
"Mujhe us taint wale node par jaane ki permission hai"

SIMPLE MENTAL MODEL:
  Taint = "Do Not Enter" sign on a door
  Toleration = Special key/badge to enter anyway
  
  Bina badge ke → Entry nahi
  Badge ke saath → Entry POSSIBLE hai (guaranteed nahi)

IMPORTANT TRUTH (instructor ne clearly bataya):
  TOLERATION = PERMISSION (guarantee nahi)
  Toleration se sirf ye hota hai ki pod us tainted node par
  JAA SAKTA HAI. Ye guarantee NAHI karta ki pod wahan JAAYEGA.
  Pod un-tainted nodes par bhi ja sakta hai.

  Node ATTRACTION ke liye → Node Affinity use karo
  Taints/Tolerations → EXCLUSION ke liye use karo
```

### 9.2 Exact Commands and YAML

```bash
# Node par taint lagao:
kubectl taint nodes <node-name> key=value:effect

# Tumhara example (instructor ka demo):
kubectl taint nodes node-1 dal=purani:NoSchedule

# Taint remove karo (minus sign at end):
kubectl taint nodes node-1 dal=purani:NoSchedule-

# Verify taint:
kubectl describe node <node-name> | grep -A5 Taints
```

```yaml
# Pod mein toleration add karo:
spec:
  tolerations:
  - key: "dal"
    operator: "Equal"    # "Equal" ya "Exists"
    value: "purani"
    effect: "NoSchedule"
```

### 9.3 Taint Effects — 3 types

```
NoSchedule:
  - Naye pods SCHEDULE NAHI honge agar toleration nahi
  - EXISTING pods on that node AFFECTED NAHI hote
  - Use case: Node ko future scheduling se "protect" karo

PreferNoSchedule:
  - Scheduler PREFER karega nahi jaana
  - Lekin agar koi option nahi to fir bhi ja sakta hai
  - Soft version of NoSchedule

NoExecute:
  - Naye pods schedule nahi honge
  - EXISTING pods bhi EVICT ho jayenge agar unke paas
    toleration nahi hai
  - Optional: tolerationSeconds add karo — toleration
    ke bawajood kitne seconds baad evict karo
  
  Use case: Node ko maintenance pe lena — existing pods
  bhi hata do, sirf critical pods rakhne do

SYSTEM DEFAULT TOLERATIONS (every pod mein auto-added):
  # Ye tumhare notes mein lecture 5 mein dekhe the bhi
  tolerations:
  - effect: NoExecute
    key: node.kubernetes.io/not-ready
    operator: Exists
    tolerationSeconds: 300    # 5 min wait before eviction
  - effect: NoExecute
    key: node.kubernetes.io/unreachable
    operator: Exists
    tolerationSeconds: 300
  
  # Ye isliye: agar node temporarily unreachable ho (network
  # blip, kubelet restart), pods immediately evict na hon
```

### 9.4 Real Production Use Cases

```
USE CASE 1 — GPU Nodes reserve karna:
  kubectl taint nodes gpu-node-1 accelerator=gpu:NoSchedule
  
  Sirf ML pods (with toleration) GPU nodes par jaayenge.
  Regular web server pods accidentally GPU node par nahi
  jayenge (aur GPU node waste nahi karega web traffic par).

USE CASE 2 — Node maintenance:
  kubectl taint nodes node-1 maintenance=true:NoExecute
  
  Saare pods evict ho jaate hain (NoExecute).
  Ek baar maintenance complete → taint hata do:
  kubectl taint nodes node-1 maintenance=true:NoExecute-

USE CASE 3 — Control plane isolation:
  kubectl describe node <control-plane-node>
  # Automatically ye taint lagta hai kubeadm se:
  Taints: node-role.kubernetes.io/control-plane:NoSchedule
  
  Isliye normal application pods control plane par
  accidentally schedule nahi hote.

USE CASE 4 — Cluster-wide deployment block:
  Instructor ne demonstrate kiya — sab nodes par taint:
  kubectl taint nodes --all hardened=true:NoSchedule
  
  Ab koi naya pod schedule nahi hoga jab tak explicitly
  toleration nahi rakha (production freeze scenario).
```

---

## 10. Node Placement Mechanisms — Complete Comparison

### 10.1 NodeName — Scheduler Bypass

```
WHAT: Pod spec mein directly node ka naam specify karo
EFFECT: kube-scheduler completely BYPASS ho jaata hai

spec:
  nodeName: gke-cluster-node-abc0d   # Direct node name
  containers:
  - name: myapp
    image: nginx

WHEN TO USE:
  - Almost NEVER in production
  - Debug/testing purposes only
  - CKA exam: ek specific question ho sakta hai "without using
    scheduler, deploy pod on specific node" → nodeName

RISKS:
  - Node down ho jaaye → pod Pending rehega indefinitely
    (scheduler nahi hai jo automatically redistribute kare)
  - RBAC ya admission controllers ki scheduler-based checks
    bypass ho sakti hain (security concern)
  - Node labels, taints, resource availability — kuch check
    nahi hota

INTERVIEW: "nodeName is the only way to completely bypass
the Kubernetes scheduler — the API server assigns the pod
directly to that node without any scheduling algorithm."
```

### 10.2 NodeSelector — Simple label matching

```
WHAT: Pod ke liye matching labels wale nodes sirf eligible hain

spec:
  nodeSelector:
    disktype: ssd          # Sirf "disktype=ssd" wale nodes
    accelerator: gpu       # AND "accelerator=gpu" wale nodes
  containers: ...

BUILT-IN NODE LABELS (automatically available):
  kubernetes.io/hostname: node-1
  kubernetes.io/arch: amd64
  kubernetes.io/os: linux
  node.kubernetes.io/instance-type: n1-standard-4   # cloud
  topology.kubernetes.io/region: us-central1        # cloud
  topology.kubernetes.io/zone: us-central1-a        # cloud

CUSTOM NODE LABELS:
  kubectl label nodes node-1 disktype=ssd
  kubectl label nodes node-2 accelerator=gpu

LIMITATION vs Node Affinity:
  NodeSelector = simple equality only (key=value)
  Node Affinity = complex expressions (In, NotIn, Exists, etc.)
  
  NodeSelector deprecated nahi hua, lekin Node Affinity
  zyada powerful hai — new workloads mein Node Affinity prefer
  karo.
```

### 10.3 Node Affinity — Full Power

```
TWO MODES:

requiredDuringSchedulingIgnoredDuringExecution:
  HARD CONSTRAINT — MUST satisfy hona chahiye
  Satisfy nahi → Pod PENDING (indefinitely)
  "IgnoredDuringExecution": Pod already running hai aur rule
  change ho jaye → Pod EVICT NAHI HOTA (ye alag hai)

preferredDuringSchedulingIgnoredDuringExecution:
  SOFT CONSTRAINT — try karo but schedule anyway
  Satisfy nahi bhi hua → Pod SCHEDULE HO JAATA HAI
  weighted score → high weight = more influence

OPERATORS:
  In       → key ka value in [list]
  NotIn    → key ka value NOT in [list]
  Exists   → key exist karta hai (value check nahi)
  DoesNotExist → key exist nahi karta
  Gt       → Greater than (numeric string comparison)
  Lt       → Less than

FULL YAML:
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: topology.kubernetes.io/zone
            operator: In
            values:
            - us-central1-a
            - us-central1-b
          - key: node.kubernetes.io/instance-type
            operator: In
            values:
            - n1-highmem-8
            - n1-highmem-16
      
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 80          # Higher = more preferred
        preference:
          matchExpressions:
          - key: disktype
            operator: In
            values:
            - ssd
      - weight: 20
        preference:
          matchExpressions:
          - key: node-generation
            operator: In
            values:
            - gen4

TAINT + NODE AFFINITY = COMPLETE ISOLATION:
  Taint → Other pods ko EXCLUDE karo from node
  NodeAffinity → Your pod ko ATTRACT karo to that node
  
  Together: True isolation
  Only taint → Other pods blocked, but your pod might go
               somewhere else too
  Only affinity → Your pod goes there, others can too
```

### 10.4 Pod Affinity and Anti-Affinity

```
CONTEXT: Node affinity = node ke labels ke basis par
         Pod affinity = EXISTING pods ke basis par

POD AFFINITY = "Uss node par schedule karo jahan YE pod
               already chal raha hai"

Use case: API server + Redis cache — inhe SAME NODE par
rakhna chahte ho latency kam karne ke liye:

spec:
  affinity:
    podAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
      - labelSelector:
          matchExpressions:
          - key: app
            operator: In
            values:
            - redis-cache
        topologyKey: kubernetes.io/hostname
        # ↑ "Same node" — hostname level pe co-locate

POD ANTI-AFFINITY = "Uss node par mat schedule karo jahan
                    YE pod already chal raha hai"

Use case: 3 replicas of web-app — teeno ALAG ALAG nodes par
chahiye (ek node down → sabkuch nahi girna chahiye):

spec:
  affinity:
    podAntiAffinity:
      preferredDuringSchedulingIgnoredDuringExecution:
      - weight: 100
        podAffinityTerm:
          labelSelector:
            matchLabels:
              app: web-app
          topologyKey: kubernetes.io/hostname

TOPOLOGY KEY:
  kubernetes.io/hostname → Same/different NODE level
  topology.kubernetes.io/zone → Same/different ZONE level
  topology.kubernetes.io/region → Same/different REGION level
  
  Ye key decide karta hai "same" ka matlab kya hai

PRODUCTION RECOMMENDATION 2026:
  Pod spreading ke liye Anti-Affinity se BETTER hai
  TopologySpreadConstraints (neeche cover kiya) — zyada
  expressive aur performant hai. Anti-affinity sirf tab use
  karo jab specific pod-to-pod co-location/exclusion chahiye.
```

---

## 11. TopologySpreadConstraints — Deep Explanation

```
WHAT: Pods ko multiple "failure domains" (zones, nodes) mein
      EVENLY distribute karo — anti-affinity se better aur
      more expressive.

PROBLEM ANTI-AFFINITY SOLVES NAHI KARTA:
  Anti-affinity: "No two same-label pods on same node" →
  Simple binary rule — ek replica per node.
  Lekin kya hoga agar 5 replicas, 3 nodes hain?
  Anti-affinity: 2 pods Pending (no eligible nodes)
  TopologySpreadConstraints: 2-2-1 distribution, sab schedule

FULL YAML:
spec:
  topologySpreadConstraints:
  - maxSkew: 1              # Maximum imbalance allowed
    topologyKey: topology.kubernetes.io/zone
    # ↑ Kis dimension mein spread karna hai
    whenUnsatisfiable: DoNotSchedule
    # Options: DoNotSchedule (hard) | ScheduleAnyway (soft)
    labelSelector:
      matchLabels:
        app: my-app         # Konse pods count karne hain
  
  - maxSkew: 1
    topologyKey: kubernetes.io/hostname
    whenUnsatisfiable: ScheduleAnyway

MAXSKEW EXPLAINED:
  maxSkew = 1 ka matlab:
  Agar Zone A mein 3 pods hain, Zone B mein kam se kam 2
  hone chahiye (max difference = 1)
  
  maxSkew = 2:
  Zone A: 4, Zone B: minimum 2 allowed
  
  Smaller maxSkew = stricter balancing

BUILT-IN TOPOLOGY LABELS (cloud providers auto-add these):
  kubernetes.io/hostname        → Node level
  topology.kubernetes.io/zone  → Availability zone (us-central1-a)
  topology.kubernetes.io/region → Region (us-central1)

WHEN TO USE:
  - High availability: 3+ zone mein spread karo
  - Node-level HA: Sab replicas same node par nahi
  - Cost optimization: Nodes evenly utilize karo
  
  BEST PRACTICE 2026: Anti-Affinity ke bajaye TopologySpread
  use karo pod spreading ke liye — Kubernetes 1.27+
  mein TopologySpreadConstraints zyada performant aur expressive
  hai.

EXAM COMMAND:
  kubectl explain pod.spec.topologySpreadConstraints
  # Live documentation terminal mein
```

---

## 12. OpenCost — Cost Optimization Tool

```
INSTRUCTOR NE LIVE DEMO DIYA — concept samjho:

PROBLEM: kubectl default mein per-namespace billing nahi dikhata
  Cloud provider ka bill = cluster-level total
  "Kaun sa namespace kitna kharcha kar raha hai?" — natively nahi

OPENCOST:
  Open source, CNCF sandbox project
  Per-namespace, per-pod, per-deployment COST BREAKDOWN
  
  HOW IT WORKS:
  1. Cluster mein deploy hota hai (operator pattern — Lecture 2 yaad karo)
  2. Prometheus metrics collect karta hai (CPU/memory usage per pod)
  3. Cloud billing API se instance pricing fetch karta hai
  4. Math karta hai: "Ye pod 200m CPU use kar raha hai, instance
     ki cost $X/hour hai, isliye ye pod $Y/hour ka hai"
  5. Namespace rollup: saare pods ki cost add karke namespace cost
  
  DASHBOARD: Web UI se visualization
  RECOMMENDATION: "Ye pod 100m allocate kiya gaya lekin sirf
  20m use karta hai — over-provisioned hai"
  
  ALTERNATIVES: KubeCost (commercial, more features), Cloudability

WHY NAMESPACES 60% PLAY KARTE HAIN COST OPTIMIZATION MEIN:
  - Namespace-level ResourceQuota → waste prevent karta hai
  - Per-team/env cost visibility possible hoti hai
  - Chargeback model: Team A ne itna kharch kiya, Team B ne itna
  - Over-provisioned resources identify hote hain namespace level par
```

---

## 13. 50 Things Update

```
113. kube-node-lease mein Lease objects = NUMBER OF NODES
     (pods nahi) — har node ka heartbeat mechanism
114. Namespace nested nahi ho sakte — flat hierarchy only
     (HNC ek addon hai, Kubernetes core nahi)
115. Linux namespace (kernel isolation) ≠ Kubernetes namespace
     (API-level label) — ye sirf same word hain different concepts ke
116. CPU ki limit production mein kabhi define mat karo
     (idle CPU waste hoga)
117. Memory request == Memory limit hamesha rakho
     (incompressible, OOM kill prevent karo)
118. Guaranteed QoS = CPU + Memory dono ke req == limit
     (LAST to be evicted)
119. Taint = Node par "Do Not Enter" sign
     Toleration = Permission to enter (NOT guarantee)
120. Taint effects: NoSchedule (future only) | PreferNoSchedule
     (soft) | NoExecute (evict existing too)
121. nodeName = scheduler bypass karta hai (NEVER in production)
122. nodeSelector = simple label match (equality only)
     nodeAffinity = complex expressions (In, NotIn, Exists, etc.)
123. requiredDuring = hard (Pending if fails) |
     preferredDuring = soft (schedules anyway)
124. "IgnoredDuringExecution" = running pod evict nahi hoga
     rule change hone par
125. Pod affinity = "Co-locate with pods having this label"
     Pod anti-affinity = "Don't go where these pods are"
126. topologyKey determines scope: hostname=node, zone=AZ level
127. TopologySpreadConstraints > Anti-affinity for pod spreading
     in 2026 (more expressive, performant)
128. maxSkew = maximum imbalance allowed across topology domains
129. OpenCost = per-namespace cost visibility tool (CNCF project)
130. Namespace compliance cases: separate cluster use karo
     (GDPR, HIPAA, PCI-DSS)
```



---
#  Taints-Tolerations, Affinity, QOS, OpenCost

## Hierarchy
- Cluster then namespaces, nodes then service then deployment then replica set then endpoint slice then pods then containers, containers run in linux and in linux control groups are used to maintain memory allocated.
- namespaces can not be nested
- when creating a cluster, some namespaces are created by default:
  - default: Here your all the deployment resides if specific namespace is not specified
  - kube-system:?
  - kube-node-lease: ?, lease objects are present in it, number of lease objects = number of pods 
  - kube-public:?
 
- To check what all resources can be put in a namespace, kubectl api-resources --namespaced=true, this command lists all the resources which can reside in a namespace
- what are kubernetes namespace and linux namespace? differences between them?
- why don't we create different clusters instead of creating namespace because namespace are logical separation only? We don't create multiple clusters for namespace usage because cluster to cluster communication increases networking costs a lot, higher than infra itself sometimes if cluster if big.
- Multiple clusters mean multiple observability tools because each cluster have some observability tools running, maintainance cost.
- Multi tenancy: ? (explain it in deep, most important), multiple systems can run in a cluster keeping their communication and resources isolated using namespace via RBAC. 

### where namespaces should not be used and downsides of namespaces
- fine grained billing
- different versions of your applications unless you have a very large change, you are going to have a track both versions of your application and collecting some metrics out of that.
- compliance and assured workloads: ?
- Geo-distributed deployment:?

### CPU vs Memory
- CPU is Compressible, can take it away where as memory is non-compressible, can not take it away. ??
- In CPU: 1 core = 1 AWS vCPU = 1 GCP vCore = 1 Azure vCore = milliCPU/core
- In Memory: Mi (Mebi bytes), 1 Mebibyte is not equal to 1 Megabyte
- limits and requests in yaml of pod while defines how much memory and cpu is to be assigned to this pod. which one should be bigger or smaller or whatever.
- we can define only cpu limits and requests, we can also define only memory limits and requests, all depends on us.
- when process asks more memory than limit then kernel kills it
- CPU ki limit define nahi karte and memory ke liye request and limit should be equal.

## QOS (Quality of Service)
- best effort:?
- burstable:?
- guaranteed:?

---

## resource quota: applied at namespace level, cpu, memory, resource count(number of deployments, services, pods,...), pv, pvc can be defined at namespace level, preventing how many and much resources are allowed, not more than that.
- resource quota is not applied on cluster wide only applied at namespace level.
- hard and soft limits:?
- In namespace resource don't compete for cpu and memory, it is based on first come first server.

 ## limit ranges:?
 - At namespace level, things are being limited via resource quota but inside a namespace there one pod may take more memory or CPU and other might not sufficient amount of CPU and memory, so it is limited by limit ranges. limit range saves pods race condition. it by default applies defined limits on each pod created.
- limits can also be applied by using admission controller

---

## Taints vs Tolerations:?
- taints node pe apply hote h and tolerations pods pe. for example if taints on a node is dal=purani then on this node only those pod would be created which have tolerations equal to dal=purani, if a pod does not have or different toleration then it would not be scheduled on any of the pod.
- for example one node has a taint as dal=purani and a pod is create with dal=purani toleration then it is not guaranteed that it will got to that particular because on which node that new pod will be scheduled is decided by scheduler, so it can be scheduled on a different node if either that node has same taints or no taints. but wait pods are present inside node?. what is this mean...

- without using scheduler a pod can be deployed on a specific node by putting nodename in yaml file. if nodeName is defined in yaml then that specific pod will be scheduled on that specific node and in this why scheduler will not be used or skipped.
-  

## Affinity vs Anti-Affinity, node affinity, pod affinity 
- Depending on previous pods, placement of new pods can be defined.

## Topology Spread Constraint





