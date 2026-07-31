# THINGS I MUST KNOW COLD — CKA Master List
(Living document — har lecture ke baad update karna. Goal: in sab par bina sochke, bina docs dekhe answer ya command aana chahiye.)

Exam reality check: 2 hours, ~17-25 tasks, ~6-8 min per task, partial credit milta hai
isliye HAR task attempt karo. Domain weights: Troubleshooting 30% | Cluster Architecture
25% | Services & Networking 20% | Workloads & Scheduling 15% | Storage 10%.
Sabse zyada practice Troubleshooting + Cluster Architecture par karo — ye 55% hai.

---

## A) MUSCLE MEMORY COMMANDS (typing speed = exam speed)

1. `kubectl run <name> --image=<img> --dry-run=client -o yaml > pod.yaml` — pod template generate
2. `kubectl create deployment <name> --image=<img> --replicas=3 --dry-run=client -o yaml` — deployment template
3. `kubectl expose deployment <name> --port=80 --target-port=8080 --type=ClusterIP` — service banana
4. `kubectl scale deployment <name> --replicas=5` — scale up/down
5. `kubectl set image deployment/<name> <container>=<new-image>` — rolling update trigger
6. `kubectl rollout status / undo / history deployment/<name>` — rollout manage karna
7. `kubectl explain <resource>.<field>` — field documentation terminal mein hi
8. `kubectl get <resource> -o wide` / `-o yaml` / `-o json` / `-o jsonpath='{...}'`
9. `kubectl describe <resource> <name>` — Events section = WHY kuch fail hua
10. `kubectl logs <pod> -c <container> --previous` — crashed container ka last log

## B) TROUBLESHOOTING WORKFLOW (30% weight — highest priority)

11. Node NotReady → workflow: `kubectl describe node` → SSH node → `systemctl status kubelet` → `journalctl -u kubelet -n 100` → check `containerd` → fix → `systemctl restart kubelet` → verify from control plane
12. `journalctl -u kubelet -f` — live tail karna real-time debugging mein
13. `crictl ps -a`, `crictl pods`, `crictl logs <id>` — jab kubectl access na ho ya runtime-level issue ho
14. Pod stuck in `Pending` → check: resource requests vs node capacity, taints/tolerations, PVC unbound
15. Pod stuck in `CrashLoopBackOff` → `kubectl logs --previous` pehle dekho, phir `describe` ke Events
16. Pod stuck in `ImagePullBackOff` → image name/tag typo, registry auth missing (imagePullSecrets), network access to registry
17. Service not reachable → check: Selector labels match karte hain Pod labels se? Endpoints populate hue? (`kubectl get endpoints`)
18. DNS not resolving inside pod → CoreDNS pods running hain? `kubectl exec` se `nslookup kubernetes.default` test karo
19. Static pod issue (etcd/api-server/scheduler/controller-manager crashing) → edit file directly in `/etc/kubernetes/manifests/`, kubelet auto-detects change
20. `kubectl get events --sort-by=.metadata.creationTimestamp -A` — cluster-wide recent events, bada-picture debugging

## C) CLUSTER ARCHITECTURE & INSTALLATION (25% weight)

21. Control plane components: `kube-apiserver`, `etcd`, `kube-scheduler`, `kube-controller-manager` — sabka role ek line mein bol sako
22. Worker node components: `kubelet`, `kube-proxy`, container runtime (`containerd`)
23. `kubeadm init` / `kubeadm join` — cluster bootstrap karne ka basic flow
24. `kubeadm upgrade plan` / `kubeadm upgrade apply` — node-by-node upgrade procedure
25. etcd backup: `etcdctl snapshot save <path>` with correct `--cacert`, `--cert`, `--key`, `--endpoints` flags — ye exam mein guaranteed aata hai
26. etcd restore: `etcdctl snapshot restore` → naya data-dir point karna static pod manifest mein
27. RBAC basics: `Role` vs `ClusterRole`, `RoleBinding` vs `ClusterRoleBinding` — namespace-scoped vs cluster-scoped
28. `kubectl auth can-i <verb> <resource> --as=<user>` — RBAC permission test karna
29. CRI/CNI/CSI — teeno interface ka matlab alag-alag bol sako (Container Runtime, Network, Storage interface)
30. Helm basics: `helm install`, `helm upgrade`, `helm list` — package manager ka role

## D) SERVICES & NETWORKING (20% weight)

