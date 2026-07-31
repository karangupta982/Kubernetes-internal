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





