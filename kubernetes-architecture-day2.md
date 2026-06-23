# Lecture 2 — Phase 1: kube-apiserver, Admission Control, Request Lifecycle

## 1. Fixing the master-node-down question first

Your note says: *"if master node get down... application would run because application is deployed on worker node and master node is for management only."* This is **directionally correct but incomplete**, and an interviewer will push on it.

**The precise truth:**
- Already-running Pods keep running. `kubelet` on the worker node doesn't need the control plane to keep an already-scheduled container alive — it already has the Pod spec cached locally.
- What **breaks immediately**: anything requiring a *new decision* — no new Pods get scheduled, no `kubectl` commands work, no scaling, no self-healing if a Pod actually crashes (because `kube-controller-manager` can't detect/react — it's down too), no Service endpoint updates if Pods change.
- So the honest answer is: **"Existing traffic to existing healthy pods keeps flowing. But the cluster is flying blind — no self-healing, no new deployments, no scaling, until the control plane comes back."**

This nuance ("flying blind" vs "completely down") is exactly the kind of answer that signals seniority in an interview.
---
Kubelet ko bilkul aasan bhasha (Hinglish) mein samajhte hain, taaki CKA/CKAD exam ke liye tumhara concept ekdum crystal clear ho jaye.

Kubelet ko tum cluster ka **"Galli ka Captain"** ya worker node ka **"Manager"** samajh sakte ho. Kubelet ko is baat se koi matlab nahi hota ki baaki nodes par kya chal raha hai, ya cluster ka global network kaisa hai. Uska sirf ek hi kaam hai: **Apne khud ke node ko sambhalna aur Control Plane se mile orders ko execute karna.**

Ek bohot important baat jo interview aur exams mein puchi jaati hai: Kubelet baki Kubernetes components (jaise API Server, Scheduler) ki tarah container ke andar nahi chalta. Ye worker node ke operating system par directly ek **systemd background service (daemon)** ki tarah run karta hai.

---

### Kubelet Asli Mein Karta Kya Hai? (Core Responsibilities)

#### 1. Pods ko Create aur Manage Karna (The Sync Loop)

Kubelet ke andar ek infinite loop chalta rehta hai jise **Sync Loop** kehte hain. Is loop mein kubelet do cheezein lagatar check karta hai:

* **Desired State:** API Server kya chahta hai? (Matlab, master node ne is node ko kaun sa Pod run karne ko bola hai).
* **Actual State:** Node par abhi real mein kya chal raha hai? (Kubelet local container runtime se poochta hai).

Agar dono state match nahi karti—for example, API Server chahta hai ki `nginx` container chale, par container crash ho gaya hai—to kubelet turant action leta hai aur use restart karta hai.

#### 2. Drivers Se Kaam Karwana (CRI and CNI)

Kubelet khud container nahi banata. Wo ek manager ki tarah niche diye gaye tools se kaam karwata hai:

* **CRI (Container Runtime Interface - e.g., containerd):** Kubelet containerd ko order deta hai: *"Bhai, Nginx ka image pull karo aur ek container start karo."* Kubelet sabse pehle ek "Pod Sandbox" ya network namespace banata hai jahan saare containers sath mein reh sakein.
* **CNI (Container Network Interface - e.g., Calico/Flannel):** Kubelet CNI plugin ko bolta hai ki is naye bane Pod ko ek unique Cluster IP address assign kare.

#### 3. Probes Check Karna (Health Monitoring)

Jab containers chalne lagte hain, to kubelet unka physical instructor ban jata hai. Tum jo YAML file mein Probes likhte ho, unhe kubelet hi execute karta hai:

* **Liveness Probe:** Kubelet baar-baar container se poochta hai, *"Tu zinda hai?"* Agar response nahi aata ya fail hota hai, to **kubelet container ko kill karke dubara restart kar deta hai**. isme master node ka koi role nahi hota, sab kuch kubelet local level par handle karta hai.
* **Readiness Probe:** Kubelet check karta hai ki kya container traffic lene ke liye ready hai? Agar nahi, to wo API Server ko bol deta hai ki is Pod par baki traffic mat bhejo.

#### 4. Node ki Report Master ko Bhejna

Kubelet har thodi second mein master node (API Server) ko ek heartbeat bhejta hai jise **Lease object** bolte hain. Isse master ko pata chalta hai ki node zinda hai.
Agar node par RAM full ho jaye (OOM) ya Disk full ho jaye, to kubelet master ko inform karta hai aur node par `MemoryPressure` ya `DiskPressure` ka tag lag jata hai.

---

### CKA Exam Special: Static Pods Kya Hote Hain?

Normal Pods ko Master Node ka Scheduler schedule karta hai. Lekin Kubelet ke paas ek super-power hoti hai jise **Static Pods** kehte hain.

Kubelet worker node ke ek specific folder ko lagatar watch karta hai (usually `/etc/kubernetes/manifests/`).
Agar tum is folder ke andar koi bhi standard Pod ki `yaml` file rakh doge, to kubelet bina API Server ya Scheduler se pooche, **us pod ko directly node par run kar dega.**

**Exam Connection:** Kubernetes ka poora control plane isi tarah chalta hai! Jab tum `kubeadm` se cluster banate ho, to `kube-apiserver`, `etcd`, aur `kube-controller-manager` isi `/etc/kubernetes/manifests/` folder ke andar static pods ki tarah chalte hain, jinhe wahan ka local kubelet control karta hai.

---

### 💻 Debugging Commands (Hinglish Cheat Sheet)

Agar exam mein koi worker node `NotReady` aa raha hai, to tum us node par SSH karoge aur ye commands chalaoge:

```bash
# 1. Check karo ki kubelet chal bhi raha hai ya crash ho gaya
sudo systemctl status kubelet

# 2. Agar kubelet stopped hai, to use start karo
sudo systemctl start kubelet

# 3. Live logs dekho ki kubelet ko kya pareshani ho rahi hai (Bohat kaam aayega exam mein)
sudo journalctl -u kubelet -f

# 4. Check karo ki Static Pods ka path kya set hai kubelet ki config mein
cat /var/lib/kubelet/config.yaml | grep staticPodPath

```
---

Ye bohot accha sawaal hai! Isko bilkul aasan real-life example aur technical step-by-step se samajhte hain. CKA/CKAD exam ke liye ye concept samajhna bohot zaroori hai.

Pehle real-life example lete hain:
Maan lo tum ek **Flat (Apartment)** किराए (rent) par lete ho. Us flat ka ek fixed address hai, ek hi main door hai, aur ek hi electricity/water connection hai. Ab us flat ke andar 3 dost (containers) alag-alag kamron mein reh rahe hain. Wo teeno dost aapas mein ek dusre se bina kisi phone call ke directly baat kar sakte hain kyunki wo ek hi boundary ke andar hain, aur bahar wale kisi bhi vyakti ke liye un sabka address ek hi flat number hoga.

Kubernetes mein **Pod Sandbox** ya **Network Namespace** wahi "Flat" hai.

---

### Asli mein kya hota hai? (Step-by-Step)

Jab tum ek Pod ki YAML file apply karte ho jisme ek se zyada containers hain (jaise ek tumhara main App container aur ek Sidecar container), to Kubelet direct tumhare containers ko start nahi karta.

Kubelet ye steps follow karta hai:

#### Step 1: Kubelet "Flat" taiyar karta hai (The Sandbox)

Kubelet sabse pehle Container Runtime (containerd) ko bolta hai ki ek khali boundary banao. Linux ki bhasha mein is boundary ko **Network Namespace (netns)** kehte hain.
Is boundary ko active rakhne ke liye back-end mein ek bohot chhota sa, dummy container chala diya jata hai jise **`pause` container** kehte hain. Is poore setup ko hum **Pod Sandbox** bolte hain.

#### Step 2: CNI Isko Address deta hai

Jaise hi ye khali boundary (Sandbox) banti hai, Kubelet network plugin (CNI - jaise Calico ya Flannel) ko bolta hai: *"Bhai, is naye Sandbox ko ek IP address de do."* CNI is pure Sandbox ko **ek unique IP address** de deta hai.

#### Step 3: Tumhare Containers is "Flat" ke andar aate hain

Ab Kubelet tumhare asli application containers (jaise Nginx ya Node.js) ko start karta hai. Lekin unhe start karte waqt containerd ko bola jata hai: *"In containers ka apna alag network mat banana. Inko usi pehle se bane hue Sandbox (flat) ke network ke andar daal do."*

