# Lecture 0.5 — Local Cluster Setup (KinD) & kubectl Fundamentals
(Hands-on practice notes — yeh "lecture" nahi hai, but exam-critical
muscle memory yahan se banti hai)

---

## A) KinD — Local Multi-Node Cluster Setup

### A.1 Single node vs multi-node — confirmed correct

`kind create cluster --name node_name` ek SINGLE Docker container
banata hai jo CONTROL PLANE aur WORKER dono ka kaam karta hai (single
node cluster). `--name` cluster ka naam set karta hai, individual node
ka nahi — Gemini ne ye sahi bataya tha.

### A.2 Multi-node config file — confirmed correct, command:

```yaml
# kind-config.yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
  - role: control-plane
  - role: worker
  - role: worker
```

```bash
kind create cluster --name multi-node-cluster --config kind-config.yaml
kubectl get nodes   # verify
```

### A.3 MISSING from Gemini's answer — why KinD matters for CKA specifically

```
KinD (Kubernetes IN Docker) practice ke liye best hai kyuki:
- Real kubeadm-based cluster jaisa behavior deta hai (static pods,
  multi-node, etc. — jo humne Lecture 2 mein discuss kiya)
- Minikube se zyada CKA-realistic hai kyuki Minikube zyada abstraction
  deta hai (single VM mein sab kuch chhupa deta hai)
- FREE hai, laptop par chal jata hai, cloud cost nahi lagti practice ke liye

CKA exam ACTUAL exam environment mein cloud-provisioned clusters milte
hain (tumhe khud cluster banana nahi padta exam mein) — isliye KinD
sirf PRACTICE ke liye hai, exam mein nahi use hoga. Lekin static pod
manifests, kubeadm upgrade jaisi cheezein practice karne ke liye KinD
sabse accessible tool hai.
```

### A.4 Deleting clusters — organized reference (Gemini's answer, cleaned up)

```bash
# KinD
kind delete cluster --name <cluster-name>

# Minikube
minikube delete

# kubeadm-based (on-prem/VM clusters) — run on EVERY node:
sudo kubeadm reset

# Managed cloud (won't appear on exam, but good to know for job):
eksctl delete cluster --name <name> --region <region>      # AWS
gcloud container clusters delete <name> --zone <zone>        # GCP
az aks delete --name <name> --resource-group <rg>             # Azure
```

---

## B) kubectl Fundamentals — Command Anatomy

### B.1 The mental model (Gemini's framing is good, keep it):

```
kubectl → ACTION → RESOURCE → FLAGS
kubectl    get   →  pods   → -o wide
```

### B.2 Core commands — organized and corrected

**`kubectl get`** — list resources
```bash
kubectl get nodes
kubectl get pods -o wide          # extra columns: IP, node, etc.
kubectl get pods -A                # ALL namespaces (CRITICAL flag, see section D)
kubectl get pods -n kube-system    # specific namespace
```

**`kubectl run`** — quick single-pod creation (imperative, testing only)
```bash
kubectl run web-test --image=nginx
```
⚠️ Correction to Gemini's note: `kubectl run` ONLY creates a bare Pod
— NOT a Deployment. Production mein bare Pods kabhi use nahi karte
(no self-healing — agar crash ho jaye, koi controller use restart
nahi karega, kyuki ReplicaSet controller hi nahi hai bare Pod ke peeche).
`kubectl run` sirf QUICK TESTING/DEBUGGING ke liye hai.

**`kubectl logs`**
```bash
kubectl logs web-test           # one-time dump
kubectl logs web-test -f        # follow/stream live
kubectl logs web-test --previous # CRASHED container ka last log (Lecture 1 mein cover kiya tha)
kubectl logs web-test -c <container-name>  # multi-container pod mein specific container
```

**`kubectl delete`**
```bash
kubectl delete pod web-test
kubectl delete pod web-test --force --grace-period=0   # immediate force delete (stuck pods ke liye)
```

---

## C) `kubectl apply` — CORRECTED and DEEPENED (Gemini's explanation had real gaps)

