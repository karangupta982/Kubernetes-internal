# Cert-Manager, Gateway API, Namespaces, TLS

- how communication happens in internet: Two entities client and server, client has session key and server has public key and private key. user needs to connect and communicate, user sends his query then server return it's public key in response then user sends this public key to Certificate authority then certificate authority checks whether the api key is valid for the certificate and if it is then user wraps it's session key with public key of server and sends it to the server and if it is valid then with it's private key it can decrypt the session key and now both user and server have session key, so in this way communication happens.
- Instructor also said something like http works on layer 7 and TLs works on layer 4, so how it happens something like that please explain the hidden concept in this also.
- TLS:?, MTLS:?

- let's encrypt: certificate authority, you will make an api call and that server will tell you whether this certificate is valid or not.
- cert manager: ?
- cluster issuer:?
- certificate and certificate request

- application is running standalone with ip address of service but using ingress application is not running. why is it so?

- Ingress advantages and disadvantage
  - ingress controller can handle only layer 7 load balancer and not layer 4 , layer 5
    
- Gateway API:
## 1. WHY — Ingress ki problems samjho pehle

```
Ingress ke saath 4 real problems hain production mein.
In problems ke solution ke roop mein Gateway API exist karta hai.
Bina problems samjhe, Gateway API "just another resource" lagega.
```

### Problem 1 — Annotation Hell (sabse badi problem)

```
Ingress spec bahut basic hai — sirf host, path, backend.
Baaki sab features ANNOTATIONS se karna padta hai.

PROBLEM: Annotations STANDARDIZED NAHI HAIN.

nginx controller:
  nginx.ingress.kubernetes.io/rewrite-target: /
  nginx.ingress.kubernetes.io/rate-limit: "100"
  nginx.ingress.kubernetes.io/ssl-redirect: "true"

Traefik controller:
  traefik.ingress.kubernetes.io/router.middlewares: ...
  traefik.ingress.kubernetes.io/rule: ...

AWS ALB Controller:
  alb.ingress.kubernetes.io/scheme: internet-facing
  alb.ingress.kubernetes.io/target-type: ip

EK CONTROLLER SE DOOSRE PAR MIGRATE KARO:
  → Saare annotations rewrite karne padte hain
  → Testing dobara karni padti hai
  → Production mein risk hota hai

REAL SCENARIO: Tumhare paas 50 Ingress objects hain nginx
annotations se bhar ke. Company decide karti hai Traefik pe
migrate karna hai. SARE 50 files manually update karne padte
hain alag-alag annotations ke saath. Ye 3-4 din ka kaam hai
aur har step mein outage ka risk hai.
```

### Problem 2 — Role separation nahi hai

```
Real company mein 3 alag teams hoti hain:

TEAM 1 — INFRA/PLATFORM TEAM:
  "Hamare paas cluster hai, network setup karo"
  Ye log decide karte hain: konsa controller, konsa cloud LB,
  TLS certificates kahan se aayenge, ports kaunse open honge

TEAM 2 — APP DEVELOPER TEAM:
  "Mujhe meri application expose karni hai"
  Ye log decide karte hain: kaunsi URL path, konsi Service,
  timeout kitna ho

TEAM 3 — SECURITY TEAM:
  "Auth, rate limiting, allowed IPs" manage karte hain

INGRESS KI PROBLEM: EK SINGLE Ingress object mein SABA KUCH
mix hai — infra config (TLS), app routing (paths), security
(annotations) — SABA EK FILE MEIN.

Iska matlab: App developer ko infra details pata honi chahiye.
Ya infra team ko har app ke routing rules manage karne padte
hain. Responsibilities blur hoti hain — production mein ye
real organizational problem hai.
```

### Problem 3 — Sirf HTTP/HTTPS, kuch aur nahi

```
Ingress ONLY HTTP aur HTTPS support karta hai.

Aaj ke applications ko chahiye:
- gRPC (microservices communication — Google ne banaya)
- TCP (databases, raw socket connections)
- UDP (DNS, gaming, streaming)
- WebSocket (real-time apps)
- TLS passthrough (end-to-end encryption without terminating)

INGRESS IN SABA KO NATIVELY SUPPORT NAHI KARTA.
Sirf HTTP/HTTPS ke liye design hua tha 2015 mein jab
Kubernetes simple tha.

Workaround: TCP/UDP ke liye ConfigMap use karo nginx-specific
  — phir wahi problem, non-portable, hacky.
```

### Problem 4 — Multi-tenancy nightmare