---

### Iska Fayda Kya Hota Hai? (Why is this important?)

1. **Shared IP Address:** Pod ke andar jitne bhi containers chalenge, un sabka **IP address ek hi hoga** (jo Sandbox ko mila tha).
2. **`localhost` Communication:** Kyunki saare containers ek hi network boundary (Flat) ke andar hain, wo aapas mein ek dusre se `localhost:port` par directly baat kar sakte hain. For example, agar tumhara main app port `8080` par chal raha hai aur sidecar port `9090` par, to sidecar main app ko directly `http://localhost:8080` par hit kar sakta hai. Unhe kisi external IP ya service ki zaroorat nahi hoti.
3. **Port Conflicts:** Kyunki wo ek hi network share kar rahe hain, do containers ek hi Pod ke andar same port (jaise dono port `80`) par nahi chal sakte, bilkul waise hi jaise ek hi flat ke andar do log same room number use nahi kar sakte.

**Short mein kahein to:** Pod Sandbox ek khali network ka dibba (box) hai jise IP milta hai, aur tumhare containers us dibbe ke andar baith kar us IP ko share karte hain.

---

## 2. kube-apiserver — your notes are good, here's the missing depth

Your description of the three gates (Authentication → Authorization → Admission Control) is correct. CRI communication aside, the key point that's missing from your notes is that this is the *only* component that talks to etcd directly — you noted this but didn't explain **why this matters**:

**Add this — why "single source of truth access" is architecturally important:**

```
## kube-apiserver akela etcd se kyu baat karta hai — iska reason

Agar har component (scheduler, controller-manager, kubelet) directly
etcd se baat karta, to:
- Har component ko etcd ka auth/cert knowledge chahiye hota (security risk)
- Concurrent writes se race conditions/corruption ka risk badh jata
- Validation/admission logic har component mein duplicate karna padta

Isliye Kubernetes ek single gatekeeper rakhta hai (api-server).
Sab components (scheduler, kubelet, controller-manager) bhi
api-server se hi baat karte hain — WO BHI etcd ko directly touch
nahi karte. Sirf api-server karta hai.

Flow: kubelet → api-server → etcd (NOT kubelet → etcd directly)

Interview one-liner: "etcd is intentionally only accessible through
the API server — this centralizes validation, auth, and consistency
checks in one place instead of duplicating that logic across every
control plane component."
```

## 3. Admission Controllers — this needs real expansion, you flagged it as dangerous but didn't explain the actual mechanism

Your note correctly senses danger ("mutating admission controller is very dangerous... whole production may get down") but doesn't explain the *order* or give concrete examples. This is heavily tested conceptually in interviews because it's where real production outages come from.

Let me verify the current official list before writing this, since admission controllers get added/removed across versions.An admission controller intercepts requests to the Kubernetes API server prior to persistence of the resource, but after the request is authenticated and authorized — confirms your instructor's gate ordering is correct. Now let me write the full admission controller depth your notes need, plus continue through etcd/Raft, scheduler, controller-manager, and the CRI pipeline. I'll keep going phase by phase as you asked.

---

```markdown
## 3 continued) Admission Controllers — full depth

Tumhare notes mein "validating vs mutating, mutating dangerous hai" likha
hai — sahi hai, lekin order aur mechanism missing hai.

### Sahi sequence (interview mein exact order bolna important hai)

Request → Authentication → Authorization → MUTATING admission controllers
  → Object schema validation → VALIDATING admission controllers → etcd

Note: mutating PEHLE chalte hain, validating BAAD mein. Kyun?
Kyuki mutating controllers object ko change kar sakte hain (e.g. default
values inject karna), aur validating controllers ko CHANGED/FINAL version
par check karna hota hai, raw input par nahi. Agar order ulta hota, to
validating ek aisi object ko check karta jo abhi badlne wala hai — useless.

### Mutating vs Validating — concrete examples (yaad rakhne layak)

MUTATING (object ko CHANGE karte hain):
- "DefaultStorageClass" → agar PVC mein storageClass specify nahi ki,
  to default class automatically assign kar deta hai
- "MutatingAdmissionWebhook" → custom webhooks jo sidecar inject karte
  hain (e.g. Istio service mesh proxy automatically inject hona)
- "DefaultTolerationSeconds" → pod ko default toleration time de deta hai

VALIDATING (object ko sirf CHECK karte hain, reject/accept karte hain,
change nahi karte):
- "ResourceQuota" → namespace ki limit cross ho rahi hai kya, check karta hai
- "PodSecurity" → security standards follow ho rahe hain ya nahi
- "NamespaceLifecycle" → kisi terminating namespace mein naya object
  banane se rokta hai

Tumhara observation sahi hai ki mutating dangerous hai — REASON:
Mutating webhook agar galat configured ho (buggy custom webhook), to wo
SILENTLY har pod ko modify kar sakta hai (e.g. wrong resource limits inject
karna, wrong sidecar inject karna) bina kisi error ke. Validating sirf
reject karta hai — fail loudly. Mutating fail SILENTLY karta hai (wrong
config apply karke) — ye zyada dangerous hai production mein.

### Real production incident example (samjhane ke liye)

Socho: tumne ek mutating webhook deploy kiya jo sab pods mein automatically
a memory limit "256Mi" inject karta hai agar specify na ho. Lekin tumhara
naya high-memory ML workload pod bina explicit limit ke deploy hua — 
webhook silently 256Mi limit laga degi, pod OOMKilled hoga baar baar,
aur logs mein seedha pata nahi chalega ki webhook ne ye kiya, kyuki YAML
mein tumne limit specify hi nahi ki thi (it was injected after the fact).
Ye debug karna mushkil hota hai kyuki "kubectl get pod -o yaml" final
state dikhata hai, jisme webhook ka mutation already merged hai.

### CRD ka exact relation admission controllers se (clarify karna zaroori)

Tumhara note thoda confuse karta hai: "CRD is a way to deploy some
application with prewritten configurations" — yeh thoda incomplete hai.

Sahi definition:
CRD (CustomResourceDefinition) Kubernetes API ko EXTEND karta hai —
naya resource TYPE define karta hai (jaise "Pod", "Deployment" already
built-in hain, waise hi tum "MyApp" naam ka naya type bana sakte ho).

CRD khud kuch deploy NAHI karta. CRD sirf SCHEMA define karta hai — 
"MyApp resource mein kya fields honge."

Actual deployment logic ek separate program likhta hai jise "Operator"
ya "Controller" kehte hain, jo CRD ko WATCH karta hai aur jab koi naya
"MyApp" object banta hai, to wo controller actual Pods/Deployments
create karta hai uske according.

Flow: CRD (schema define) → Custom Resource (actual instance/object
  banaya gaya) → Operator/Controller (background mein dekh raha hai,
  actual action leta hai)

Admission controllers se relation: CRDs ke liye bhi custom validating/
mutating webhooks likhe ja sakte hain (e.g. "agar MyApp resource mein
replicas > 100 hai to reject karo") — ye dynamic admission control
kehlata hai, aur ye production mein bahut use hota hai (e.g. Istio,
cert-manager, ArgoCD sab CRD + controller pattern use karte hain).

### Interview-ready answer

Q: "What's the difference between validating and mutating admission
   controllers, and why does order matter?"

A: "Mutating admission controllers run first and can modify the
   incoming object — for example, injecting default values or sidecar
   containers. Validating admission controllers run after, and they
   check the final, already-mutated object against policy rules,
   either accepting or rejecting it, but never modifying it. The order
   matters because validation needs to see the final state of the
   object, not the raw user input, since mutations could change
   whether it's valid. Mutating webhooks are riskier in production
   because they change behavior silently — a validating webhook fails
   loudly with a clear rejection, but a buggy mutating webhook can
   quietly inject wrong configuration into every object that passes
   through it."

### Check enabled admission controllers (tumne already note kiya hai, sahi hai)

cat /etc/kubernetes/manifests/kube-apiserver.yaml | grep enable-admission-plugins

Default enabled list version-dependent hota hai, current default includes:
CertificateApproval, CertificateSigning, CertificateSubjectRestriction,
DefaultIngressClass, DefaultStorageClass, DefaultTolerationSeconds,
LimitRanger, MutatingAdmissionWebhook, NamespaceLifecycle, NodeRestriction,
PersistentVolumeClaimResize, PodSecurity, Priority, ResourceQuota,
RuntimeClass, ServiceAccount, StorageObjectInUseProtection,
TaintNodesByCondition, ValidatingAdmissionPolicy, ValidatingAdmissionWebhook

Exam mein exact list yaad karna zaroori nahi — concept aur check command
yaad hona zaroori hai.
```

