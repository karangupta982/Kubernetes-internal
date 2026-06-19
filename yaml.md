# Lecture 1 Notes — YAML, Linux Basics, and Kubernetes Intro

## 1) First Principle: Data ko store / send kaise karte hain?

### Problem

Computer ke andar data normally **memory** mein hota hai.
Lekin jab data ko:

* file mein save karna ho
* network ke through bhejna ho
* kisi dusre system ko dena ho

to usko ek **standard format** mein convert karna padta hai.

### Example

Maan lo tumhare paas ye data hai:

```text
name = Karan
age = 22
skills = MERN, Python
```

Ye memory mein structured data hai.
Ab agar isko file mein save karna hai ya API se bhejna hai, to isko kisi readable / transferable format mein convert karte hain.

---

## 2) Serialization aur Deserialization

### Serialization kya hota hai?

Serialization ka matlab hai:

> In-memory structured data ko ek aise format mein convert karna jo store ya transmit kiya ja sake.

Ye format text bhi ho sakta hai, binary bhi ho sakta hai.

### Deserialization kya hota hai?

Deserialization ka matlab hai:

> Stored/transmitted format ko wapas structured object mein convert karna.

### Simple flow

```text
Object in memory
    ↓ serialize
Text / binary format
    ↓ store or send
File / network
    ↓ deserialize
Object again
```

### Important correction

Tumne jo “serialization = converting objects into bytes” bola, woh **half-correct** hai.

Better way to understand it:

* Serialization = structured data ko portable format mein convert karna
* Uska end result usually bytes hota hai, kyunki disk aur network bytes hi handle karte hain
* YAML/JSON jaise formats **textual serialized forms** hain

Matlab:

* `YAML` khud ek **serialized text representation** hai
* Computer file mein ise finally bytes ke roop mein hi store karta hai, but human ke liye ye text hota hai

---

## 3) YAML kya hai?

### Full form

**YAML = YAML Ain’t Markup Language**

**Kubernetes API server is incredibly strict about how it parses YAML**

### Matlab kya hai?

YAML koi programming language nahi hai.
Ye ek **data representation / serialization format** hai.

Isko isliye use karte hain kyunki:

* human-readable hota hai
* configuration files ke liye clean hota hai
* nested data ko padhna easy hota hai

---

## 4) YAML kyu use hota hai?

Kubernetes, Docker Compose, CI/CD, Ansible jaise systems mein configuration deni hoti hai.

Example:

* kis image ko run karna hai
* kitne replicas chahiye
* environment variables kya hain
* ports kaunse open hone chahiye

Ye sab **configuration** hai, code nahi.

YAML isliye use hota hai kyunki config ko readable aur manageable banata hai.

---

## 5) YAML, JSON, XML, Protobuf — difference ka practical sense

### YAML

* human-friendly
* indentation-based
* config files ke liye best

### JSON

* structured
* machine-friendly + readable
* YAML se stricter hota hai
* valid JSON, YAML 1.2 mein valid hota hai

### XML

* verbose
* tags zyada hote hain
* readable but heavy

### Protobuf

* machine-efficient
* fast
* compact
* human readability important nahi hoti

### Real-world thinking

* humans ke liye: YAML / JSON
* machines ke liye fast exchange: Protobuf
* old enterprise systems: XML

---

## 6) YAML ki core properties

### 6.1 YAML case-sensitive hota hai

```yaml
name: Karan
Name: Prerit
```

Yahan `name` aur `Name` different keys hain.

---

### 6.2 YAML key-value pair format follow karta hai

```yaml
name: Karan
age: 22
```

Yahan:

* `name` = key
* `Karan` = value

---

### 6.3 Keys unique hone chahiye

Ek key ko same level par dobara use nahi karna chahiye.

Galat:

```yaml
name: Karan
name: Prerit
```

---

### 6.4 Space important hai

YAML mein `:` ke baad **space** hona chahiye.

Valid:

```yaml
name: Karan
```

Problematic / wrong:

```yaml
name:Karan
```

Isliye YAML mein **space discipline** bahut important hai.

---