Gemini ne three-way merge ka CONCEPT sahi bataya, lekin MECHANISM
(kaise track karta hai) aur ek IMPORTANT GOTCHA missing tha. Yahan
full corrected version hai:

### C.1 What apply actually compares (Gemini said this, confirmed correct)

```
kubectl apply -f 11.yaml karne par 3 cheezein compare hoti hain:

1. LOCAL FILE (11.yaml) — tum abhi kya bhej rahe ho
2. LIVE STATE — cluster mein abhi kya chal raha hai
3. LAST-APPLIED-CONFIGURATION — pichli baar jo apply kiya tha, uska snapshot
```

### C.2 MISSING — the actual mechanism (HOW #3 is stored — Gemini skipped this)

```
Last-applied-configuration kahan store hoti hai — EXACT mechanism:

Jab tum pehli baar `kubectl apply -f pod.yaml` chalate ho, kubectl
TUMHARI PURI YAML FILE ko JSON mein convert karke ek ANNOTATION ke
roop mein resource ke andar hi save kar deta hai:

metadata:
  annotations:
    kubectl.kubernetes.io/last-applied-configuration: |
      {"apiVersion":"v1","kind":"Pod","metadata":{...}...}

Dekhne ka command:
kubectl get pod <name> -o jsonpath='{.metadata.annotations.kubectl\.kubernetes\.io/last-applied-configuration}'

Agli baar jab tum apply karte ho, kubectl is annotation ko PADHTA hai
aur naye file se COMPARE karta hai — isi se pata chalta hai ki KAUN
SA field tumne explicitly REMOVE kiya hai (jo sirf live-state aur
new-file compare karne se pata nahi chalta).
```

### C.3 ⚠️ CRITICAL GOTCHA — Gemini's answer didn't mention this, EXAM-RELEVANT

```
BIGGEST GOTCHA: `kubectl create` se banaya gaya resource agar baad
mein `kubectl apply` se update karne ki koshish karo, to PROBLEM hoti
hai — kyuki `create` ye annotation SET NAHI karta. Toh agli baar jab
`apply` chalate ho, kubectl ko "last applied" pata nahi hota, aur
merge BEHAVE UNEXPECTEDLY kar sakta hai.

GOLDEN RULE (yaad karo): Resource ko EK HI tarike se manage karo
HAMESHA — ya to `create` ya `apply`, dono mix mat karo. CKA/production
best practice: SHURU SE `apply` use karo.

Agar already `create` se bana hua resource hai aur ab `apply` se
manage karna hai:
kubectl create --save-config -f deploy.yaml   # future mein 'apply' chalega
# YA agar already create ho chuka hai bina --save-config:
kubectl replace --save-config -f deploy.yaml
```

### C.4 The 3 outcomes — confirmed correct, kept as-is

```bash
kubectl apply -f 11.yaml
# Resource exists nahi karta  → "pod/amit-pod created"
# Resource exists, kuch change hai → "pod/amit-pod configured"
# Resource exists, kuch change nahi → "pod/amit-pod unchanged"
```

### C.5 create vs apply — table (Gemini's table, confirmed accurate)

| | `kubectl create -f` | `kubectl apply -f` |
|---|---|---|
| Philosophy | Imperative | Declarative |
| Resource already exists? | FAILS with error | Updates smoothly |
| Best for | One-time test/setup | CI/CD, GitOps, repeated updates |

### C.6 Interview-ready answer (yaad karo)

```
Q: "Explain what happens internally when you run kubectl apply."

A: "kubectl apply performs a three-way merge between three sources:
the local manifest file, the live state of the resource in the
cluster, and a snapshot of the previously-applied configuration stored
in the kubectl.kubernetes.io/last-applied-configuration annotation.
This three-way comparison — rather than a simple two-way diff between
local file and live state — is what lets Kubernetes correctly detect
when a field has been deliberately REMOVED from the manifest, not
just added or changed. It's why apply is idempotent and safe to run
repeatedly, which makes it the standard approach for GitOps and CI/CD
pipelines. Since Kubernetes 1.22, server-side apply is also available,
which moves this merge computation to the API server itself instead
of the local kubectl client, giving better conflict detection when
multiple actors manage the same resource."
```