```
Scenario: Ek cluster mein 5 teams hain.
Team A ko apni routes chahiye, Team B ko apni.

Ingress mein: KOI BHI TEAM KISI BHI NAMESPACE KI Ingress
modify kar sakti hai (RBAC se restrict karo, but Ingress spec
khud ye concept nahi rakhta).

Real problem:
  Team A ne accidentally Team B ki route overwrite kar di
  → Production outage
  → "Kisne change kiya?" post-mortem
  → Manual RBAC policies likhke band karna padta hai —
    Ingress khud ye nahi enforce karta
```

---

## 2. WHAT — Gateway API kya hai exactly

```
DEFINITION: Gateway API ek NAYA set of Kubernetes resources
hai jo traffic management ke liye — Ingress ka REPLACEMENT,
designed specifically unhi 4 problems ko fix karne ke liye
jo upar explain kiye.

IMPORTANT FACTS:
- Ye Kubernetes core mein bundled NAHI hai — alag se
  install karna padta hai (CRDs install karte hain)
- Gateway API v1.0 → October 2023 (GA/Stable)
- Gateway API v1.4 → October 2025 (latest, more features)
- Ingress ABHI BHI exist karta hai — Gateway API ne
  forcefully replace nahi kiya, side by side chal sakte hain
- Ingress-NGINX Controller deprecation 2026 mein planned
  hai — isliye ab Gateway API seriously lena zaroori hai
```

---

## 3. WHERE — 3 resources mein split kiya (CORE CONCEPT)

```
Ingress ki ek file mein sab kuch tha.
Gateway API ne same concept ko 3 ALAG resources mein split
kiya — har resource ek ALAG TEAM ke liye.

RESOURCE 1: GatewayClass  → Infra/Cloud team
RESOURCE 2: Gateway       → Platform team
RESOURCE 3: Routes        → App developer team
```

### GatewayClass — "Konsa controller use karna hai"

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: GatewayClass
metadata:
  name: nginx-gateway                # naam
spec:
  controllerName: k8s-gateway.nginx.org/nginx-gateway-controller
  # ↑ Ye string us controller binary ko identify karta hai
  # jo is class ko implement karega — exactly IngressClass
  # jaisa concept, sirf Gateway API version
  description: "Production nginx gateway"
```

```
KAUN BANATA HAI: Infra/Cloud team — CLUSTER LEVEL, ek baar
MATLAB: "Hamara cluster nginx-gateway-controller use karta
hai infrastructure ke liye"

INGRESS EQUIVALENT: IngressClass (almost same concept)

AVAILABLE IMPLEMENTATIONS (controllers):
  k8s-gateway.nginx.org      → NGINX Gateway Fabric
  gateway.envoyproxy.io      → Envoy Proxy (Google)
  cilium.io/gateway          → Cilium (eBPF based)
  istio.io/gateway           → Istio service mesh
  traefik.io/gateway         → Traefik v3
  gateway.aws.amazon.com     → AWS Gateway Controller
```

### Gateway — "Kahan sunna hai, kaun allowed hai route karne ke liye"

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: main-gateway
  namespace: infra-team       # Platform team ka namespace
spec:
  gatewayClassName: nginx-gateway  # Upar wala GatewayClass

  listeners:                  # Kahan sunna hai
  - name: http
    protocol: HTTP
    port: 80

  - name: https
    protocol: HTTPS
    port: 443
    tls:
      mode: Terminate         # Yahan TLS terminate karo
      certificateRefs:
      - name: myapp-tls       # Certificate ka Secret
        namespace: infra-team

  - name: grpc
    protocol: HTTPS           # gRPC TLS ke upar run karta hai
    port: 8443

  - name: tcp-db
    protocol: TCP
    port: 5432                # Direct TCP! Ingress mein possible
                              # nahi tha natively

  allowedRoutes:              # KAUN routes attach kar sakta hai
    namespaces:
      from: Selector          # Specific namespaces hi routes
      selector:               # attach kar sakein
        matchLabels:
          allowed-gateway: "true"
    # YE HAI MULTI-TENANCY SOLUTION — Gateway khud enforce
    # karta hai ki kaun routes add kar sakta hai
```

```
KAUN BANATA HAI: Platform team — namespace-scoped
MATLAB: "Port 80, 443, 8443, 5432 par suno, sirf
  'allowed-gateway: true' label wale namespaces se
  routes accept karo"

INGRESS EQUIVALENT: Koi direct equivalent nahi tha — yahi
  to problem thi. Ingress mein ye sab annotations mein
  chhupi hoti thi.
```

### Routes — "Actual routing rules" (multiple types!)

