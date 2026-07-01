Youtube video Link: https://www.youtube.com/watch?v=A7F7lKzfcsM&t=573s

Usefull Links: 
https://cilium.io/use-cases/service-mesh/
https://katacontainers.io/
https://kubernetes.io/docs/tasks/configure-pod-container/security-context/
https://kubernetes.io/docs/concepts/workloads/controllers/daemonset/
https://github.com/aquasecurity/kube-bench
https://kyverno.io/policies/
https://github.com/stackrox/kube-linter
https://github.com/aquasecurity/trivy
https://github.com/sharkdp/bat



Image of an application is build on base images, there are multiple layers in image and sometimes in some layer there might be CVE (Common Vulnerability and exposure), each layer has changes for having CVE, we want that each layer should be secure so that no hacker could enter in container (How hacker enters and does everything), So for solving this we use trivy tool, it has all the CVE stored. Trivy can be integrated standalone or in CICD. Our aim is to minimize CVE's. Three types of CVE, Critical, High, Low.
commands: trivy image nginx --severity CRITICAL
For security issues we use:
  - Multi stage build
  - Distroless images
  - Trivy tool

# Bat
Beutiful coloring and good representation of file instead of using cat command.

 # What happens when kubectl command is hit
 ## i write kubectl command -> it goes to api server -> api server does authentication, authorization then admission controllers check with mutating controllers then validating controllers then ...(Don't know what happens after this, explain it clear first then take an example and then dry run the explaination)

# Kyverno
### What is it? why does it exist? what it is used for? What it actually does when run in real time like when instructor run it, it was creating: CRDs (what is it?), Admission Controller, Report controller, Cleanup controller, Background controller.

## Use Cases of Kyverno:.....

- Alternative of kyverno: OPA, Gategay.

- kubectl get pods --watch (to see changes in any running command)

- If we want that each new pod getting created should have a defined CPU and RAM, so first we need to apply kyverno policy, and then when we create any pod then it will have CPU and Memory already assigned.

- Difference between admission controllers (mutating and validation) and kyverno policies?
-  Kyverno policies are applied clusterwide and admission controllers are also applied cluster wide.
-  kubelinter a tool to lint your yamls based on production grade implmentation.
-  kube-bench is a tool that scans whole cluster and tells you what are the mistakes present in configurations with the solution to be done.

# Pods
- Static pods: what is it? kubelet is responsible for static pods. kubelet, api server, etcd runs as static pods while you are working with kubeadm cluster. It runs as static pods because if any static pods get down then kubelet could restart and reboot that without asking to api server. Location of these static pods: etc/kuberenetes/manifests

- Daemonset comes with limited features, can't do rollout upgrades and can't define surge. Deployments comes with advanced configurations, can define deployment strategies, surge, rollout upgrades.
- backoff algorithm in kubernetes: in every resource of kubernetes this backoff algorithm is applied and when some resource is not responding then kuberenetes trigger it starting at 20, 40, 80, 160,... seconds till 600 seconds , after that announces it as dead resource.

# init containers:
- A pod can run multiple containers based on usecase and money you have.
- An init runs before application containers, it mainly used to download configurations, files from repo which are required for application to run, download of distroless containers. Once init container returns 0 (Success, kubernetes me success matlab 0 otherwise nozero for failure) then only application container will run.
- There can be multiple init containers in a pod.
- Init containers run sequentially, we can also define priority.
- kubectl logs humara-pod -c init-container.
- Init container once it runs successfully, it gets deleted

# Side-car Container
- Side-car existence me 1.29 version in beta
- Same as Init-container but it does not get deleted after successful run.
- IT run continuously
- If you change restart policy in inti container to always then it will work as side car container
- It's main purpose is that until main container is running side car container should run.
- -c in commands specifies in which container we want to enter

# complete workflow
- Kubectl calls api server for new pod, api server calls etcd to create entry in db then once done etcd calls apiserver done then api server calls scheduler to find node for this pod then schduler calls api server done then api server calls kubelet that this is the node and create you pod, then init containers run if present if it's response is 0 then sequence wise containers run then probes are set (probes make sure that your node is healthy).

- we can run python or any type of commands in pod based on base image because in base image only the dependencies are present for that command.
- companies use sidecar for installing dependencies of running containers

# Restart policy
Three types of restart policies are there: 
Never: If pod dies then no need to create it,
On Failure: In case pod get fail then recreate it,
Always (default): If is enabled in all containers whether it is side car, main containers, init container


# Containers
Container can only run one process, if it would be running multiple processes then we could do logging and monitoring with in the same container but we could not because container can only run one process. That's why we use sidecar for fetching logs and metrics.

# Termination of POD
As you apply kubectl it get to api server, apiserver-> etcd -> scheduler->apiserver-> kubelet-> 
Kubelete first send a linux signal called SIGTERM and Grace Timeout Period of 30 sec to Container runtime saying that ye pod delete ho jana chahiye with grace full deletion. Once that time get out then kubelet again sends SIGKILL command to container runtime saying that kill it.

Grace full deletion: Here you give buffer so within that buffer all the files get deleted.
Non grace full deletion: You want to delete instantly then here some of files don't get deleted completely.

## Pod lifecycle policy: Pending-> succeeded -> failed -> Running -> unknown

## Container Lifecycle: Running -> waiting  -> terminated

So it may happen that container is dead but pod is still running and many of it's permutation and combinations are possible

(Explain about it completely, explain multiple topics like what these are, why these exist, what it does, what questions i should be aware about, interview questions)
## Runtime class: Famous runtime class (Kata)
## Security Context:
## Service mesh:
