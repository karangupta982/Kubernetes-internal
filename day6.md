- service is an another layer above deployments so that app could be available to user.
- If we try to pull image from any repository then we must specify tag, in case if tag is not specified then image would get downloaded with tag 'latest' and if there is not image with latest then you would get error.
- internal vs external ips: ## Internal vs. External IP Addresses
The difference between internal and external IP addresses comes down to scope and visibility:

* Internal IP Address (Private IP): This is an address used inside a private network (like your home Wi-Fi, an office building, or a Kubernetes cluster node network). These addresses are hidden from the outside world. Devices inside the network use them to talk to each other. They cannot be reached directly from the internet.
* External IP Address (Public IP): This is a globally unique address assigned to a device connected to the public internet (like a router, a cloud load balancer, or a public website). Anyone on the internet can attempt to route traffic to an external IP address.

------------------------------
## What is RFC 1918?
RFC 1918 (Request for Comments 1918) is the official global internet standard that reserves specific blocks of IPv4 addresses exclusively for private (internal) networks.
Before this standard was created in 1996, every single connected device needed a unique public IP address. As the internet grew rapidly, the world began running out of IPv4 addresses. RFC 1918 solved this by setting aside three distinct blocks of addresses that internet routers are strictly forbidden from routing publicly.
## The Three RFC 1918 Private Ranges
If you check your computer's IP address, your home router, or your Kubernetes node IPs, they will almost always fall into one of these three blocks:

| Block Range | Total IP Addresses | Common Use Case |
|---|---|---|
| 10.0.0.0 to 10.255.255.255 | ~16.7 Million (10.0.0.0/8) | Large enterprise networks, Cloud VPCs (AWS/GCP), and Kubernetes Pod/Service networks. |
| 172.16.0.0 to 172.31.255.255 | ~1.04 Million (172.16.0.0/12) | Medium-sized corporate networks, Docker default bridge networks, and local kind nodes. |
| 192.168.0.0 to 192.168.255.255 | ~65,536 (192.168.0.0/16) | Home Wi-Fi routers and small office networks (e.g., 192.168.1.1). |

------------------------------
## How they work together: The Analogy 🏢
Think of a large office building:

* Internal IPs (RFC 1918): These are like the internal desk extension numbers (e.g., Extension 402). Anyone inside the building can dial Extension 402 to talk to you. However, a person standing out on the street cannot pick up their mobile phone and dial "402" to reach your desk.
* External IP: This is the main corporate phone number of the building printed on the billboard outside.

To bridge the gap, routers use a technology called NAT (Network Address Translation). When your laptop (with a private IP like 192.168.1.5) requests a webpage, your home router swaps your private IP out for its own single public External IP, fetches the data from the internet, and hands it back to your laptop.

- Types of Services: ClusterIp, Load balancer, Node port, Headless, External Name
  
- ClusterIP (The Default): This exposes the Service on an internal IP address within the cluster. It makes the Service only reachable from inside the cluster.
  Best for: Internal backend microservices, databases, or caching layers that should never be exposed directly to users.
  How it works: Pod A talks to Pod B using the stable cluster-internal IP or DNS name.

- Nodeport: This opens a specific, identical port on every single Node (machine) in your cluster and forwards traffic arriving at that port to your Service. Every Kubernetes cluster is made up of one or more physical or virtual machines called Nodes.When you create a NodePort service, Kubernetes reserves a specific port number from a dedicated range (30000 to 32767) and opens that exact same port on every single Node in your cluster.Any network traffic that hits any node's IP address on that specific port is automatically caught and forwarded directly into your Pods.
  A user must be in the "same network" to access a NodePort service, it means the user's device must have direct network routing to the internal IP addresses of your Kubernetes cluster nodes. If there is a firewall, router, or public internet boundary separating the user from the nodes, the connection will fail.
  [ Public Internet User ] 
       │ 
       ▼ 
┌───────────────────────────────────────┐
│  VPC / Company Firewall Boundary      │
│                                       │
│  [ Internal VM ] ───(Can Access!)──┐  │
│                                    ▼  │
│                       [ K8s Node IP: 10.0.0.5 ]
│                          └── Port: 32123 (NodePort)
└───────────────────────────────────────┘

  Best for: Quick development testing, or when you want to route external traffic to your cluster without paying for a cloud provider's load balancer.
  How it works: Kubernetes allocates a random port in the default range 30000–32767. You can access your application by hitting http://<Any-Node-IP>:<NodePort>.
  apiVersion: v1