---

# Lecture 2 — Complete Deep Explanation (Continuing from Phase 1)

## 4. etcd — full depth

### 4.1 What etcd actually is, precisely

Your note: *"etcd (etc d stands for distributed) is a file system and we use it as a database."* This is **slightly wrong and worth fixing precisely** — it'll sound off in an interview.

```
## etcd naam ka matlab — clarify karna

etcd ka naam '/etc' (Linux config directory) + 'd' (distributed) se aata hai.
'/etc' Linux mein single-machine configuration store karta hai.
etcd ka idea hai: "/etc jaisa hi configuration store, lekin DISTRIBUTED
multiple machines ke across."

etcd EK FILE SYSTEM NAHI HAI. Ye ek standalone DATABASE hai jiska
on-disk storage engine BoltDB (ab renamed bbolt) use karta hai, jo
internally B+ tree based key-value store hai. File system se confuse
mat karo — file system files/directories manage karta hai, etcd
key-value pairs manage karta hai database ki tarah.

Correct one-liner: "etcd is a distributed, consistent key-value store
used as Kubernetes's backing store for all cluster data."
```

---

### 4.2 Why Kubernetes specifically picked etcd, not MySQL/MongoDB/Redis

This is a very common interview question your notes don't address at all.

```
## Kubernetes ne etcd kyu choose kiya, normal database kyu nahi?

Reasons (interview mein in points ko expand karke bolo):

1. STRONG CONSISTENCY: Kubernetes ko "eventually consistent" data
   acceptable nahi hai. Agar do API server replicas same time pe
   ek Pod schedule karne ki koshish karein, to etcd guarantee karta
   hai ki sirf EK write commit ho — Raft consensus ki wajah se.
   MongoDB/Redis jaise systems by default eventual consistency 
   dete hain (CAP theorem ka trade-off), jo Kubernetes ke liye
   dangerous hota — duplicate scheduling, race conditions ho sakte hain.

2. WATCH MECHANISM: etcd ka built-in "watch" feature components ko
   REAL-TIME notify karta hai jab koi data change hota hai (kube-scheduler,
   kubelet, controller-manager — sab etcd ko 'watch' karte hain via
   api-server). Traditional SQL databases mein ye feature nativе
   nahi hota — polling karna padta.

3. SIMPLE KEY-VALUE MODEL: Kubernetes ka data hierarchical hai 
   (/registry/pods/default/my-pod), jo key-value store mein naturally
   fit hota hai. Relational DB mein isko tables/joins mein todna 
   unnecessary complexity add karta.

4. LIGHTWEIGHT aur FAST: etcd Go mein likha gaya hai (Kubernetes ki
   tarah), gRPC + Protobuf use karta hai jo JSON/REST se fast hai.

Interview one-liner: "etcd was chosen because Kubernetes needs strong
consistency for cluster state, not eventual consistency — and etcd's
Raft-based design plus native watch support fits Kubernetes's 
declarative, reconciliation-based architecture perfectly."
```

### 4.3 Distributed Consensus Problem — what it actually is

Your note flags this as "need to be covered." Here's the real explanation:

```
## Distributed Consensus Problem kya hai

Problem statement: Maan lo tumhare paas 5 computers hain (nodes), aur
sabko EK SAME value par agree karna hai — chahe network slow ho jaye,
ek node crash ho jaye, ya messages delay se pahunche.

Real Kubernetes example: etcd ke 3 nodes hain. Tumne ek Pod create
kiya. Sawal: "Pod create hua" ye fact teeno nodes par SAME order mein
aur SAME tarike se kaise confirm ho?

Naive approach fail ho jata hai:
- Agar har node independently decide kare, to conflicting decisions ho
  sakte hain (node A ne maana Pod create hua, node B abhi tak nahi jaanta)
- Network delay ki wajah se ek node purana data dekh sakta hai

Distributed consensus algorithms (Raft, Paxos) ye guarantee dete hain:
1. SAFETY: Galat decision kabhi commit nahi hoga, chahe failures ho jayein
2. LIVENESS: Jab tak majority nodes alive hain, system progress karta
   rahega (naye decisions le sakta hai)

Raft is etcd's chosen algorithm to solve EXACTLY this problem — 
Paxos academically pehle aaya tha lekin samajhna bahut complex tha,
Raft isko understandable banane ke liye design kiya gaya (2014 paper,
Stanford — "In Search of an Understandable Consensus Algorithm").
```

### 4.4 Raft Consensus, Leader Election — corrected mechanics

Your note's example (*"1,2,3,4 nodes... 3 supports 2 and 4 supports 1... tie... re-election takes place until someone becomes leader"*) is **conceptually close but mechanically imprecise**. Here's the exact correction:

```
## Raft Leader Election — exact mechanism (correction zaroori)

Teen states: FOLLOWER, CANDIDATE, LEADER (tumne sahi likha hai)

Election process, step by step:

1. Sab nodes FOLLOWER state mein start hote hain.
2. Har follower ke paas ek "election timeout" hota hai — randomized,
   typically 150ms-300ms ke beech (HAR NODE KA TIMEOUT ALAG-ALAG RANDOM
   HOTA HAI — yahi tumhare "tie forever" problem ka SOLUTION hai)
3. Leader periodically "heartbeat" bhejta hai sab followers ko.
4. Agar koi follower apne election timeout tak heartbeat nahi sunta
   (matlab leader dead ho gaya ya network issue), to wo CANDIDATE ban
   jata hai aur apne aapko vote karta hai, phir baaki nodes se vote
   request karta hai ("RequestVote RPC").
5. Jis candidate ko MAJORITY vote milta hai (n/2 + 1), wo LEADER ban
   jata hai. Baaki sab apne aap FOLLOWER mein wapas convert ho jate hain.

### Tumhara "tie" scenario — kyu practically rare hota hai

Tumne socha "1,2 support each other, 3 supports 2, 4 supports 1 → tie."
REALITY: Raft mein candidates RANDOM TIMEOUT ke baad start hote hain,
isliye practically EK candidate dusre se PEHLE start ho jata hai aur
uske paas already kuch votes collect ho jate hain before doosra candidate
start kare. Agar genuinely tie ho bhi jaye (rare case), Raft mechanism:
- Election TIMEOUT khatam hone par, agar koi majority nahi mila, to
  saare candidates apna RANDOM timeout phir se reset karte hain aur
  NAYE term ke saath re-election start hoti hai
- Randomization ki wajah se next round mein tie lagna almost impossible
  hota hai (different random delays)

Isliye "infinite tie" nahi hota — randomized timeout guarantee karta hai
ki eventually koi ek candidate jeetega.

### TERM concept (missing from your notes, important)

Har election attempt ek "term number" leke hota hai (term 1, term 2...).
Agar ek node purane term ka message dekhta hai, wo use ignore kar deta
hai. Ye prevents "split brain" — purana leader agar wapas aaye to use
pata chal jata hai ki naya term/leader exist karta hai, aur wo follower
ban jata hai.

### Heartbeat / failure detection (tumne sahi note kiya tha)

Leader continuously "AppendEntries" RPC (heartbeat ka role bhi karta
hai) bhejta hai sab followers ko, typically every 50-150ms. Agar 
follower ko ek election-timeout duration tak heartbeat nahi milta,
wo maan leta hai leader down hai aur candidate ban jata hai.

Interview one-liner: "Raft uses randomized election timeouts specifically
to avoid split votes — since timeouts are randomized per node, one
candidate almost always starts its election slightly before others,
collecting votes before a tie can form. If a tie does occur, Raft just
restarts the election with fresh random timeouts in a new term."

Visual resource for self-study (recommended in your notes already,
good instinct): thesecretlivesofdata.com/raft
```

### 4.5 Quorum — your explicit "?" answered with exact numbers