```
YE INGRESS KA SABSE BADA ADVANTAGE HAI:
Ingress mein EK resource type tha — Ingress (sirf HTTP).
Gateway API mein ALAG ALAG resource types hain ALAG ALAG
protocols ke liye:

HTTPRoute   → HTTP/HTTPS routing    (STABLE, v1)
GRPCRoute   → gRPC routing          (STABLE, v1)
TCPRoute    → TCP routing            (EXPERIMENTAL)
UDPRoute    → UDP routing            (EXPERIMENTAL)
TLSRoute    → TLS passthrough        (EXPERIMENTAL)
```

### HTTPRoute — sabse common

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: product-routes
  namespace: team-products    # App team ka namespace
spec:
  parentRefs:                 # Konse Gateway se attach karo
  - name: main-gateway
    namespace: infra-team     # Platform team ka Gateway
    sectionName: https        # Specific listener

  hostnames:
  - "myshop.com"

  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /products
      headers:                # HEADER BASED ROUTING (Ingress mein nahi tha)
      - name: X-User-Region
        value: india
    backendRefs:
    - name: product-service-india
      port: 80
      weight: 100

  - matches:
    - path:
        type: PathPrefix
        value: /products

    filters:                  # REQUEST MODIFICATION (standardized, no annotations)
    - type: RequestRedirect
      requestRedirect:
        scheme: https
        statusCode: 301

    - type: RequestHeaderModifier
      requestHeaderModifier:
        add:
        - name: X-Internal-Source
          value: gateway

    - type: URLRewrite          # STANDARDIZED REWRITE (nginx annotation nahi)
      urlRewrite:
        path:
          type: ReplacePrefixMatch
          replacePrefixMatch: /v2/products

    backendRefs:
    - name: product-service
      port: 80
      weight: 90              # TRAFFIC SPLITTING (canary!) — built in
    - name: product-service-v2
      port: 80
      weight: 10              # 10% traffic naye version ko
```

### GRPCRoute — gRPC ke liye dedicated (Ingress mein nahi tha)

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: GRPCRoute
metadata:
  name: grpc-routes
  namespace: backend-team
spec:
  parentRefs:
  - name: main-gateway
    namespace: infra-team
    sectionName: grpc

  hostnames:
  - "grpc.myshop.com"

  rules:
  - matches:
    - method:
        service: com.myshop.UserService    # gRPC service name
        method: GetUser                     # gRPC method name
    backendRefs:
    - name: user-grpc-service
      port: 8080

  - matches:
    - method:
        service: com.myshop.OrderService
    backendRefs:
    - name: order-grpc-service
      port: 8080
```

---

## 4. HOW IT SOLVES THE 4 PROBLEMS

### Problem 1 fix — Annotation hell khatam

```
INGRESS:
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /v2/$2
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/limit-rps: "100"
    # Controller-specific, non-portable

GATEWAY API:
  filters:
  - type: URLRewrite              # STANDARDIZED
    urlRewrite: ...
  - type: RequestRedirect         # STANDARDIZED
    requestRedirect: ...
  
  # Rate limiting → Policy objects (RateLimitPolicy CRD)
  # CONTROLLER INDEPENDENT — same YAML, different controller pe
  # same behavior

MIGRATION: Nginx se Traefik jao → Route YAML wahi rahega,
  sirf GatewayClass change karo. Koi annotation rewrite nahi.
```

### Problem 2 fix — Role separation

```
INFRA TEAM:
  GatewayClass banata hai (ek baar, cluster level)

PLATFORM TEAM:
  Gateway banata hai:
    - Ports define karta hai
    - TLS manage karta hai
    - allowedRoutes se decide karta hai KON routes add kar sakta hai
  
  App team directly Gateway modify NAHI kar sakti — RBAC
  automatically enforce ho jata hai resource boundaries se

APP DEVELOPER TEAM:
  HTTPRoute banata hai apne namespace mein:
    - Apni Service ko expose karta hai
    - Gateway ka naam reference karta hai
    - Infra details se completely decoupled

KUBERNETES RBAC naturally boundaries enforce karta hai:
  infra-team namespace → sirf infra team ka access
  team-products namespace → sirf product team ka access
  Routes cross-namespace reference karte hain via parentRefs
```

### Problem 3 fix — Multiple protocols