---

## D) Namespace vs Cluster Hierarchy — CORRECTED (Gemini's framing had one real error)

### D.1 The error to fix

Gemini's answer said *"Nodes exist [at Cluster level]... Namespaces exist here [also Cluster level]"* — phrased like Nodes and Namespaces are at the same conceptual level. **This is misleading.** The precise correct framing:

```
## Correct hierarchy — physical vs logical, kept STRICTLY separate

PHYSICAL/INFRASTRUCTURE LAYER (no namespace concept applies here AT ALL):
Cluster
  └── Nodes (control-plane, worker)

LOGICAL/API LAYER (namespace concept applies ONLY here):
Cluster-scoped resources (NO namespace): Nodes (as API objects),
  Namespaces themselves, PersistentVolumes, ClusterRoles,
  ClusterRoleBindings, StorageClasses
Namespace-scoped resources (LIVE INSIDE a namespace): Pods,
  Deployments, Services, ConfigMaps, Secrets, PVCs, Roles, RoleBindings

GEMINI KA ERROR: "Nodes exist [at cluster level], Namespaces exist
[at cluster level]" likhne se lagta hai jaise dono SAME type ki cheez
hain jo cluster ke andar baithi hain. REALITY: Node ek PHYSICAL/
INFRASTRUCTURE object hai (real machine), Namespace ek LOGICAL
PARTITIONING mechanism hai (sirf API-level concept, koi physical
existence nahi). Dono "non-namespaced" honay ka matlab sirf itna hai
ki API level par dono "cluster-scoped" categorized hain — lekin
conceptually woh bahut different cheezein hain. Ye distinction
interview mein clearly bolna important hai, warna lagta hai aapne
sirf surface-level samjha hai.

Correct one-liner: "A Namespace is purely a logical partition within
the API — it has no physical footprint. A Node is physical infra. The
fact that both happen to be 'cluster-scoped' (not namespaced) in
kubectl's API categorization is incidental, not because they're the
same kind of thing."
```

### D.2 The single most important command — Gemini buried this, surfacing it properly

```bash
kubectl api-resources
```
```
Iske 'NAMESPACED' column mein:
true  → resource namespace ke andar rehta hai (Pods, Services, Deployments)
false → resource cluster-wide hai (Nodes, Namespaces, PV, ClusterRole)

YE COMMAND YAAD KARO — agar exam mein confusion ho ki "ye resource
namespaced hai ya nahi," seedha ye command chalao, guess mat karo.
```

### D.3 The warehouse analogy — keeping Gemini's, it's genuinely good

```
Warehouse = Cluster
Floor grids = Nodes (physical)
Security fences = Namespaces (logical, NOT physical)
Boxes = Pods (logically fenced by namespace, PHYSICALLY sitting on a Node)

Key realization: Ek Pod EK namespace mein "belongs" karta hai (logical)
lekin EK specific Node par "runs" karta hai (physical). Dono ALAG
dimensions hain — ek Pod ka namespace uske node se KOI relation nahi
rakhta. Same namespace ke Pods different nodes par chal sakte hain,
aur same node par different namespaces ke Pods chal sakte hain.
```

---

## E) Debugging Workflow — Real Example You Hit, Organized as a Template

You actually hit a real `ErrImagePull` scenario — this is valuable because it's *your own* debugging experience, not theory. Organizing it as a reusable template:

```
## ErrImagePull / ImagePullBackOff — debugging template (tumne ye khud face kiya)

SYMPTOM: kubectl get pods → STATUS shows ErrImagePull ya ImagePullBackOff,
READY shows 0/1

STEP 1: kubectl describe pod <name>
        → Events section mein scroll karo, EXACT reason dikhega

COMMON ROOT CAUSES (sabko yaad karo, exam mein variations aate hain):
1. Image name mein typo (nginxx instead of nginx)
2. Image tag exist nahi karta (nginx:9.99 jaisa invalid tag)
3. Private registry — authentication missing
   FIX: imagePullSecrets field add karna padega pod spec mein
4. Network/DNS issue — node registry tak reach nahi kar pa raha
   FIX: node se manually test karo: crictl pull <image-name>

STEP 2: Bare pod ka FIX — tumhara note sahi tha:
kubectl delete pod amit1
kubectl run amit1 --image=nginx

⚠️ IMPORTANT ADDITION (Gemini ne mention nahi kiya): Agar ye Pod kisi
Deployment ka part hota (na ki bare pod), to delete karne ki zarurat
NAHI hoti — sirf image field ko fix karna hota:
kubectl set image deployment/<name> <container>=<correct-image>
Deployment khud purane bad pod ko replace kar dega naye se. Bare pods
mein hi delete+recreate karna padta hai kyuki koi controller unko
manage nahi karta.

Interview answer: "ErrImagePull almost always traces back to either
a typo in the image reference, a missing tag, or missing registry
credentials via imagePullSecrets. The fastest diagnostic is always
kubectl describe — the Events section gives the exact underlying
error from the container runtime."
```

---

## F) Master Reference Table — All Commands From This Session, Organized by Use Case

```bash
# ===== CLUSTER SETUP (local practice only, not exam) =====
kind create cluster --name <name> --config <file.yaml>
kind get clusters
kind delete cluster --name <name>

# ===== CONTEXT & CLUSTER INFO =====
kubectl cluster-info
kubectl config get-contexts
kubectl config use-context <context-name>     # EXAM: always run this FIRST per question
kubectl config current-context

# ===== VIEWING RESOURCES =====
kubectl get <resource>
kubectl get <resource> -A                     # all namespaces — use when "missing" resources
kubectl get <resource> -n <namespace>
kubectl get <resource> -o wide                # extra columns
kubectl get <resource> -o yaml                # full YAML
kubectl api-resources                         # shows NAMESPACED true/false for every resource type

# ===== IMPERATIVE CREATION (exam time-savers) =====
kubectl run <pod-name> --image=<image>
kubectl create deployment <name> --image=<image> --replicas=3
kubectl create namespace <name>
kubectl expose deployment <name> --port=80 --target-port=8080 --type=NodePort

# ===== DRY-RUN PATTERN (use CONSTANTLY in exam) =====
kubectl create deployment web --image=nginx --dry-run=client -o yaml > deploy.yaml
kubectl run pod1 --image=nginx --dry-run=client -o yaml > pod.yaml

# ===== DECLARATIVE MANAGEMENT =====
kubectl apply -f <file.yaml>
kubectl create --save-config -f <file.yaml>   # if you must use create but want apply later

# ===== DEBUGGING TRIPLET =====
kubectl describe <resource> <name>
kubectl logs <pod> [-f] [--previous] [-c <container>]
kubectl exec -it <pod> -- /bin/sh

# ===== DELETION =====
kubectl delete pod <name>
kubectl delete pod <name> --force --grace-period=0   # immediate, for stuck pods

# ===== NODE MAINTENANCE (Cluster Architecture domain — 25% weight) =====
kubectl cordon <node>                          # block new scheduling only
kubectl drain <node> --ignore-daemonsets --delete-emptydir-data
kubectl uncordon <node>

# ===== KUBEADM (runs on host shell, NOT through kubectl) =====
kubeadm upgrade plan
kubeadm upgrade apply v1.34.0
sudo kubeadm reset

# ===== RBAC CHECKS =====
kubectl auth can-i create deployments --as=<user>
kubectl auth can-i create deployments --as=system:serviceaccount:<ns>:<sa-name>

# ===== METRICS (needs Metrics Server installed) =====
kubectl top nodes
kubectl top pods -A

# ===== EXAM-DAY ALIASES (set these FIRST, every exam attempt) =====
alias k=kubectl
export do="--dry-run=client -o yaml"
export now="--force --grace-period=0"
# usage: k create deploy nginx --image=nginx $do
# usage: k delete pod stuck-pod $now
```