```
## Quorum — exact definition aur formula

Quorum = minimum number of nodes jinko AGREE karna ZAROORI hai for any
write/read decision ko valid maana jaye.

FORMULA: quorum = floor(n/2) + 1, jahan n = total etcd members

Exact numbers (yaad karo, exam mein direct pucha jaata hai):
- 1 node  → quorum = 1 (ZERO fault tolerance)
- 2 nodes → quorum = 2 (ZERO fault tolerance — WORST setup, never use)
- 3 nodes → quorum = 2 (tolerates 1 failure)
- 4 nodes → quorum = 3 (tolerates 1 failure — SAME as 3 nodes, WASTE of a node)
- 5 nodes → quorum = 3 (tolerates 2 failures)
- 7 nodes → quorum = 4 (tolerates 3 failures)

### Tumhara "odd number" sawal ka exact answer

Even number cluster mein KOI FAULT TOLERANCE ADVANTAGE nahi hota apne
se ek chhota odd number ke comparison mein:
- 4-node cluster: quorum = 3, tolerates 1 failure
- 3-node cluster: quorum = 2, tolerates 1 failure
SAME fault tolerance, but 4-node cluster mein EXTRA node maintain karna
pad raha hai (cost+complexity badhi, benefit zero). ISLIYE odd number
hi use karte hain — extra node ka koi fayda nahi, sirf cost badhta hai.

### Production mein actual practice

Kubernetes control plane mein etcd typically 3 ya 5 nodes mein chalta hai:
- 3 nodes: chhote/medium clusters ke liye (tolerates 1 node down)
- 5 nodes: bade production clusters ke liye (tolerates 2 nodes down)
7+ rarely use hota hai kyuki write latency badh jati hai (Raft ko 
zyada nodes se confirmation chahiye hota hai har write ke liye — 
trade-off hai fault-tolerance vs write-speed ka)

### Quorum loss hone par kya hota hai (CRITICAL for CKA/CKS)

Agar quorum loss ho jaye (e.g. 3-node cluster mein 2 nodes down):
- EXISTING running Pods chalte rehte hain (worker nodes independent hain)
- LEKIN: koi NAYA write nahi ho sakta — naye Pods schedule nahi honge,
  kubectl apply/create/delete sab FAIL honge
- kubectl get abhi bhi kaam karega READ ke liye agar wo node abhi 
  bhi up hai (stale read ho sakta hai, depends on configuration)
- Cluster effectively "READ-ONLY-ish aur frozen" ho jata hai jab tak
  quorum restore nahi hota

Interview/exam answer: "If etcd loses quorum, the cluster doesn't crash
— pods that are already running keep running — but no new scheduling
decisions or API writes can be made until quorum is restored, because
Raft requires majority agreement to commit any change."
```

### 4.6 Write-Ahead Log — depth added

```
## Write-Ahead Log (WAL) — tumne sahi samjha hai, depth add karte hain

Tumhara samajh sahi hai: disk write se pehle text/log file mein likha
jata hai. Yahan exact reason aur mechanism:

WHY: Disk par actual data structure (B+ tree, BoltDB) update karna
slow operation hai (multiple disk seeks, tree rebalancing). Agar
beech mein power loss ho jaye, to tree corrupt ho sakta hai — half-
written state mein.

SOLUTION: Pehle ek SIMPLE APPEND-ONLY LOG file mein likho ("WAL")
— ye operation fast hai (sirf end mein add karna hai, kuch rearrange
nahi karna). Phir baad mein, asynchronously, actual database (BoltDB)
mein apply karo.

Crash recovery: Agar machine crash ho jaye actual DB update se pehle,
restart hone par etcd WAL file ko READ karta hai aur jo bhi entries
abhi DB mein apply nahi hui thi, unko REPLAY kar deta hai. Isse koi
committed write LOST nahi hota.

Location on disk: /var/lib/etcd/member/wal/ (default kubeadm setup mein)

Interview one-liner: "etcd writes to a write-ahead log before applying
changes to its underlying BoltDB store — this ensures durability: even
if the process crashes mid-write, the WAL can be replayed on restart
to recover any committed-but-not-yet-applied changes."
```

### 4.7 No encryption by default — depth and CKS relevance

```
## etcd encryption — tumne sahi flag kiya, ye CKS (Security) ke liye CRITICAL hai

Tumhara note sahi hai: by default etcd data DISK PAR PLAINTEXT mein
store hota hai. Iska matlab Secrets (jo base64 "encoded" hote hain,
ENCRYPTED nahi) bhi plaintext readable form mein etcd ke disk par
baithe hote hain (base64 decode karna trivial hai).

CKS ke liye ye EXTREMELY important hai:

1. ENCRYPTION AT REST enable karna padta hai manually:
   - EncryptionConfiguration object banate hain
   - kube-apiserver ko `--encryption-provider-config` flag se point
     karte hain
   - Providers: aescbc, aesgcm, kms (KMS sabse secure, cloud provider
     ke saath integrate hota hai)

2. etcd access control: etcd communication TLS certificates use karta
   hai (client-to-server aur peer-to-peer dono encrypted hone chahiye)
   — agar koi etcd ke socket tak access pa jaye bina cert ke, wo reject
   ho jayega, LEKIN agar disk file directly access kar le (jaise node
   compromise ho jaye), wo plaintext data padh sakta hai agar encryption
   at rest enabled nahi hai.

3. RBAC se etcd access restrict karna — sirf kube-apiserver ko hi
   etcd tak access hona chahiye, koi aur component direct nahi.

Interview/CKS answer: "By default, etcd stores all data — including
Secrets — unencrypted at rest on disk; Secrets are merely base64-
encoded, not encrypted. For production security, you must explicitly
configure encryption at rest using an EncryptionConfiguration resource
and the --encryption-provider-config flag on the API server, ideally
using a KMS provider for the strongest security."
```

## 5. kube-scheduler — full correction and depth

Your notes here are the most garbled part of the lecture — lots of half-remembered terms ("tentoleration", "prefilter", "preenqueue"). Let me give you the actual correct pipeline.