```
Protocols table Gateway API ke saath:

| Protocol   | Ingress   | Gateway API     | Status      |
|------------|-----------|-----------------|-------------|
| HTTP       | ✅        | HTTPRoute ✅    | Stable      |
| HTTPS      | ✅        | HTTPRoute ✅    | Stable      |
| gRPC       | ❌ hacky  | GRPCRoute ✅    | Stable      |
| TCP        | ❌        | TCPRoute ✅     | Experimental|
| UDP        | ❌        | UDPRoute ✅     | Experimental|
| TLS passthrough | ❌   | TLSRoute ✅     | Experimental|
| WebSocket  | ❌ hacky  | HTTPRoute ✅    | Stable      |
```

### Problem 4 fix — Multi-tenancy

```
Gateway ki allowedRoutes field:

  allowedRoutes:
    namespaces:
      from: Selector
      selector:
        matchLabels:
          allowed-gateway: "true"

Sirf wo namespaces jin mein label "allowed-gateway: true" hai
wo is Gateway par HTTPRoute attach kar sakte hain.

Team A accidentally Team B ki routes affect NAHI kar sakti —
har team apne namespace mein apni routes manage karti hai,
Gateway ne boundary define kar di hai.
```

---

## 5. Protocols aur Methods — Gateway API kya kya support karta hai

```
## HTTP Methods — HTTPRoute kaunse methods route kar sakta hai

HTTPRoute spec mein method-based routing supported hai:

rules:
- matches:
  - path:
      type: PathPrefix
      value: /users
    method: GET          # Sirf GET requests
  backendRefs:
  - name: read-service

- matches:
  - path:
      type: PathPrefix
      value: /users
    method: POST         # Sirf POST requests
  backendRefs:
  - name: write-service

Supported methods: GET, POST, PUT, DELETE, PATCH, HEAD, OPTIONS

USE CASE: Read-heavy aur write-heavy endpoints alag-alag
  services par route karo (different scaling needs).

## Query Parameter based routing

rules:
- matches:
  - queryParams:
    - name: version
      value: "v2"        # ?version=v2 wale requests alag service ko
  backendRefs:
  - name: api-v2-service

## Header based routing

- matches:
  - headers:
    - name: X-Beta-User
      value: "true"      # Beta users ko naye version pe bhejo
  backendRefs:
  - name: beta-service

## Traffic splitting (Canary deployment natively)

backendRefs:
- name: stable-service
  weight: 95
- name: canary-service
  weight: 5
# 95% stable, 5% canary — NO ANNOTATIONS NEEDED
```

---

## 6. ADVANTAGES — complete list

```
1. STANDARDIZED FEATURES: Rewrite, redirect, header modification
   — sab spec mein hain, annotations mein nahi. Controller
   switch karo, YAML wahi kaam karega.

2. ROLE SEPARATION: GatewayClass (infra) → Gateway (platform)
   → Route (app dev) — har team apna resource manage kare,
   doosre ki responsibility touch na kare.

3. MULTI-PROTOCOL: HTTP, HTTPS, gRPC, TCP, UDP, TLS — ek API
   se sab handle. Ingress sirf HTTP/HTTPS tha.

4. BUILT-IN TRAFFIC SPLITTING: Canary deployments natively
   supported (weight field) — Ingress mein annotation hack
   tha ya 2 separate Ingress objects.

5. MULTI-TENANCY BUILT-IN: allowedRoutes se namespace-level
   isolation — Ingress mein manually RBAC likhna padta tha.

6. EXPRESSIVENESS: Header routing, query param routing, method
   routing — sab natively. Ingress mein controller-specific
   annotations se karna padta tha.

7. FUTURE-PROOF: Kubernetes SIG-Network actively develop kar
   raha hai — Ingress "frozen" hai (koi naye features nahi
   aayenge official spec mein).

8. SERVICE MESH INTEGRATION: Istio, Cilium, Linkerd — sab
   Gateway API support karte hain, same resources east-west
   traffic (pod-to-pod) ke liye bhi use ho sakte hain
   (GAMMA initiative via experimental features).

9. CROSS-NAMESPACE ROUTING: Ek Gateway se multiple namespaces
   ki Services ko route kar sakte ho — Ingress mein same
   namespace tak limited tha mostly.
```

---

## 7. DISADVANTAGES — honest assessment

```
1. COMPLEXITY: 3 resources (GatewayClass + Gateway + Route)
   samajhna aur manage karna Ingress ke ek resource se zyada
   complex hai. Chhote teams/projects ke liye overkill.

2. MATURITY: Ingress bahut mature hai — documentation, tutorials,
   community knowledge, StackOverflow answers bahut hain.
   Gateway API naya hai — debugging mein resources kam hain
   abhi.

3. SEPARATE INSTALLATION: Kubernetes mein bundled nahi — alag
   se CRDs install karne padte hain. Ek extra step.

4. EXPERIMENTAL FEATURES: TCP/UDP/TLS routes abhi bhi
   EXPERIMENTAL hain — production mein carefully use karo,
   API break ho sakti hai future versions mein.

5. CONTROLLER SUPPORT VARIES: Har controller sab features
   implement nahi karta — "supportedFeatures" check karna
   padta hai GatewayClass status mein, assume mat karo.

6. LEARNING CURVE: Team ko retrain karna padega — Ingress
   annotations yaad karne wale logon ko naya mental model
   chahiye.

7. MIGRATION COST: Existing Ingress-heavy clusters ke liye
   Gateway API pe migrate karna significant effort hai.
```

---

## 8. Ingress vs Gateway API — final comparison

```
| Aspect              | Ingress                | Gateway API           |
|---------------------|------------------------|-----------------------|
| API version         | networking.k8s.io/v1   | gateway.networking.k8s.io/v1 |
| Bundled in K8s?     | Yes                    | No (CRDs install karo)|
| Status              | Frozen (no new features)| Actively developed   |
| Protocols           | HTTP/HTTPS only        | HTTP,gRPC,TCP,UDP,TLS|
| Role separation     | No                     | Yes (3 resources)     |
| Advanced routing    | Annotations (hacky)    | Spec mein native      |
| Traffic splitting   | Annotations            | weight field native   |
| Portability         | Low (annotations vary) | High (standardized)   |
| Complexity          | Low                    | Medium-High           |
| Multi-tenancy       | Manual RBAC            | Built-in (allowedRoutes)|
| Learning resources  | Abundant               | Growing               |
| Production maturity | Very high              | Growing (v1.4 stable) |
| When to use         | Simple HTTP apps, existing clusters | New clusters, multi-team, gRPC, advanced routing |
```

---

## 9. Install aur verify karo

```bash
# Gateway API CRDs install karo (ye Kubernetes mein built-in nahi)
kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.2.0/standard-install.yaml

# Verify CRDs installed hue
kubectl get crd | grep gateway
# gatewayclass, gateway, httproute, grpcroute dikhne chahiye

# GatewayClass dekho (controller ne banaya hoga apna)
kubectl get gatewayclass

# Gateway dekho
kubectl get gateway -A

# Routes dekho
kubectl get httproute -A
kubectl get grpcroute -A
```

---

## 10. Interview answers

```
Q: "Gateway API kyu banaya Ingress hote hue?"

A: "Ingress had four fundamental problems at scale: annotation
sprawl where every controller used different non-portable
annotations, no role separation between infra and app teams,
support for only HTTP and HTTPS, and no built-in multi-tenancy.
Gateway API was designed to fix all four — it splits Ingress
into three resources (GatewayClass, Gateway, Route) aligned
to three personas, standardizes advanced features in the spec
itself, adds native support for gRPC, TCP, and UDP, and gives
platform teams explicit control over which namespaces can
attach routes to a Gateway."

Q: "Gateway API aur Ingress saath mein chal sakte hain?"

A: "Yes — they are completely independent resources. A cluster
can run both simultaneously. Many teams migrate incrementally
— new services use HTTPRoute while legacy services stay on
Ingress. Some controllers like NGINX Gateway Fabric and
Traefik v3 support both APIs behind the same controller
binary."

Q: "Kab Ingress use karo, kab Gateway API?"

A: "Ingress is still the right choice for simple HTTP routing
in small teams or existing clusters where migration cost isn't
justified. Gateway API makes sense when you need gRPC or TCP
routing, multiple teams sharing one cluster with clear
ownership boundaries, portable advanced routing without
annotation lock-in, or native canary deployment via traffic
splitting. For new clusters in 2026, Gateway API is the
recommended starting point given that Ingress is frozen and
Ingress-NGINX deprecation is planned."
```
---

- Loadbalancer: may be used for not only balancing traffic but also for a lot of work collecting metric, implementing ddos protecting, taking count of unique inputs, latency, failure rate,...

## Namespace
- Limit Range: A kubernetes object that defines the limit of resources a particular namespace can have.
- Noisy Neighbour: when a pod started taking more resources which is causing other nodes down and the completed system is down. That's why we use limit range for limiting the resources a particular namespace can contain. For stopping a particular namespace to get down we use pod level resource limit also defining what resource a specific pod can have.
- namespace: For stopping the whole system to get down because of noisy neighbour problem we use to divide the complete system into multiple namespaces based on requirement.
  