31. Service types: `ClusterIP` (default, internal-only) vs `NodePort` (external, fixed port range 30000-32767) vs `LoadBalancer` (cloud-provider dependent)
32. Pod-to-Pod communication: flat network model — har pod ko unique IP, NAT ki zarurat nahi
33. NetworkPolicy: default deny + selective allow — `podSelector`, `ingress`, `egress` rules likhna aana chahiye
34. CoreDNS: pod ka DNS name format → `<service-name>.<namespace>.svc.cluster.local`
35. Ingress vs Gateway API — Ingress purana tarika hai, Gateway API newer/more flexible (exam dono cover karta hai ab)
36. `kube-proxy` ka role — Service IP ko actual Pod IP mein translate karta hai (iptables/IPVS rules ke through)
37. Network namespace sharing within a Pod — same IP, localhost communication between containers
38. Pause container — Pod ki network identity hold karta hai, real containers iska namespace join karte hain

## E) WORKLOADS & SCHEDULING (15% weight)

39. Liveness vs Readiness vs Startup probes — kab kaunsa use karna hai (clear difference bolna aana chahiye)
40. Taints aur Tolerations — node taint karta hai "mujhe avoid karo," pod tolerate karta hai "main phir bhi aa sakta hoon"
41. Node Affinity / Anti-affinity — pod ko specific node par schedule karne ka declarative tarika
42. ConfigMap vs Secret — dono key-value store karte hain, Secret base64-encoded hota hai (encryption NAHI)
43. ConfigMap/Secret update hone par pods automatically restart NAHI hote — manual `kubectl rollout restart deployment/<name>` chalana padta hai
44. HPA (Horizontal Pod Autoscaler) — `kubectl autoscale deployment <name> --min=2 --max=10 --cpu-percent=80`
45. Init containers — main container se pehle chalte hain, complete hone tak main container start nahi hota

## F) STORAGE (10% weight)

46. PV (PersistentVolume) vs PVC (PersistentVolumeClaim) — PV = actual storage resource, PVC = request for storage
47. Access modes: `ReadWriteOnce`, `ReadOnlyMany`, `ReadWriteMany` — kab kaunsa applicable hai
48. StorageClass — dynamic provisioning enable karta hai, manual PV banane ki zarurat khatam karta hai
49. Reclaim policy: `Retain` vs `Delete` — PVC delete hone par PV ka kya hota hai

## G) FOUNDATIONAL THEORY (interview-only, low exam-weight but HIGH interview-weight)

50. Serialization/Deserialization, YAML vs JSON vs Protobuf, why Kubernetes chose YAML for manifests but JSON internally — ye "tell me about yourself / explain basics" type interview questions mein aata hai, exam mein nahi aata seedha
51. `kubectl get <resource> -A` — pehla command jab koi resource "missing" lage
52. `kubectl api-resources` — namespaced vs cluster-scoped check karne ka authoritative tarika
53. `kubectl.kubernetes.io/last-applied-configuration` annotation — apply ka three-way merge isi par based hai
54. create + apply ko MIX mat karo same resource par — annotation missing hone se merge unpredictable hota hai
55. Bare Pod fix = delete+recreate; Deployment-managed pod fix = `kubectl set image` (controller khud replace karega)
56. kubectl context switch HAR exam question se pehle — `kubectl config use-context <name>`
57. Exam-day aliases: `alias k=kubectl`, `export do="--dry-run=client -o yaml"`, `export now="--force --grace-period=0"`
58. veth pair + bridge (cni0) = same-node pod-to-pod mechanism
59. Overlay (VXLAN encapsulation) vs Direct Routing (BGP) = two CNI approaches for cross-node traffic
60. NetworkPolicy = allow-list model, only enforced if CNI supports it (Flannel basic mode does NOT)
61. Service = no process, just iptables/IPVS DNAT rules maintained by kube-proxy
62. DNS FQDN: <svc>.<namespace>.svc.cluster.local — search domains in /etc/resolv.conf make shortcuts work
63. Service type hierarchy: ClusterIP ⊂ NodePort ⊂ LoadBalancer (each builds on the previous)
64. Ingress needs an Ingress Controller running — the Ingress object alone does nothing
65. Egress uses SNAT at the node level to make pod traffic internet-routable
66. Kubernetes Networking Model's 3 guarantees: no-NAT pod-to-pod, no-NAT node-to-pod, consistent self/external IP view
67. imagePullPolicy — Always/IfNotPresent/Never; latest tag → Always by default
68. securityContext — runAsNonRoot, readOnlyRootFilesystem, capabilities drop/add
69. Liveness restarts on fail; Readiness removes from endpoint on fail; Startup
    protects slow starters