### 6.5 Tabs use nahi karne chahiye

YAML indentation ke liye **spaces** use karta hai, tabs nahi.

Agar tabs use karoge to parser confuse ho sakta hai.

---

### 6.6 Indentation ka matlab hierarchy hota hai

Indentation se pata chalta hai kaun kis ke andar hai.

Example:

```yaml
person:
  name: Karan
  age: 22
```

Yahan `name` aur `age` `person` ke child keys hain.

---

## 7) YAML mein list kya hoti hai?

Tumhara hyphen wala doubt important hai.

### Hyphen `-` ka matlab:

YAML mein hyphen ka matlab hota hai:

> list ka item

Example:

```yaml
fruits:
  - apple
  - mango
  - banana
```

Iska matlab:

```text
fruits = [apple, mango, banana]
```

### Yaad rakhne layak baat

* `:` = key-value relation
* `-` = list item
* indentation = nesting / structure

In Kubernetes manifests, you rarely see a simple list of strings. Instead, you see a List of Objects (Key-Value maps).
```yaml
spec:
  containers:
    - name: nginx-container
      image: nginx:latest
      ports:
        - containerPort: 80
    - name: sidecar-container
      image: fluentd:latest
```

How to read this like a parser:
containers: tells the system: "An array/list of containers is starting here."

The first - name: nginx-container means: "This is the first item in the list, and this item is an object. The object starts with the key name."

Because image: and ports: are aligned horizontally right under name (without a new hyphen), they belong to that same first object.

The second - name: sidecar-container has a hyphen aligned with the first one. This means: "The first object is finished. This is the second item in the containers list."
      
---

2. Critical Missing YAML Features for K8s Exams
Kubernetes manifests use advanced YAML features extensively, especially for managing configuration data (ConfigMaps and Secrets). These are frequently tested in certifications.

A. Multiline Strings: Literal Block Scalar (|) vs. Folded Scalar (>)
When passing scripts, configuration files, or certificates inside a YAML, you cannot just write multiple lines. You must use specific indicators:

Literal Indicator (|): Preserves newlines and formatting exactly as written. (Crucial for embedded scripts or Nginx configs).

Folded Indicator (>): Replaces newlines with spaces, converting a multi-line paragraph into a single long string.

YAML
apiVersion: v1
kind: ConfigMap
metadata:
  name: special-configs
data:
  # Preserves newlines exactly as shown
  script.sh: |
    #!/bin/bash
    echo "Starting process..."
    systemctl restart nginx
  
  # Will turn into: "This is a very long sentence compiled into one line."
  one-line-string: >
    This is a very
    long sentence compiled
    into one line.
