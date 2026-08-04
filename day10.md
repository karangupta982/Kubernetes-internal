# Service Mesh, Secrets, CMs, External Secret Operator, Google Cloud Secret Manager
- Configmap: Store non confidential key value pairs
  - we can set it as immutable as true, so once it is made, can't be modified
  - always set readonly:true in configmaps
  - always implement configmap in volumes
  - Configmaps consumed in a volume are updated automatically if configmap is updated while if consumes as ENV Variables then it requires a pod restart, if pod is restarted then downtime will comes,..

  - Secrets:
    - type: opaque, generic, docker, tls,..
    - kubernetes me secrets encode hote h encrypt nahi. Encoded secrets ko decode bhi kar sakte h. it's a issue in kubernetes, because kubernetes says it is for orchestration and not for security.
    - These are consumed using Environment variables and data volumes. preferred way is data volumes
    - Almost nobody uses kubernetes secrets even for non confidential data, they use:
      - sealed secrets:
      - External Secret Operator (ESO): I didn't understand it properly, it has a lot of things to understand, so explin it carefully with details
      - Hashicorp Vault:
      - Cloud Secret Manager:
      - Secret Store CSI driver
     
  ---

  ## Service Mesh
  - OSI Model: explain it.
  - sidecarless: service mess runs on node level  
  - sidecar:
  - EBPF:
  - Cilium:
  - Istio:
  - Problems before service mesh:
    - communication with in pods
    - Serve traffic to end users
    - Rate limit
    - Recreate and eviction
    - Infra vs Kubernetes networking
    - restrict / Not restrict communication
    - No encryption in traffic
    - Observe in case of errors