```
## Kube Scheduler — Correct Full Pipeline (yaad rakhna easy banane ke liye)

Tumne jo terms suna tha unke CORRECT naam:
"tentoleration" → ye actually "Taints and Tolerations" hai (do alag
  concepts jinka combined naam scheduler filtering mein use hota hai)
"prefilter" → "PreFilter" (sahi suna tha)
"preenqueue state" → "PreEnqueue" / unschedulable queue mein wapas jaana

### Full pipeline (exact official order):

SCHEDULING CYCLE (ek time par sirf EK pod ke liye chalta hai — sequential):
1. PreEnqueue — pod queue mein jaane se pehle ka check
2. (Pod jata hai "Scheduling Queue" mein, priority se sorted)
3. PreFilter — pod ki requirements ko ek baar process karta hai (e.g.
   pod ki affinity rules kya hain) — yahan ek hi baar hota hai, har
   node ke liye nahi
4. Filter — HAR NODE ke against check hota hai (PARALLEL chalta hai
   sabhi nodes ke liye): kya is node mein enough CPU/memory hai? Kya
   taint hai jo pod tolerate nahi karta? Kya nodeSelector match karta
   hai? Jo nodes fail hote hain Filter mein, wo eliminate ho jate hain.
5. PostFilter — agar Filter ke baad KOI bhi node nahi bacha (sab fail
   ho gaye), to ye PREEMPTION try karta hai — low priority pods ko
   evict karne ki koshish (tumne ye sahi note kiya tha "priority pod"
   wali baat)
6. PreScore — scoring shuru hone se pehle ka setup
7. Score — bache hue (filtered) nodes ko RANK karta hai (e.g. kis node
   mein resources sabse balanced hain, kis node mein pod ke existing
   pods already chal rahe hain affinity ke liye) — PARALLEL chalta hai
8. NormalizeScore — different scoring plugins ke scores ko common scale
   par normalize karta hai (0-100 jaisa)
9. Reserve — winning node par resources TEMPORARILY reserve kar diye
   jate hain (taaki doosra pod isी resources ko na le le scheduling 
   ke beech mein)
10. Permit — final check, kuch plugins pod ko WAIT karne ke liye bol
    sakte hain (e.g. gang scheduling — sab related pods ek saath ready
    hone tak wait)

   ===== Yahan SCHEDULING CYCLE KHATAM, BINDING CYCLE SHURU =====

BINDING CYCLE (ASYNCHRONOUS chal sakta hai, multiple pods parallel mein):
11. PreBind — bind se pehle ka kaam (e.g. volume actually provision
    karna agar zaroori ho)
12. Bind — ACTUAL decision likhi jaati hai etcd mein ("Pod X node Y
    par jayega") — yahi step hai jisko tum "binding" kehte ho
13. PostBind — cleanup/logging

### Scheduling cycle vs Binding cycle kyu alag hain (important "why")

Scheduling cycle SEQUENTIAL hai (ek time pe ek hi pod) kyuki resource
calculations mein RACE CONDITION nahi honi chahiye — agar 2 pods 
simultaneously same node ke resources count kar rahe hain, to dono
"fit" ho sakta hai dikhe jab actually sirf ek fit hota.

Binding cycle ASYNC ho sakta hai kyuki Bind sirf etcd mein WRITE karna
hai — ye independent operations hain, parallel mein safely chal sakte
hain.

Interview one-liner: "The scheduler splits work into a scheduling cycle,
which runs sequentially per pod to avoid resource-counting race
conditions during filtering and scoring, and a binding cycle, which can
run asynchronously since it's just committing the already-made decision
to etcd."

### Taints and Tolerations — full explanation (tumne flag kiya tha)

TAINT = Node par lagaya jata hai. Matlab: "Mujhse door raho, jab tak
tumhare paas special permission (toleration) na ho."
kubectl taint nodes node1 key=value:NoSchedule

TOLERATION = Pod par lagaya jata hai. Matlab: "Main is taint ko ignore
kar sakta hoon, mujhe is node par schedule karne do."

Effect types:
- NoSchedule: naya pod schedule nahi hoga (existing pods affected nahi)
- PreferNoSchedule: scheduler try karega avoid karne ka, but guarantee nahi
- NoExecute: naye pods schedule nahi honge AUR existing pods bhi EVICT
  ho jayenge jo toleration nahi rakhte

Use case: Master/control-plane nodes par by default ek taint hota hai
(`node-role.kubernetes.io/control-plane:NoSchedule`) taaki normal
application pods control plane par accidentally schedule na ho jayein.

### Priority aur Preemption (tumne partially samjha tha, ab complete)

Har Pod ko ek PriorityClass assign ho sakta hai (number, jyada number
= jyada priority).

Agar high-priority pod schedule hona chahta hai lekin koi node fit
nahi ho raha (resources kam hain), to PostFilter step mein scheduler
dekhta hai: "Kya kisi node par LOW priority pod hai jisko evict karke
main high-priority pod ko fit kar sakta hoon?"

Agar haan: low-priority pod EVICT (delete) hota hai, uski jagah pe
high-priority pod schedule hota hai. Evicted pod agar Deployment/
ReplicaSet se belong karta hai, to wo phir se "Pending" state mein
jata hai aur next available slot par schedule hone ka wait karta hai
— ye hi tumhara "preenqueue" wala observation tha, sahi disha mein
socha tha.

Interview answer: "Pod priority determines eviction order during 
scheduling pressure — a pending high-priority pod can trigger preemption
of a lower-priority pod on an otherwise-suitable node, evicting it to
make room. The evicted pod returns to the scheduling queue and waits
for the next available slot."
```

## 6. kube-controller-manager — full depth

Your note here is genuinely thin (*"checks whether everything is running fine... no more information is present"*). Here's the real depth.

```
## Kube Controller Manager — full explanation

### Core concept: "Reconciliation Loop" (most important idea, missing entirely from your notes)

Kubernetes ka SABSE FUNDAMENTAL design pattern hai: "DESIRED STATE vs
ACTUAL STATE."

Tum YAML mein likhte ho "main 3 replicas chahta hoon" — ye DESIRED STATE
hai, etcd mein store hota hai.

Controller-manager ke andar HAR controller ek INFINITE LOOP chalata
hai jisko "Reconciliation Loop" / "Control Loop" kehte hain:

```
LOOP FOREVER:
  observed_state = check actual cluster state (kitne pods abhi chal rahe hain)
  desired_state = etcd se padho (kitne chahiye)
  if observed_state != desired_state:
      take action (naya pod banao ya extra pod delete karo)
  sleep briefly, repeat
```

Ye pattern EK controller ka nahi — Deployment controller, ReplicaSet
controller, Node controller, Service controller — SABKA same pattern
hai, sirf "kya observe karna hai" aur "kya action lena hai" alag hai.

### Tumne jo controllers list kiye unka exact role:

- NODE CONTROLLER: Node "Ready" hai ya nahi monitor karta hai. Agar
  node ka heartbeat (kubelet se) miss ho jaye 40 seconds (default) tak,
  node ko "NotReady" mark karta hai. Agar 5 minutes (default, configurable
  `--pod-eviction-timeout`) tak NotReady rahe, to us node ke saare pods
  ko EVICT karke dusre nodes par reschedule karne ka trigger deta hai.

- SERVICE CONTROLLER: Service object create/delete hone par cloud
  provider ke saath integrate karta hai (e.g. LoadBalancer type Service
  banane par AWS/GCP mein actual load balancer provision karne ka
  request bhejta hai — actually ye CLOUD CONTROLLER MANAGER ka kaam
  zyada hai, kube-controller-manager sirf internal Endpoints update
  karta hai)

- NAMESPACE CONTROLLER: Namespace delete hone par uske andar ke SAARE
  resources (pods, services, etc.) ko cleanup karta hai (cascading
  deletion) before namespace ko fully remove karta hai.

- DAEMONSET CONTROLLER: Ensure karta hai ki DaemonSet ka EK pod HAR
  node par chal raha ho (naya node add hone par automatically wahan
  bhi pod create karta hai).

- CRONJOB CONTROLLER: Schedule ke according (cron syntax) naye Job
  objects create karta hai time par.

- REPLICASET/DEPLOYMENT CONTROLLER (your notes missed this — IMPORTANT):
  Ensure karta hai specified replica count match ho. Agar ek Pod crash
  ho jaye, ReplicaSet controller IMMEDIATELY naya Pod create karta hai
  replace karne ke liye — YE HAI ACTUAL "SELF-HEALING" MECHANISM jiske
  baare mein log baat karte hain Kubernetes mein.

### Tumhara incomplete flow — corrected

Tumne likha: "if something breaks it calls api server then api server
calls etcd and kubelet... no more information." Sahi flow:

1. Controller WATCHES api-server (continuous watch connection, polling
   nahi — efficient hai)
2. Controller dekhta hai: actual state != desired state (e.g. ek
   ReplicaSet ko 3 chahiye, abhi 2 hi chal rahe hain — 1 pod crash ho
   gaya)
3. Controller api-server ko REQUEST bhejta hai: "naya Pod object banao"
4. api-server (authentication/authorization/admission ke baad) etcd
   mein naya Pod object likh deta hai (status: Pending, abhi tak koi
   node assign nahi)
5. kube-scheduler is naye Pod ko dekhta hai (wo bhi watch kar raha tha),
   ek node assign karta hai (jo upar pura explain kiya)
6. us node ka kubelet (jo apne node ke liye assigned pods watch karta
   hai) is naye Pod ko dekhta hai, CRI ko bolta hai container banane ke
   liye

Important clarity: Controller-manager KABHI directly kubelet ko 
"command" nahi deta. Sab kuch api-server ke through hota hai — har
component sirf "watch karta hai aur api-server ko request bhejta hai."
Ye decoupled architecture hai — koi bhi component directly doosre 
component se baat nahi karta.

Interview-ready answer: "The controller manager runs multiple control
loops, each watching the API server for a specific resource type and
continuously comparing the desired state stored in etcd against the
observed actual state. When they diverge — say a Pod crashes and the
ReplicaSet count drops below its target — the relevant controller
issues a request to the API server to reconcile the difference, such
as creating a replacement Pod. This is the actual mechanism behind
Kubernetes's self-healing behavior."

### Work Queue concept (tumne note kiya "queue is used at most of places" — yahan expand karte hain)

Controllers internally ek "work queue" (rate-limited queue, Go library:
client-go ke andar `workqueue` package) maintain karte hain. Jab koi
change observe hoti hai (e.g. Pod deleted), controller us object ki
"key" (namespace/name) ko queue mein daal deta hai, phir queue se pop
karke process karta hai.

KYU queue use karte hain directly process karne ke bajaye:
1. RATE LIMITING: Agar same object baar-baar fail ho raha hai, queue
   exponential backoff laga deta hai (zyada baar try mat karo turant turant)
2. DEDUPLICATION: Agar same object 5 baar changed hua quickly, queue
   usko sirf ek baar process karta hai (latest state ke according)
3. RETRY LOGIC: Processing fail hone par item wapas queue mein chala
   jata hai retry ke liye

Yahi reason hai ki Prometheus metrics mein "workqueue_adds_total" aur
"workqueue_depth" jaisi metrics hoti hain (tumne apne notes mein already
list ki hain) — ye controller health ka indicator hain. Agar
workqueue_depth consistently high rahe, matlab controller processing
se slow hai ya stuck hai — debugging signal hai.
```

## 7. Cloud Controller Manager — depth

```
## Cloud Controller Manager — full explanation

Tumhara note sahi tha but thin: "same as kube controller manager but
at cloud level."

WHY it exists separately (historical context, often asked):
Pehle (Kubernetes ke early versions mein), cloud-specific logic
(AWS/GCP/Azure ke saath integration) HARDCODED tha kube-controller-
manager ke andar. Isse problem yeh thi: Kubernetes core code mein
AWS-specific, GCP-specific code mix hota tha — maintain karna mushkil,
aur naya cloud provider add karna matlab Kubernetes core ko hi modify
karna.

SOLUTION: Cloud-specific logic ko ALAG kar diya gaya — "Cloud Controller
Manager" (CCM) ek separate binary/component ban gaya jo sirf cloud
provider ke saath integrate karta hai. Har cloud provider apna CCM
implementation banata hai (AWS CCM, GCP CCM, Azure CCM).

### CCM ke andar kya controllers hote hain:

1. NODE CONTROLLER (cloud version): Jab naya VM cloud mein banta hai
   aur Kubernetes node banta hai, CCM cloud API se node ki details
   (region, zone, instance type) fetch karke Node object mein labels
   add karta hai.

2. ROUTE CONTROLLER: Cloud provider ke VPC/network mein routes setup
   karta hai taaki pods cross-node communicate kar sakein (cloud-specific
   networking).

3. SERVICE CONTROLLER: Jab tum `type: LoadBalancer` ka Service banate
   ho, CCM cloud provider ke API ko call karta hai actual Load Balancer
   resource provision karne ke liye (AWS ELB, GCP Load Balancer, etc.)
   aur Service object mein External IP update karta hai.

Interview one-liner: "The cloud controller manager decouples cloud-
provider-specific logic from the core Kubernetes controller manager —
it's what lets Kubernetes provision real cloud load balancers, attach
cloud disks, or label nodes with cloud metadata, without that
provider-specific code living in Kubernetes core itself."

On-prem/bare-metal clusters: CCM often NAHI hota agar cloud integration
ki zarurat nahi hai (e.g. kubeadm bare-metal cluster bina cloud ke).
```

## 8. Worker Node Components — kube-proxy depth

Your notes here have a real factual gap on private IP ranges and kube-proxy mechanism.

```
## kube-proxy — corrected and deepened

### Private IP ranges — tumne likha tha incompletely, full correct ranges:

RFC 1918 private IP ranges (yaad karo exact, interview mein pucha
jaata hai):
- 10.0.0.0 to 10.255.255.255       (10.0.0.0/8)    — largest range
- 172.16.0.0 to 172.31.255.255     (172.16.0.0/12) — Kubernetes pods
                                                       ke liye commonly use
- 192.168.0.0 to 192.168.255.255   (192.168.0.0/16) — home networks

Tumne jo likha "10.00000000.00000000.00000000" — ye binary representation
ki koshish thi lekin format galat hai. Correct: 10.0.0.0/8 ka matlab
hai pehla octet (8 bits) fixed hai "00001010" (10 decimal mein), baaki
24 bits available hain hosts ke liye.

Kubernetes practically:
- Pod CIDR typically 10.244.0.0/16 ya 10.0.0.0/8 jaisa range use karta
  hai (CNI plugin decide karta hai, e.g. Flannel default 10.244.0.0/16)
- Service CIDR (ClusterIP range) alag hota hai, typically 10.96.0.0/12
  ya similar — Pod CIDR se clash NAHI karna chahiye

### kube-proxy ka EXACT mechanism — tumne "IP tables banata hai" kaha,
yahan deep explanation:

kube-proxy ka kaam hai: Service ka STABLE Cluster IP ko ACTUAL changing
Pod IPs mein translate karna.

PROBLEM: Service ka IP fixed hota hai (e.g. 10.96.0.5), lekin uske
peeche jo Pods hain, unki IP badalti rehti hai (Pod restart, scale up/
down, reschedule). Application client ko sirf Service IP pata hona
chahiye, Pods ki changing IPs ka pata nahi hona chahiye.

SOLUTION mechanism (3 possible modes, tumhe sab teen pata hone chahiye):

1. IPTABLES MODE (most common, default in most clusters):
   kube-proxy Linux kernel ke netfilter/iptables rules use karta hai.
   Jab packet Service IP (10.96.0.5:80) par aata hai, iptables rule
   usko RANDOMLY ek actual Pod IP (jaise 10.244.1.5:8080) mein 
   redirect kar deta hai (DNAT — Destination NAT).
   Limitation: Bahut zyada Services hone par (1000s), iptables rules
   linearly scan hoti hain — PERFORMANCE issue ban sakta hai bade
   clusters mein.

2. IPVS MODE (newer, better performance):
   Linux IPVS (IP Virtual Server) kernel module use karta hai — ye
   HASH TABLE based hai (O(1) lookup), iptables ke linear scan se
   bahut fast hai bade clusters ke liye. Load balancing algorithms bhi
   support karta hai (round-robin, least-connection, etc.)

3. (Legacy) USERSPACE MODE: Purana tarika, ab use nahi hota — kube-proxy
   process khud packet forward karta tha (slow, deprecated).

EndpointSlice/Endpoints object: kube-proxy ko pata kaise chalta hai
ki konse Pod IPs current hain? Wo "Endpoints" (ya naye "EndpointSlice")
object ko watch karta hai, jo controller automatically update karta
rehta hai jab Pods change hote hain (Pod labels Service selector se
match karte hain to wo Endpoints list mein add/remove hota hai).

Interview answer: "kube-proxy watches Services and their corresponding
Endpoints, and programs the node's iptables (or IPVS) rules so that
traffic sent to a stable Service ClusterIP gets transparently DNAT'd
to one of the current healthy Pod IPs backing that service. This is
what gives Kubernetes Services their stable-IP, load-balanced behavior
even as the underlying Pods change."

### kubelet — your notes are mostly right, one addition

Tumne sahi kaha: "kubelet ka kaam h api server se baat karna or 
container ko run karta h and manage karta h."

ADD karne layak: kubelet "Node Agent" kehlata hai. Iska EXACT primary
job hai: "PodSpecs" (jo usko api-server se milte hain, uske node ke
liye assigned) ko le kar ENSURE karna ki described containers RUNNING
aur HEALTHY hain.

Kubelet bhi STATUS WAPAS bhejta hai api-server ko (Node status, Pod
status) — yahi mechanism hai jiske through `kubectl get nodes` aur
`kubectl get pods` ko current status pata chalta hai.
```

## 9. CRI / CNI / CSI — your notes are good conceptually, here's the missing precision

```
## CRI/CNI/CSI — tumhara conceptual samajh sahi hai, precision add karte hain

Tumhara mnemonic ("CRI makes containers run, CNI lets them talk, CSI
gives storage") EXCELLENT hai — rakho ise.

### CNI ke baare mein additional depth (tumne "observability provide
karna" likha — thoda clarify):

CNI plugin ka PRIMARY job sirf IP assign karna aur network connect
karna hai — observability (packet count, traffic monitoring) CNI ka
CORE responsibility nahi hai, ye SOME CNI plugins (jaise Cilium, jo
eBPF use karta hai) ke ADDITIONAL features hain, sab CNI plugins mein
nahi hota (e.g. simple Flannel sirf basic networking karta hai,
heavy observability nahi deta).

CNI EXACT specification: Ek single binary hota hai jo kubelet se 2
basic commands sunta hai: ADD (container ko network mein add karo) 
aur DEL (remove karo). Configuration JSON format mein hoti hai
`/etc/cni/net.d/` mein.

### Important addition missing from your notes: Kya hota hai agar
CNI na ho?

Bina CNI plugin install kiye, Node "NotReady" state mein reh jata hai
status mein "NetworkPluginNotReady" jaisa message ke saath. Ye exam
mein common troubleshooting scenario hai — naya cluster banaya, nodes
"NotReady" dikh rahe hain kyuki abhi CNI install nahi hua.

Verify command: kubectl describe node <name> | grep -A5 Conditions
```

