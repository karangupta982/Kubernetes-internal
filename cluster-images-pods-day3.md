- Use kind distribution so that more than one node can be created instead of minikube.
- With this command: kind create cluster --name karan , it creates one node and for creating two or more nodes we need to have yaml file
- kabhi bhi version of kubernetes in worker node can not be greater than version of kubernetes in master node because backward compatibility issues aate h.
- Cluster ?, virtual machine ?, Object: kubernetes objects - pods, service, ingress, secrets, configmaps

- How to run specific version of K8s? Answer: Create a yaml file, make a node and put this specific version from this repo: https://github.com/kubernetes-sigs/kind/releases in the value of image key.

- kubeadm, kind, minikube, GKE, EKS creates different types of clusters.

Images are very important because these comes first before anything in infrastructure, so i should be known every concept involved with images.
- Docker Image: Distroless: ?
   - Scratch: ?
   - Alpine: ?
   - Rootless: ?
   - multi stage image: ?
   - Size of Distroless < Scratch < Alpine
- Question: Can we use latest tag? Image_name:tag (version), Answer: Never because if we give latest tag to any image then we and machines would not be able to keep track of versions, like what was the previous version? and what is the next version? because we always keep release as latest, to keep track of versions we use v1.0.0.

- An image consists of multiple layers, we don't know how many layers would be in an image, it depends on application and how we are writing the docker file.
- Run and copy creates new layers that implies bigger size which implies security issues. Small image less issues good for keeping and bigger image more secruity issues. If more layers then big size then more security issues then more coost in storing and running then more time in pulling the image then application would start late then load time of image will increase then revenue will decrease.

- Docker file is not case sensitive.

# Container:
  - Stateless Application or containers (Not sure which one is correct): It does not store the state of application, for e.g. Frontend applications.
  - Stateful application or containers (Not sure which one is correct): It does maintain the state of application for e.g. Database, because db needs to store every single entry.
  - Init container: ?
  - Ephemeral container: ?, we don't directly enter into running pods so we create ephimerical containers (copy of those containers and use these ephemeral containers as a backdoor to do something). Ephemeral containers are not used generally because these can't be killed.
  - Sidecar Containers: ?
  - Difference between Init and SideCar Containers: ?
  - Kata Containers: ?
  - Gvisor: ?
  - k3s: A kubernetes distribution with less components, remove some components of k8s and then package that removed version and you will get k3s, k3s is packaged k8s with less components.
  - k9s: it is a dashboard for visualizing the resources in you cluster, no comparison of k3s with k9s.
  - When we go from our local system to cloud based environment for kubernetes then we face problems because we can't access control plan in cloud based environments like GKE, EKS.

  - Pods: These are Disposable and Ephemiral.
     - apiGroups: ?
     - apiVersion: ?
     - kind: Type of object/resource(Pod, volume,...)
     - metadata: more information about objects/resource
     - spec:
        containers:
           name: nginx
           image: nginx:1.14.2 (image_name:tag)
           ports:
           - containerPort: 80
  - kubectl api-resources : a command that list api version of each resource and some other important details as well
  - kubectl run amit --image nginx -- sleep 1d (explain this command like i didn't understand -- mean, why is it there, sleep 1d what does it mean, why we have put it there)
  - kubectl amit1 --image nginx -- sleep 1d
  - kubectl get pods -o wide
  - kubectl delete pod amit1
  - kubectl run amit1 --image=nginx
  - kubectl edit pod amit
  - downward api: when resources can get information about itself.
  - we can configure network policy or firewall on pods so that traffic only come from selected resources.
  - A running pod can't be modified, if you try to modify it then kubernetes will make a new temporary file with those changes.
  - kubectl apply -f 11.yaml  (-f means file name, apply means achieve the final state, create means to implement this now)
    
# pods:
    - Normal: ?
    - Static: ?
    - Multi Container: ?
      - Dryrun:  kubectl run preritmunjal --image nginx --dry-run=client -o yaml
        - kubectl run preritmunjal --image nginx --dry-run=client -o yaml > pod.yaml

      - Output: ?
      - Exec: pod ke ander ek input based terminal open karna, we can enter in a pod. Command: kubectl exec -it nginx -- bash , after pressing enter we would be in interactive terminal of pod where we can run commands.
      - we never do exec in production because there we never touch running cluster for exec because pods are ephemeral. 
      - image pull policy: ?
      - Common errors: for examples: Image Pull Back Off error
