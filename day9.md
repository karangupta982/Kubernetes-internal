# RBAC, Volumes, Storage Classes
- Authentication:
- Authorization:
- Kube api server is responsible for authentication and authorization.
- how does any command work like for example i did, kubectl get nodes then how will it be running, as far as i understood form lecture, request first goes to kubeConfig file which stores every detail about user, cluster, context(?),..
then request gets authenticated and authorized then don't know wha happens.
- Authentication can be done using webhook, static token, x-509 certs, OpenId connect, custom

- As i understand it is that, kubernetes uses context as where changes has to be made in which cluster or namespace,... and who is making that change from RBAC point of view.

- when there was namespace present then why context came?

---

# Authorization
- Cluster role: applied at cluster level, can be applied with cluster binding and role binding
- Role: applied at namespace level, would be applied with role binding
- Role Binding: Applied at namespace level
- ClusterRole Binding: Applied at cluster level

- best practices:
  - Role bindings >>> cluster role binding
  - limit sensitive workload
  - use namespaces

- Modes:
  - Always allow
  - Always Deny
  - Role based access control/ Attribute based access control
    - etc/kubernetes/manifests contains all the files which stores these information

- Subjects:
  - User:
  - Service Account:
  - Group

- Verb: get, list, watch, create, delete, update, patch, proxy, connect, port-forward

---

## Volumes:
- pods are ephemeral
- types: Ephemeral, Persistent
- AWS, Azure, GCP,.. have invented their volumes which are applied as in-tree in kubernetes. So, Kubernetes though it is a problem so it introduced something like CSI which lets these cloud vendors to build volumes on top of CSI. This is my high level understanding about it, i still not clear on this and don't have deep understanding of this.
- Categories: Cloud:?, EmptyDir:?, hostpath:?

- Persistent volume, Persistent Volume Claim
  - Phases:
    - Provision: persistent volume is created
    - Bind: claim using PVC
    - Use: then use that in pod
   
  - static provisioning:
  - dynamic provisioning:

  - Access Mode: readwriteonce, readwrite many,...
 
  - Reclaim policy:?
  - Status:?
  - StorageClass:? (Important)
- Can video, images, CSV,... be stored in volumes?
- Storage class, access mode on pv and pvc should be same and asked storage should be less than the capacity.