## 10. The full kubelet → container boot pipeline — corrected and unified

This was the most confused part of your notes ("two different things he is telling"). Here is the **single correct version**, verified against current docs.

```
## kubelet → Container Creation — EXACT correct pipeline (confusion clear karte hain)

Tumne 2 versions likhe the jo contradictory lag rahe the. Yahan EK
unified, correct version hai jo dono explanations ko reconcile karta hai:

### Full pipeline (top to bottom):

1. kubectl apply → API Server → etcd mein Pod spec save hota hai
2. kube-scheduler is Pod ko ek node assign karta hai (upar explain kiya)
3. WORKER NODE ka kubelet (jo api-server ko WATCH kar raha tha apne
   node ke liye assigned pods ke liye) is naye Pod ko dekhta hai
4. kubelet, CRI (gRPC protocol ke through, jaise containerd ya CRI-O)
   ko 2 main commands deta hai:
   a. RunPodSandbox — PEHLE "pod sandbox" banta hai (ye basically
      wahi PAUSE CONTAINER hai jo humne pehle discuss kiya — network
      namespace hold karne ke liye)
   b. CreateContainer + StartContainer — phir actual application
      container(s) is sandbox ke andar banaye jate hain

5. containerd (CRI daemon) ye kaam karta hai:
   - Image pull karta hai (agar local mein nahi hai)
   - HAR CONTAINER ke liye ek "containerd-shim" process spawn karta hai
     (shim ka role: containerd aur OCI runtime ke beech middleman,
     container ka lifecycle/stdio manage karta hai EVEN AGAR containerd
     daemon restart ho jaye)
   - Shim phir OCI runtime (runc) ko call karta hai actual container
     banane ke liye, ek "config.json" (OCI runtime spec) ke saath jisme
     namespaces, cgroups limits, rootfs path, capabilities sab define
     hote hain

6. runc ye actual LOW-LEVEL kaam karta hai:
   - Linux system calls (clone(), unshare()) use karke NAMESPACES
     banata hai (network, PID, mount, etc. — jo humne Pod ke andar
     discuss kiya tha)
   - CGROUPS configure karta hai (CPU/memory limits enforce karne ke
     liye — tumne "cgroups system capability check karta h" kaha tha,
     PRECISE version: cgroups RESOURCE LIMITS enforce karta hai,
     "capability check" thoda vague tha)
   - Container process ko actually START karta hai (fork/exec)
   - Container start hone ke baad, runc apna kaam khatam karke EXIT
     ho jata hai — SHIM hi reh jata hai background mein container
     manage karne ke liye

7. Status wapas kubelet ko jata hai (via CRI gRPC response), kubelet
   api-server ko Pod status update karta hai ("Running")

### Tumhara confusion clear karne ke liye — DONO versions ek hi baat
keh rahe the, bas different level of detail mein:

VERSION 1 (tumne likha): "kubelet → CRI bolta hai → runc → cgroups
→ container banta hai" — YE SAHI HAI, lekin SHIM ka step skip kar
gaya tha beech mein.

VERSION 2 (tumne likha): "Kubelet calls CRI → CGroups configured →
CNI configure → images pulled → container started" — YE ORDER THODA
GALAT HAI. Correct order: Image pull SABSE PEHLE hota hai (container
banane ke liye image chahiye hi), phir namespaces/cgroups setup, phir
container start. CNI bhi ALAG se chalta hai sandbox banने ke time
(network namespace ko actual IP assign karne ke liye, runc ke kaam ke
PARALLEL/related lekin separate process hai).

### runc vs crun (tumne pucha tha)

runc: Original, REFERENCE implementation OCI runtime spec ka. Go mein
likha gaya hai (Docker se origin). Industry standard, most widely used.

crun: Red Hat ne C mein likha (Go ke bajaye). FASTER aur LESS MEMORY
use karta hai kyuki C, Go se zyada lightweight hota hai runtime overhead
ke point of view se. CRI-O (Red Hat ka CRI implementation) crun ko
often default mein use karta hai.

Practical difference for CKA: Tumhe sirf itna pata hona chahiye ki
"runc OCI runtime ka default/most common implementation hai, crun
ek faster C-based alternative hai" — exam mein dono ke beech switch
karne ka practical task rarely aata hai, conceptual question zyada
common hai.

### Shim concept — exact answer (tumne specifically pucha tha)

SHIM kya hai: Ek halka-sa process (e.g. "containerd-shim-runc-v2")
jo HAR CONTAINER ke liye ALAG se spawn hota hai.

SHIM kyu zaroori hai (3 reasons, important):

1. PARENT PROCESS PROBLEM: Agar containerd directly container ka
   parent hota (bina shim ke), to containerd daemon restart/crash
   hone par SAARE containers bhi mar jate (orphan ho jate, kyuki
   container process apne parent ke saath terminate ho sakta hai
   ya zombie ban sakta hai). Shim is between rehta hai taaki
   containerd restart ho sakे bina containers ko affect kiye.

2. runc EXITS after creation: runc sirf container BANANE ke liye
   chalta hai, phir EXIT ho jata hai (ye ek "one-shot" tool hai).
   Lekin container ko to background mein chalte rehna hai — kisko
   uska stdout/stderr collect karna hai, exit code track karna hai?
   YE HAI SHIM KA KAAM — shim hi container ke baad bhi zinda rehta
   hai aur ye sab manage karta hai.

3. ISOLATION: Ek container crash hone par sirf uska shim affect
   hota hai, doosre containers ke shims independent rehte hain.

Verify on real node: `ps aux | grep containerd-shim` — har running
container ke liye ek alag shim process dikhega.

Interview-ready summary answer (yaad karo, ye exact question bahut
common hai): "When kubelet needs to run a container, it talks to the
CRI implementation — like containerd — over gRPC. Containerd pulls
the image if needed, then spawns a per-container shim process. The
shim calls the OCI runtime, runc, passing it a generated config.json
that specifies namespaces, cgroups, and the rootfs. runc does the
actual low-level work of creating namespaces and cgroups via Linux
syscalls, starts the container process, and then exits — leaving the
shim behind to manage the container's lifecycle, collect its exit
code, and survive even if containerd itself restarts."
```

## 11. Prometheus Metrics — depth on what each one actually means

Your raw list has no explanation attached — here's what each one tells you and why it matters for CKA/CKS troubleshooting:

```
## Prometheus Metrics — kya batate hain (tumne list ki thi, ab explain)

- etcd_server_leader: Konsa node abhi leader hai (1 = leader, value
  change hone se leader change ka signal milta hai)

- etcd_server_leader_changes_seen_total: Leader kitni baar change
  hua. AGAR YE METRIC RAPIDLY BADH RAHA HAI ("leader flapping"), iska
  matlab network instability hai ya resource starvation — etcd cluster
  unhealthy hai. CKA troubleshooting mein red flag.

- etcd_network_peer_round_trip_time_seconds_bucket: etcd nodes ke
  beech network latency. High latency = slow consensus = cluster
  instability ka risk.

- workqueue_adds_total / workqueue_depth: (already explained upar)
  Controller kitna kaam process kar raha hai aur kitna pending hai.
  High depth = controller slow/stuck.

- kubelet_running_pods: Node par abhi kitne pods actually chal rahe
  hain — kubelet ke perspective se (vs api-server ke perspective se
  jo scheduled hain — difference dikhe to discrepancy debug karne
  layak hai).

- kubelet_pod_start_duration_seconds_count: Pod start hone mein 
  kitna time lag raha hai. Slow start times = image pull slow hai,
  ya node resource-starved hai.

- rule_sync: (typically kube-proxy/networking rules sync se related,
  context-dependent)

- apiserver_admission_controller_duration_seconds: Admission controllers
  (jo humne upar discuss kiya) process karne mein kitna time le rahe
  hain. Agar koi specific webhook bahut slow hai, ye metric pakad
  lega — production debugging mein useful (slow webhook = slow pod
  creation overall).

Interview context: "These metrics are typically scraped by Prometheus
and visualized in Grafana as part of cluster observability — they're
what an SRE or platform engineer watches to catch control plane
degradation before it causes visible application impact."
```

---

