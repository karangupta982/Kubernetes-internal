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
 