kind: Service
metadata:
  name: frontend-nodeport
spec:
  type: NodePort                # 🎯 Tells K8s to open a port on all nodes
  selector:
    app: webserver              # Finds your Pods with this label
  ports:
    - protocol: TCP
      port: 80                  # The port used INSIDE the cluster (ClusterIP)
      targetPort: 80            # The port your application is listening on inside the container
      nodePort: 31050           # 🚪 Optional: The exact external port to open (30000-32767)
      
[ Your Laptop Browser ]
         │
         ├───► Hits http://172.18.0.2:31050 (Node 1) ──┐
         │                                             ▼
         └───► Hits http://172.18.0.3:31050 (Node 2) ──┼──► [ Kubernetes Routing ]
                                                       │            │
                                                       │            ▼
                                                       └────► [ Your Pod ] (Port 80)


  
- Load Balancer: This integrates directly with a cloud provider's infrastructure (like AWS, GCP, or Azure) to automatically spin up a physical, external load balancer machine.
  When you set a service type to LoadBalancer, Kubernetes automatically reaches out to your cloud provider's API and rents a physical, hardware network router outside of your cluster.
  Best for: Exposing production-grade web applications or public APIs directly to the internet.
  How it works: nce applied in a cloud environment, run kubectl get svc. You will see:PORT: 80EXTERNAL-IP: 203.0.113.45 (A real, public external IP address)Now, a user sitting at home can type http://203.0.113.45 into their web browser. The traffic hits the cloud's external router, which safely tunnels the request straight into your cluster's private Pod network. The cloud provider provisions a dedicated external public IP address. It automatically routes internet traffic straight down into your cluster's NodePorts and Pods. O
  Note for kind: Because kind runs locally inside Docker, a LoadBalancer service will sit in a permanent <pending> state unless you install a local tool like MetalLB to simulate a cloud provider.

- External Name: This acts as a simple internal alias/shortcut that points your internal pods to an external, real-world domain name outside the cluster. an ExternalName service does not use selectors and does not point to Pods. Instead, it acts as an internal redirect or alias to an external website or third-party server outside your cluster.
  Best for: When code running inside your cluster needs to securely talk to an external third-party API or database (like an AWS RDS database or Twilio API).
  How it works: Instead of hardcoding ://amazonaws.com in your app's configuration, you map it to a local service name. It handles the redirection at the DNS level without proxying any network traffic.
  Suppose your backend Pod needs to talk to a managed database hosted outside your cluster on AWS RDS, with a long address like ://amazonaws.com.Instead of hardcoding that long address inside your application's source code, you map it to a short internal service name like local-db. If the database address changes in the future, you only update this one YAML file—you never have to rebuild or recompile your app code.
  apiVersion: v1
kind: Service
metadata:
  name: local-db                # 📦 The short internal nickname
spec:
  type: ExternalName            # 🎯 Tells K8s this points outside the cluster
  externalName: ://amazonaws.com # 🌍 The real external address
  Inside your container application, you tell your database client configuration to connect to http://local-db.Kubernetes network components immediately intercept that request and substitute the real destination, making the connection seamless.

- Headless Service (The Custom Routing Service): A standard Service acts as a single proxy. It gets an internal cluster IP, and traffic hits that IP before being distributed to random Pods.
  A Headless Service is a service where you explicitly set clusterIP: None. It has no single IP address. Instead of routing traffic, it simply returns the exact, direct internal IP addresses of all matching Pods simultaneously. 
  When do you use it?You use it for stateful applications like databases (MongoDB, PostgreSQL, or Kafka). In a database cluster, you cannot talk to just a random Pod; you need to write data specifically to the Primary/Master node and read data from the Secondary/Replica nodes. Your application needs to know exactly who is who.
apiVersion: v1
kind: Service
metadata:
  name: database-headless
spec:
  clusterIP: None               # 🎯 THIS makes it a Headless Service!
  selector:
    app: mysql
  ports:
    - port: 3306

 How it behaves in action: When a backend application looks up database-headless in the cluster's network, instead of getting one service IP, the internal DNS returns a list:
 10.244.0.5 (MySQL-0 / Master)
 10.244.1.9 (MySQL-1 / Replica)
  The application can now choose to connect directly to the Master node to perform database queries.


------

- Endpoint: Automatically get created when new slice is created
- Endpoint Slice








  