# These are raw notes created from lecture
- Internally components talk in GRPC and we talk to etcd or others in REST, mostly GRPC is used for internal communication.
- etcd (etc d stands for distributed) is a file system and we use it as a database
- Master node component: etcd, kube api server, scheduler, controller manager, cloud controller manager (part of controller manager)
Kube api server: Most important part of kube api server 
- for example if master node get down and our applicaton is already deployed then would it run? Answer: application would run because application is deployed on worker node and master node is for management only, if you want to update master node then it would cause error since Kube api server is not working.
Functions:
 - Authentication: bearer token, certificate
 - Authorization: Role based access control for example Team lead can create cluster and intern can see the cluster.
 - Admission controller: Two types: Validating admission controller, mutating admission controller. Mutating admission controller is very dangerous thing inside whole cluster, if someone has twick something then whole production may get down. Admission controllers mutate and validate karte h. most of security things are applied in mutating admission controllers. 
https://kubernetes.io/docs/reference/access-authn-authz/admission-controllers/
    - the kube-apiserver is the core component that serves the Kubernetes API. It acts as the front door and the central administrative hub for the entire Kubernetes control plane.
    - Exposes the API: It presents a RESTful HTTP API utilized by external users, internal cluster components, and command-line tools like kubectl.
    - Nerve Center: Every action—such as creating, modifying, or deleting a Pod—must pass through this server.
    - Single Source of Truth Access: It is the only component in the cluster that talks directly to the etcd data store.
    - Request LifecycleWhen you send a request to the Kubernetes API, the kube-apiserver processes it through three main sequential gates:
        Authentication: Verifies your identity via client certificates, bearer tokens, or identity providers.
        Authorization: Validates if you have the permission to execute the requested action using policies like Role-Based Access Control (RBAC).
        Admission Control: Evaluates the request against cluster rules (e.g., resource quotas) and can modify or reject the object before final execution.
        Once these gates are passed, the configuration is successfully written to etcd.
    - How will you know which Admission controller is enabled in you cluster? Answer: cd /etc/kubernetes/manifests then ls then cat kube-apiserver.yaml | grep enable    or grep admission
    - Plugin: kuberenetes has plugin based architecture in which you need to add plugin of the controller you want in your cluster.
    - CRD (Custom Resource Defination): It is way to define to extend the functionality of kuberenetes by creating a hardcoded way to deploy some application with prewritten configurations. We write CRD and use it when we want to deploy someapplication with specified resources deinfed in CRD.
    - 
 - Watch for Updates: Kube api server also watch for updates as well
 - Internally talks in grpc because it is extremely fast, it has protobuf as well and other things as well

- ETCD: It is a database, a distributed key value store, a No-Sql Database, once request is successul via kube api server then it gets stored in etcd
   - A problem named, distributed consensus:  need to be covered.
   - Raft Consensus: need to cover.
   - Etcd has 3 things: Follower, candidate, Leader. when there are multiple etcd nodes then election takes place for this and voting happens for example 1,2,3,4 nodes, 1, 2 wants to be leader and 3 supports 2 and 4 supports 1 then in this case there is tie and reelection takes place until some one becomes a leader. Once someone become a leader then other becomes follower. Leader on certain interval sends notification to every other node saying i am alive, once that notification don't comes that means 2 is not alive.
   - Raft Consensus: https://thesecretlivesofdata.com/raft/
   - Etcd is protobuf(fastest protocol) based, raft consensus based, Write ahead log based means before storing data in disk or storage the data is first written in text files so that data is not lost if machine got stopped while storing data in disk, because storing data in disk takes time.
   - By default, etcd does not have any encryption on data, anybody can read the data after getting access to etcd.
   - what number of nodes should be added to a cluster so that system is highly available in highly cost optimised manner? answer is always apply odd number of nodes because it's formula is (n/2 + 1), more fault tolerance more available. not understood completely but it needs to be completed.
   - Quorem?

- Kube Scheduler: It has two cycles Scheduling cycle and Binding Cycle. Request goes from api server to etcd then kube scheduler, etcd tells scheduler to schedule kar de, etcd kehta ki pod ko queue me daalde, then wo sort hoti h and then prefilter, prefilter me dekhte h ki koi tentoleration (what is it?) to nhi h, aise cheeze dekhi jaati h ki node pe ya pod pe koi aise restriction to nhi lagi jiski wajah se hum schedule na kar paye. Ager restriction lagi h then priority pod concept like if restrictions are there then agar low priority pod run ho rha hota h then use kick out kiya jata h and iski jagah ek priority pod dalta h, low priority pod ko preenqueue state me daal deta h, then it again gets schedule when it's time come. Inside Score, pre score and normalize score nodes me pod ko ek ranking milti h then pod reserve ho jata h for scheduling here scheduling cycle finish, after that binding cycle starts and binding cycle me pod ho jata h bind, don't have much information on this.

- kube controller manager: It checks whether everything is running fine or not, if something is not fine then it takes specific actions, it check whether pos is running fine, it checks node controller, service controller, namespace controller, daemon set controller, cron job controller, custom controller,...
   - if something breaks it calls api server then api server calls etcd and kubelet and ask kubelet to unschedule that pod,... no more information is present on this, clearity needed on this.

- Cloud controller manager: it does same as kube controller manager but at cloud level.


Worker Node: It has three things: Kubelet, kubeproxy, Container runtime interface.
- kubeproxy: It runs in background like a deamon set. installed on every worker node, jo ip internal ke liye banti h use cluster ip bolte h, cluster i[ is also called private ip. 
 10.00000000.00000000.00000000
 172.00000000.00000000.00000000
 192.00000000.00000000.00000000
these are private IP's which are not allowd to be used as public IP but these are only allowed for private Ip. Kubeproxy main work is to create IP tables for mapping service to pod using Private IP. Kubelet runs as a deamon uns as Deamon. Kubelet ka kaam h api server se baat karna or container ko run karta h and manage karta h.

- Initially kubenetes was a small project so at that time everything was hard coded so system was mostly hardcoded and companies using kubernetes has to update their code with every new release of kubernetes. So, for resolving this new interfaces were developed known as CRI,CNI, CSI

CRI, CNI, and CSI are standardized interfaces (or APIs) that allow Kubernetes to communicate with various third-party tools. They define how Kubernetes communicates with the outside world, while the actual software components that implement these interfaces are the drivers or plugins.
The easiest way to remember their jobs is:CRI makes your containers run.
CNI lets them talk to each other.
CSI gives them a place to store data.
Here is how the distinction works in practice:
1. CRI: Container Runtime InterfaceThe Interface: Defines how the Kubernetes node agent (kubelet) tells the underlying engine to start, stop, and manage containers.The Drivers: The actual engine doing the work. Common drivers include containerd, CRI-O, and Docker (via a built-in shim).
2. CNI: Container Network InterfaceThe Interface: Defines how pods get network connectivity and IP addresses. Kubernetes has no built-in networking; it relies entirely on CNI.The Drivers/Plugins: The networking tools that wire up your clusters. Examples include Calico, Flannel, Cilium, and Amazon VPC CNI. It assign IP to the pod, can check kitne packets in gye, kitne out gaye, what is movement of data and observability provide karna.
3. CSI: Container Storage InterfaceThe Interface: Defines how Kubernetes mounts and unmounts persistent storage (disks, block storage, file systems) to your containers.The Drivers: Third-party storage plugins written by cloud and hardware vendors. Examples include AWS EBS CSI Driver, Azure Disk, Google Persistent Disk, and Portworx.

- kubelet: ask CRI to deploy container, OCI ek set of parameters h isme multiple cheeze likhi h, ye docker ke time pe bna tha, isme runc kerke ek tool h jo container banata h jab CRI usko bolta h container banaane ke liye, runc cgroups ko bolta h container ke liye then cgroups system capability check karta h and agar everything fine then container bana deta h. Then response back to kubelet is sent that pod has been deployed then kubelet send container to run on that pod. runc go based h, iske baad crun aaya jo c me ha or jyada fast h. now he is telling different thing: Kubelet calls CRI to take care of the container then CGroups are configured then Configure Pods using CNI then images are pulled then container is started then application is run.
- it is not clear to me correctly, a lot of questions are the different between runc and crun, does CRI makes pod? what is shim concept?

- in kubernetes, queue is used at most of places to put items to be tracked.

Key Metrics that is used to track by prometheus
- etcd_server_leader
- etcd_server_leader_changes_seen_total
- etcd_network_peer_round_trip_time_seconds_bucket
- workqueue_adds_total
- workqueue_depth
- kubelet_running_pods
- kubelet_pod_start_duration_second_count
- rule_sync
- apiserver_admission_controller_duration_second
 