B. Anchors (&) and Aliases (*) for DRY (Don't Repeat Yourself)
If you have repetitive blocks (like identical resource limits across multiple containers), you can use anchors to define a block and aliases to inject it.

YAML
# Define a reusable block using '&'     (didn't understand this, what is the exact syntax and what it is doing)
helpers: &resourceLimits
  resources:
    limits:
      cpu: "200m"
      memory: "512Mi"

spec:
  containers:
    - name: app
      image: app:v1
      <<: *resourceLimits # Inject the block using '*'   (didn't understand this, what is the exact syntax and what it is doing)
      
3. Rectifying and Deepening the Linux Section
For CKA/CKS, you will be SSH-ing into broken cluster nodes. Your Linux understanding needs to map directly to control plane components.

A. What exactly is systemd and systemt?
In your notes, you mentioned "systemd linux ki sari commands chalata h, but what is systemd" and "system etc ke ander store hota h". Let's clear up the terminology:

systemd is the Init System (PID 1) for Linux. It is the very first process that starts when the machine boots up. Its job is to bootstrap the entire operating system and manage background services (daemons).

In a Kubernetes cluster, the kubelet daemon and your Container Runtime (containerd) run as systemd services. If a node shows a NotReady status, your first debugging step is checking systemd.

The path configuration: Service configuration files are stored in /etc/systemd/system/ (not system etc).

Bash
# Critical CKA Commands to check if K8s is alive on a node:
sudo systemctl status kubelet        # Check if kubelet is running
sudo systemctl restart kubelet       # Restart if it crashes or freezes
B. Advanced Logging with journalctl
Since kubelet runs natively on the host OS via systemd (not inside a container), you cannot run kubectl logs to debug it. You must use journalctl.

Bash
# View kubelet logs chronologically
sudo journalctl -u kubelet

# View only the most recent logs with real-time streaming (Crucial for live debugging)
sudo journalctl -u kubelet -f

# Filter out only error-level messages
sudo journalctl -u kubelet -p err
C. Container Runtime Debugging (crictl vs docker)
Modern Kubernetes clusters do not use Docker directly; they use lightweight runtimes via the Container Runtime Interface (CRI), usually containerd.
Because Docker commands (docker ps) won't work on these nodes, you use crictl.

⚠️ Exam Gotcha: crictl requires a configuration file at /etc/crictl.yaml pointing to the runtime endpoint (unix:///var/run/containerd/containerd.sock). If it doesn't work during the exam, verify this file exists!

4. Up-to-Date Kubernetes Facts for 2026
Release Cycle: You noted that Kubernetes releases 2 times a year. Correction: Kubernetes shifted its release cadence a while ago to 3 releases per year (approximately every 4 months).

Current Version (2026): As of mid-2026, the Kubernetes community is tracking around v1.33 / v1.34. Make sure to check the exact documentation version allowed in your upcoming exam environment, as minor syntax details can shift.


---

## 8) YAML list of objects

```yaml
students:
  - name: Karan
    age: 22
  - name: Prerit
    age: 25
```

Ye list hai jisme har item ek object hai.

Socho:

* `students` = list
* har `-` = ek student object

---

## 9) Quotes kab lagte hain?

### Generally quotes mandatory nahi hote

```yaml
name: Karan
```

Valid hai.

### Quotes use karo jab:

* special characters ho
* string ke andar unusual characters ho
* parser confusion ka risk ho

Example:

```yaml
name: "Karan Gupta"
```

Single quote bhi use ho sakti hai:

```yaml
name: 'Karan Gupta'
```

---

## 10) YAML khud datatype define nahi karwata

YAML ko tum ye nahi bolte:

* ye integer hai
* ye float hai
* ye boolean hai

Parser aksar value dekh kar type infer karta hai.

Example:

```yaml
age: 22
price: 99.5
active: true
```

Yahan parser samajh lega:

* `22` = integer
* `99.5` = float
* `true` = boolean

---

## 11) YAML file extensions

Dono common hain:

* `.yaml`
* `.yml`

Dono valid hain.
Industry mein dono mil jayenge.

---

## 12) Multiple YAML documents in one file

Agar ek hi file mein multiple YAML documents rakhne hon, to `---` use hota hai.

Example:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
---
apiVersion: v1
kind: Service
metadata:
  name: my-service
```

### `---` ka matlab

Naya YAML document start ho raha hai.

Kubernetes manifests mein ye bahut common hai.

---

## 13) YAML comments

Comment likhne ke liye `#` use hota hai.

```yaml
# This is a comment
name: Karan
```

---

## 14) CamelCase kya hota hai?

Example:

```text
containerName
podName
imagePullPolicy
```

Isme:

* first word small
* next word ka first letter capital

Yeh naming convention hai.
Kubernetes mein bahut jagah camelCase use hota hai, but ye YAML ka rule nahi hai.
Ye sirf convention hai.

---

## 15) YAML aur Kubernetes ka relation

Kubernetes manifests usually YAML mein likhe jaate hain.

Typical Kubernetes YAML structure:

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: my-pod
spec:
  containers:
    - name: nginx
      image: nginx:latest
```

### Iska rough meaning

* `apiVersion` = kis Kubernetes API version ka object hai
* `kind` = object ka type kya hai
* `metadata` = name, labels, etc.
* `spec` = actual desired state

### First-principle thought

Kubernetes YAML ko isliye use karta hai kyunki:

* human easily read kar sake
* machine API server ko structured object de sake

Internally Kubernetes is data ko JSON-like structured object mein process karta hai.

---

# 16) Beginner bridge: YAML aur Kubernetes mein gap kaise samjho?

Tumhara confusion yahin hai:

> “Serialization” aur “configuration file” ka connection kya hai?

### Seedha answer:

Kubernetes ko sirf data chahiye:

* kya banana hai
* kaise banana hai
* kitna banana hai

YAML woh data dene ka **human-readable format** hai.

### Example

Tum likhte ho:

```yaml
replicas: 3
```

Iska matlab:

* 3 copies run karni hain

YAML sirf yeh batata hai.
Kubernetes usko read karke actual pods banata hai.

---

# 17) Linux basics for Kubernetes

Kubernetes environment mein Linux basics bahut kaam aate hain.

---

## `ls -a`

Hidden files bhi dikhata hai.

```bash
ls -a
```

Use case:

* `.env`
* `.git`
* hidden config files

---

## `ps aux`

Running processes list karta hai.

```bash
ps aux
```

Use case:

* kaunsa process chal raha hai
* CPU/memory use dekhna

---

## `pstree`

Process hierarchy dikhata hai.

```bash
pstree
```

Use case:

* parent-child process relation

---

## `base64`

Kubernetes secrets mein base64 encoding commonly hoti hai.

### Encode

```bash
echo karan | base64
```

### Decode

```bash
echo a2FyYW4K | base64 --decode
```

### Important clarity

Base64 **encryption nahi hai**.
Ye sirf encoding hai.
Koi bhi decode kar sakta hai.

---

## `head`

File ki top lines dikhata hai.

```bash
head file.txt
```

Usually first 10 lines.

Use case:

* config file ka start dekhna

---

## `tail`

File ki last lines dikhata hai.

```bash
tail log.txt
```

Use case:

* logs dekhna
* latest entries track karna

---

## `Ctrl + C`

Running command ko stop karta hai.

Use case:

* stuck command se nikalna

---

## `tcpdump`

Network packet capture tool.

Use case:

* network debugging
* DNS issues
* packet flow dekhna

Beginner ko poora use abhi nahi samajh aayega, but yaad rakho:

> Kubernetes networking troubleshoot karne mein kaam aata hai.

---

## `crictl`

CRI runtime se directly baat karne ke kaam aata hai.

### CRI kya hai?

**Container Runtime Interface**

Ye Kubernetes aur container runtime ke beech ka interface hai.

Examples:

* containerd
* CRI-O

### Useful `crictl` commands

```bash
crictl pods
crictl ps -a
crictl logs <container-id>
```

Use case:

* pod/container level debugging
* kubelet ke neeche kya chal raha hai dekhna

---

## `systemd`

Systemd Linux ka service manager hai.

Ye boot ke time services start/stop/manage karta hai.

Examples:

* kubelet
* container runtime
* ssh
* nginx

### Useful command

```bash
sudo systemctl status kubelet
```

### Why important?

Kubernetes nodes par kubelet aksar system service ke form mein run hota hai.

---

## `journalctl`

System logs dekhne ke kaam aata hai.

### Examples

```bash
sudo journalctl
sudo journalctl --since yesterday
sudo journalctl --since yesterday -o json-pretty
```

### Use case

* service logs dekhna
* past logs inspect karna
* structured log format lena

---

# 18) Kubernetes basics

## Kubernetes kya hai?

Kubernetes ek **container orchestration system** hai.

Simple meaning:

> Multiple containers ko manage, schedule, scale, restart, aur organize karne wala system.

---

## Kubernetes ki need kyu padi?

Agar tumhare paas sirf Docker hai, to tum image bana sakte ho aur container run kar sakte ho.
Lekin production mein aur chahiye:

* restart policy
* auto healing
* scaling
* service discovery
* load balancing
* rollout / rollback
* scheduling
* resource management

Kubernetes ye sab manage karta hai.

---

## Kubernetes hierarchy

### Cluster

Complete Kubernetes environment.

### Node

Ek machine — VM ya physical server.

### Pod

Smallest deployable unit.

### Container

Actual application process.

### Simple flow

```text
Cluster
  └── Node
        └── Pod
              └── Container
```

### Important clarity

Pod ke andar:

* ek container ho sakta hai
* multiple containers bhi ho sakte hain

But practical life mein mostly ek main container hota hai.

---

## Pod kya hai?

Pod container ka wrapper hai.

Pod ka role:

* containers ko ek saath run karna
* same network namespace share karna
* same lifecycle share karna

Pod = Kubernetes ki basic running unit.

---

# 19) Lecture mein jo important mental model tha

## From big to small

* Cluster = whole system
* Node = machine
* Pod = deployable wrapper
* Container = actual app

Ye hierarchy yaad rakhni hai.

---

# 20) Lecture 1 ka core summary

Agar ek line mein bolna ho to:

> YAML K8s manifests likhne ka human-readable format hai, Linux commands debugging ke kaam aati hain, aur Kubernetes cluster-node-pod-container hierarchy par based hota hai.

---

# 21) Beginner ke liye yaad rakhne wali “golden lines”

* YAML = data likhne ka readable format
* Serialization = structured data ko portable form mein convert karna
* Deserialization = portable form ko wapas object banana
* `:` = key-value
* `-` = list item
* indentation = nesting
* `#` = comment
* `---` = new YAML document
* base64 = encoding, not encryption
* systemd = Linux service manager
* journalctl = logs
* crictl = container runtime debugging
* Pod = smallest deployable unit
---

Real talk:

Ye dono concepts Kubernetes ke sabse important concepts me se hain. Agar ye samajh gaye to Pod ka purpose samajh aa jayega.

---

# 1. Network Namespace Kya Hota Hai?

Pehle namespace samajho.

Linux me namespace ka matlab hota hai:

> Ek isolated view of system resources.

Simple language me:

Socho ek bade office me 100 log hain.

Har bande ko ek private cabin de diya gaya.

Ab cabin ke andar:

* uska apna table
* apna chair
* apna whiteboard

hai.

Bahar same office hai, lekin andar uska private environment hai.

Linux namespaces bhi aisa hi karte hain.

---

## Container ke case me

Jab container banta hai to usko apna:

* Network
* Process IDs
* Hostname
* Filesystem

mil sakta hai.

Example:

Container A:

```text
IP = 10.0.0.1
Port = 8080
Hostname = app1
```

Container B:

```text
IP = 10.0.0.2
Port = 8080
Hostname = app2
```

Dono same machine pe chal rahe hain.

Lekin dono ko lagta hai ki unka khud ka network hai.

Ye network namespace ki wajah se possible hai.

---

# Pod ke andar kya hota hai?

Yahan Kubernetes twist aata hai.

Agar Pod me multiple containers hain:

```text
Pod
├── Container A
└── Container B
```

to ye dono SAME network namespace share karte hain.

Matlab:

```text
Pod IP = 10.244.1.15
```

Ab:

Container A:

```text
IP = 10.244.1.15
```

Container B:

```text
IP = 10.244.1.15
```

Same IP.

---

## Iska practical meaning kya hai?

Container A:

```text
Application
Port 8080
```

Container B:

```text
Logging Agent
```

Ab Logging Agent simply:

```text
localhost:8080
```

par baat kar sakta hai.

Koi network setup nahi karna.

Kyun?

Kyuki dono same network namespace me hain.

---

## Real-world example

```text
Pod
├── Nginx Container
└── Log Collector Container
```

Nginx:

```text
localhost:80
```

Log Collector:

```text
localhost:80
```

Dono ek dusre ko localhost se access kar sakte hain.

Yahi "same network namespace share karna" ka matlab hai.

---

# 2. Same Lifecycle Share Karna

Ye aur bhi important hai.

Lifecycle ka matlab:

> Container kab create hoga, start hoga, restart hoga, delete hoga.

---

## Example

Pod:

```text
Pod
├── Main App Container
└── Sidecar Container
```

---

### Pod create hua

To:

```text
Main App Start
Sidecar Start
```

---

### Pod delete hua

To:

```text
Main App Stop
Sidecar Stop
```

Dono saath me marenge.

---

### Pod reschedule hua

Node A:

```text
Pod Running
```

Node A crash ho gaya.

Kubernetes:

```text
New Pod Create on Node B
```

Ab:

```text
Main App Start
Sidecar Start
```

Dono fir saath me start honge.

---

## Important Point

Kubernetes individual container ko manage nahi karta.

Kubernetes:

```text
Pod
```

ko manage karta hai.

Isliye lifecycle Pod level par hoti hai.

---

# Real Example

Maan lo tumhari application hai:

```text
Pod
├── Spring Boot App
└── Monitoring Agent
```

Tumhara Spring Boot app crash ho gaya.

Pod restart hua.

Kya hoga?

```text
Spring Boot Restart
Monitoring Agent Restart
```

Dono restart.

Kyun?

Same Pod = Same Lifecycle.

---

# Why Pod Exists?

Bahut log sochte hain:

> "Container hi run kara dete, Pod ki kya zarurat thi?"

Google ne Pod isliye introduce kiya:

Kuch containers naturally ek saath kaam karte hain.

Example:

```text
Main Application
+
Log Collector
```

Ya

```text
Main Application
+
Metrics Exporter
```

Ye logically ek unit hain.

Isliye Kubernetes kehta hai:

> "Inko ek Pod me rakho."

Aur Pod provide karta hai:

### Shared Network

```text
Same IP
Same localhost
```

### Shared Lifecycle

```text
Together Start
Together Stop
Together Move
Together Die
```

---

# Interview Answer

### What is Network Namespace?

> Linux network namespace provides an isolated networking stack. Containers inside the same Pod share the same network namespace, therefore they share the same IP address and can communicate using localhost.

### What is Shared Lifecycle?

> Containers inside a Pod are managed as a single unit. They are scheduled, started, stopped, restarted and deleted together because Kubernetes manages the Pod rather than individual containers.

---


# These are the notes i made from lecture
# YAML
- Yaml is case sensitive.
- Data -> Byte stream + conversion content to gain the exact format again -> serialization
- Yaml has :, -, space, english
- In place of yaml, json, protobuf and xml can be used.
- Yaml is used only because it is human readable configuration and data storage.
- yaml is superset of json.
- camelcase: karanGupta (Generally in kubernetes camelCasing is used).
- YAML = YAML Ain't Markup language. It is not a language but a declarative way to define things. It is not needed to tell yaml that store it in int, float, double. It is capable enough to detect that.
- name:"karan" , double quotes are not needed in yaml until you are using special characters like n, &.
- Yaml is a key-value pair, Key is unique but value may be infinite. For example name:Karan, now name can't be used one more time, Name:Karan can be used since yaml is case sensitive.
- we can store yaml as .yaml and .yml both ways.
- --- is used when we have multiple yamls in a single file.
- hyphen means it is a list, still i didn't understand what exactly is mean by hyphen.
- Comments are added using #.


  # LINUX for kubernetes point of view
  - ls -a : shows hidden files
  - ps aux: list down running process
  - ps tree: gives hierarchical view of folder
  - base64 format is used in kubernetes secrets, conversion to base64, echo karan | base64  -> it will return base64 encoding of karan and conversion from base64 to string:
   for example: echo karan | base64 response is cHJ1cm10Cg==
   then echo cHJ1cm10Cg== | base64 --decode  will decode back.
 
  - head to see top 10 lines in any configuration file
  - tail is used to see last 10 lines in any log file
  - ctrl + c is exit in linux
  - tcpdump commands
  - crictl : to directly talk with pods
  - crictl pods, crictl ps -a, sudo crictl logs id
  - systemd linux ki sari commands chalata h, but what is systemd
  - system etc ke ander store hota h
  - sudo journalctl --no-pager, sudo journalctl --since yesterday
  - -o means output
  - sudo journalctl --since yesterday -o json-pretty
 

  # Kubernetes
  - kubernetes releases two times a year (one release cycle is of four months)
  - Always remember the current version of kubernetes, important for inerview.
  - cluster -> Node -> Pod -> Container