70. QoS classes: BestEffort (evicted first) → Burstable → Guaranteed (evicted last)
71. Label selector is mandatory and must match template labels in RS/RC
72. ownerReferences field = pod ka "parent" resource, khaali = free/bare pod
73. kubectl rollout undo (NOT kubectl rollback — ye command exist nahi karta)
74. Rollback to specific version: --to-revision=N (N becomes latest revision number)
75. Deployment internally manages multiple ReplicaSets — old ones kept for rollback history
76. RollingUpdate strategy: maxUnavailable + maxSurge control rollout speed/safety
77. StatefulSet: ordered startup, stable network identity, dedicated PVC per pod
78. kubectl rollout pause/resume — mid-rollout control
79. kubectl rollout restart — recreate pods without image change
80. terminationGracePeriodSeconds: 30 — SIGTERM → 30s → SIGKILL
81. A record = domain → IP direct mapping
82. CNAME = domain → domain (chain) — cloud LB ke liye use karo
83. Ingress = routing rules ka declaration, khud kuch nahi karta
84. Ingress Controller = actual reverse proxy pod jo rules implement karta hai
85. ingressClassName (spec field) = current standard, annotation deprecated
86. IngressClass = cluster-scoped object jo controller ko represent karta hai
87. Default IngressClass: ingressclass.kubernetes.io/is-default-class: "true"
88. Ingress annotations = controller-specific config, non-portable
89. pathType: Exact vs Prefix — Exact full match, Prefix subtree match
90. SSL termination Ingress Controller par hoti hai, pods par nahi
91. Gateway API 3 resources: GatewayClass (infra) → Gateway
    (platform) → Route (app dev) — role-oriented split
92. HTTPRoute, GRPCRoute stable; TCPRoute, UDPRoute experimental
93. Traffic splitting built-in: backendRefs weight field, no annotations
94. allowedRoutes field = Gateway API ka multi-tenancy mechanism
95. Gateway API CRDs alag install karne padte hain — K8s mein bundled nahi
96. Ingress "frozen" hai — naye features nahi aayenge official spec mein
97. Ingress-NGINX deprecation 2026 planned — Gateway API migration zaroori
98. GRPCRoute: gRPC service name + method name se routing possible
99.  TLS handshake 2 goals: server identity verify + session
     key securely exchange → phir fast symmetric encryption
100. TLS 1.3: 1 round trip (1.2 mein 2 tha), forward secrecy
     mandatory (ECDHE), weak ciphers removed
101. Certificate chain: Root CA → Intermediate CA → Server cert
     Root CA offline rakha jaata hai — security reason
102. mTLS: dono sides certificates present karte hain, shared
     internal CA verify karta hai — service mesh ka foundation
103. Zero Trust: "cluster ke andar = trusted" WRONG model
     mTLS ensures even internal traffic is authenticated
104. ACME protocol: RFC 8555 standard — Let's Encrypt implement karta hai
105. HTTP-01 challenge: cert-manager automatically Pod + Service +
     Ingress banata hai temporarily, cleanup bhi karta hai
106. DNS-01 challenge: TXT record `_acme-challenge.domain.com`
     Wildcard certs ke liye zaroori
107. Let's Encrypt staging pehle use karo — production rate limits
     strict hain (5 dup certs/week)
108. CertificateRequest auto-created hai — manually banana nahi
109. OOMKill = memory limit cross → process killed (incompressible)
     CPU throttle = CPU limit cross → slow but alive (compressible)
110. LimitRange = per pod/container cap
     ResourceQuota = entire namespace total budget
     DONO use karo production mein
111. Ingress debug workflow: DevTools → controller logs →
     exec inside pod → compare standalone vs ingress behavior
112. Mixed content = HTTPS page pe HTTP resource → browser blocks
     Fix: sab API calls HTTPS se karo
---

## 📌 Update Rule for this list
Har naye lecture ke baad:
- Naye exam-relevant commands/concepts is list mein ADD karo (number 51, 52...)
- Agar koi item "ab cold pe aata hai" to usko ✅ mark karo
- Weekly ek baar pura list bina dekhe bolne ki practice karo (jaise flashcards)
