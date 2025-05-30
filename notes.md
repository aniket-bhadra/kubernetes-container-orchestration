If I use AWS ECS for container orchestration, migrating to another cloud provider like GCP or DigitalOcean later may be difficult, as I might need to rewrite the deployment code for each provider. However, Kubernetes offers a common interface for container orchestration, making it cloud-agnostic. This allows me to run and migrate my applications across different cloud providers with minimal changes.

✅ Kubernetes provides: Container orchestration + a unified, portable interface across clouds.

The Core Problems Kubernetes Solves:

*Auto Scale Up / Down & Self-Healing:*
If a container dies or load increases, Kubernetes automatically replaces crashed containers, adds more instances (scale up), or removes extras (scale down).

*Common Interface (Cloud Agnostic):*
Provides a consistent way to deploy and manage containers across different cloud providers or on-premises, making your app portable with no cloud lock-in.

*Container Orchestration: Manages scheduling, networking, and lifecycle of containers automatically.*

## 🚀 Kubernetes Architecture (Super Crisp Flow)

### 1️⃣ **Control Plane** (The *Brain* of Kubernetes)  
*(Physical/Virtual Machines — Manages the Cluster)*  

- **Control Plane = Admin layer (makes global decisions).**  
- **Key Components:**  

  - **API Server (kube-apiserver)**  
    → **Entry point** for all commands (users/kubectl/components talk to it).  
    → **Validates** requests, updates **etcd**, and **notifies** other components via **watches**.  
    → **Only component** that directly talks to **etcd**.  

  - **Controller Manager (kube-controller-manager)**  
    → Runs **control loops** (e.g., ReplicaSet, Deployment, Node controllers).  
    → **Watches API Server** for changes (e.g., "Desired vs. Actual State").  
    → Triggers actions (e.g., "Scale up Pods if needed").  
    it also tracks what happening in the cluster, if pod dies or pods needs to created or not 

  - **etcd**  
    → **Key-value store** (Kubernetes’ database).  
    → etcd stores both the desired state (what you declared in manifests like    Deployments) and the current state (real-time status of pods, nodes, configs, etc.). It’s the cluster’s source of truth—controllers use it to detect and fix drift
    → **Only the API Server** reads/writes to it.  


  - **Scheduler (kube-scheduler)**  
    → **Watches unscheduled Pods** (`spec.nodeName == ""`).  
    → Decides **which Worker Node** a Pod should run on (based on resources, labels, etc.).  

---

### 2️⃣ **Worker Node** (Where Containers Actually Run)  
*(Physical/Virtual Machines — Runs Your Apps)*  

- **Worker Node = Executes workloads (Pods/Containers).**  
- **Minimum 2 Nodes** (for high availability; scales infinitely).  
- **Key Components per Node:**  

  - **Kubelet**  
    → **Agent** that talks to the API Server.  
    → **Manages Pods** on its node (creates/deletes/stops containers).  
    → **Watches Pods assigned to its Node** (gets updates from API Server).  

  - **Kube Proxy**  
    → Handles **networking rules** (IP forwarding, load balancing).  
    → Ensures Pods can talk to each other/services.  

  - **Container Runtime Interface (CRI)**  
    → Runs the **actual containers** (Docker, containerd, cri-o).  
    → **Kubelet** instructs the CRI (e.g., "Start this container").  

  - **Pods**: Wrap around one or more containers; they share:
  ✅ Storage
  ✅ Network
  ✅ Lifecycle
---  
# 🔁 What Happens Under the Hood:
## kubernetes Watch mechanism
1. **Components (like Controller Manager, Scheduler, Kubelet)**
   → Open a **long-lived HTTP connection** to the API Server using `?watch=true`.
2. **API Server** Maintains these open **watch connections** .

3. **API Server updates etcd** (example: a new Pod is added, deleted, or modified)

4. **API Server then checks:**
   “Do I have any watch connections interested in this type of change?”

5. **If yes**
   → API Server **sends an response with event** (like `ADDED`, `MODIFIED`, or `DELETED`)
   → **Only to the specific component(s)** watching that resource type.

   *etcd changes → API Server is the only one directly reading/writing etcd.*
---

🔹 When the API Server responds with an event:
It sends the event type: ADDED, MODIFIED, or DELETED.

Along with the full latest object state from etcd.

Scheduler: Gets an ADDED event with Pod object in Pending state (from etcd).

Controller Manager: Gets events like ADDED, MODIFIED, or DELETED with actual state, so it can compare to desired state.

Kubelet: Gets a MODIFIED or DELETED event for Pods assigned to it, including current config/image/etc.

The structure ALWAYS remains the same whenever API Server responds with events to any component:
json{
  "type": "ADDED",     // Event type
  "object": { ... } 
                      // Full resource object,object contains the latest etcd state   (not a diff).
}

Each component only gets updates for the specific resources it watches.The API Server filters events, so components don’t see unrelated changes.
Components also handle connection failures by re-establishing watches and doing a fresh LIST to catch up on missed events.


### Step-by-step simplified flow — you say: “Run 2 nginx pods”

1. **You send request** to API Server: “Run 2 nginx pods.”

2. API Server **authenticates** → If OK, API Server **updates etcd** with desired state = 2 nginx pods.

3. **Controller Manager** has a **watch connection open to API Server** (not direct to etcd).

   * API Server streams changes from etcd to Controller Manager.
   * Controller Manager sees: *“Desired = 2 pods, Current = 0 pods” → mismatch!*

4. Controller Manager decides: *“I need to create 2 pods”* → it tells API Server to create Pod objects.

Controller Manager = decides and requests Pod creation.

API Server = actually creates the Pods in etcd.because The API Server is the only component that can write to etcd.

5. API Server updates etcd with these Pod objects in **Pending** state.

6. **Scheduler** has a watch on API Server too → notices 2 new Pods pending (no node assigned).
1️⃣ When the API Server creates a Pod, it writes the Pod object to etcd with status Pending (because it’s created but not yet assigned to any node).

2️⃣ The Scheduler is watching the API Server 

3️⃣ When the Pod object with status Pending appears in etcd, API Server notifies the Scheduler about this new Pod

7. Scheduler assigns each Pod to suitable Worker Node → API Server updates Pod spec with assigned node.

8. **Kubelet** on each Worker Node is also watching API Server → sees Pod assigned to its node.

Scheduler assigns Pod to a Worker Node →

API Server updates the Pod object in etcd with that node assignment →

API Server notifies all Kubelets (each watching API Server for changes) about the new Pod assignments →

The Kubelet on the assigned node sees its new Pod →

That Kubelet contacts API Server to get full Pod specs (container image, commands, config, etc.) →

Kubelet uses CRI to pull the image and run the container(s) inside the Pod


---

### Now, if you say: “Run 1 nginx pod” (scale down)

1. You send request → API Server updates desired state in etcd = 1 pod.

2. The **Controller Manager** receives this event and the updated state via the API Server, and notices the mismatch between:

   * Desired state (e.g., 1 Pods)
   * Current state (e.g., 5 pods running)

3. Controller Manager tells API Server to delete extra Pod(s).

4. API Server updates etcd → marks Pod(s) for deletion.
When API Server updates etcd to mark a Pod for deletion:
It sends a response with delete event to the Kubelet on the Worker Node where that Pod is running

The Kubelet sees the Pod deletion request and then stops and removes the Pod’s containers using the container runtime (CRI).
API Server updates etcd → notifies relevant Kubelet → Kubelet deletes Pod

---


When a Pod is created, the API Server sends an ADDED event to the Scheduler containing a Pod in Pending state.When Pod is marked deleted in etcd, the API Server sends a response to Kubelet with event Deleted.
#### Important clarifications:

* Controller Manager and Scheduler **do NOT talk directly to etcd**.
* They watch **API Server**, which is the ONLY component interacting with etcd.
* **API Server acts as a gateway**: stores and streams all changes from etcd to components via watches.

---

#### So:

* The “watching” means: components keep a streaming connection to API Server to get real-time updates.
* API Server reads/writes to etcd and notifies controllers and schedulers immediately.
* Kubelet watches API Server for Pod specs (assignment, creation, deletion).

---

Just remember:
**All communication with etcd is only through API Server. Everyone else watches API Server for changes.**

---


The **API Server** is the central boss that:

* **Updates etcd** with the desired and current states.
* **Sends response with events** to other components watching it.
* Each component (Controller Manager, Scheduler, Kubelets) **watches the API Server** for relevant changes.
* When changes happen, the API Server **pushes those updates to the right components** so they can act:

  * Controller Manager reacts to desired vs current state mismatches.
  * Scheduler reacts to new Pods in Pending state and assigns nodes.
  * Kubelets react to Pod assignment and start or stop containers accordingly.

So yes:

**API Server updates etcd → notifies watchers (Controller, Scheduler, Kubelets) → they act → they tell API Server to update etcd again → cycle continues.**


### **When a Pod or container crashes:**

1. The **Kubelet** on the worker node notices the Pod/container has crashed and updates the **API Server** about the Pod’s status.

2. The **API Server** updates the **etcd** database with the new current state (e.g., Pod is not running), and sends a response with a **`DELETED`** event, along with the latest state showing the mismatch between desired and current.

3. The **Controller Manager** receives this event and the updated state via the API Server, and notices the mismatch between:

   * Desired state (e.g., 5 Pods)
   * Current state (e.g., only 4 Pods running because one crashed)

4. The **Controller Manager** tells the API Server to create a new Pod to replace the crashed one.

5. The **API Server** creates a new Pod object with status **Pending** in **etcd**.

6. After that, the API Server sends a response with an **`ADDED`** event and the Pod object (in Pending state) to the **Scheduler**. The Scheduler notices the new Pending Pod and assigns it to a suitable worker node based on load.

7. The **API Server** updates the Pod object in **etcd** with the assigned node.

8. The **API Server** notifies the **Kubelet** on that worker node about the new Pod assignment.

9. The **Kubelet** asks the API Server for the full Pod specs, container images, etc., and then starts the Pod using the container runtime.

---

### 6️⃣ **Cloud Controller Manager (CCM)** – *Handles cloud-specific stuff*

* Suppose you ask the API Server:

  > “Create 10 Node.js containers and a Load Balancer.”

* Kubernetes can handle the **Node.js Pods** just fine.

* But the **Load Balancer is cloud-specific** →
  API Server forwards that part of the request to the **Cloud Controller Manager (CCM)**.

* **CCM** talks to your **cloud provider’s API** (AWS, GCP, Azure, DigitalOcean, etc.) to:

  * Create a Load Balancer
  * Assign a public IP
  * Attach volumes
  * Manage other cloud-specific resources

---

✅ So, **Cloud Controller Manager** is responsible for:

* Creating, deleting, and managing **cloud-specific infrastructure**
  (like Load Balancers, public IPs, storage, and network routes)

---

🧠 Think of it like this:

> “Kubernetes handles the containers.
> Cloud Controller Manager handles the cloud stuff.”

### ✅ **3. Scheduler = Load Balancer?**

Kind of, yes — but only **for assigning Pods to Nodes**.

* The **Scheduler** looks at:

  * Available CPU/RAM on Nodes
  * Pod requirements (like resources, affinities, tolerations, etc.)

Based on this, it assigns Pods to the most suitable Worker Node.

So:
⚠️ It’s **not** a *network* load balancer (it doesn’t route traffic).
✅ It’s a **workload balancer** (it distributes Pods across Nodes efficiently).

---

✅ **Scheduler’s job:**

* Constantly watches for **Pods in Pending state**
* If a Pod has no Node assigned → **Scheduler steps in**

It then:

1. Checks the **current load** on all Worker Nodes
2. Reads the **Pod's requirements**
3. Assigns the Pod to the **best-fit Node**

---

🧠 Think of it as:

> “Are there any Pods waiting without a home?
> I’ll find the best place for them!”

---
Kubernetes Cluster = Multiple computers (nodes) working together, where some run the Control Plane (the brain) and others are Worker Nodes (running containers). These computers can be real physical machines, virtual machines, or a mix of both, depending on the setup.

Kubernetes Cluster = Teamwork!
A bunch of machines (physical/VMs/both) working together to run Kubernetes:

Control Plane Machines → The "brain" team (API Server, Scheduler, etc.).

Worker Machines → The "muscle" team (run your containers).

Together, they maintain the Kubernetes flow (auto-scaling, healing, etc.).


Minikube = A single computer (your laptop) running a mini Kubernetes cluster (Control Plane + Worker Node together) inside a virtual machine. It’s great for learning and development.

Minikube = One machine (laptop/VMs) running both Control Plane + Worker Node.
so in 1 node both control plane & worker node runs & docker container run time pre installed.

Minikube can simulate multi-node clusters on a single machine.
By default, Minikube starts with 1 node.


Kind = Similar to Minikube, but instead of using a virtual machine, it uses Docker containers on your laptop to create the mini Kubernetes cluster.Kind uses Docker containers to simulate the whole Kubernetes cluster.Each container can be a Control Plane node or a Worker Node.

1+ containers act as Control Plane.
1+ containers act as Worker Nodes.

Super light, fast, but still just for testing.

Example:
1 Control Plane container + 2 Worker Node containers = Mini-cluster!

so kubenetes cluser has ataleas 1 master node + 1 backup of that master node (in case of one down)
couple of worker nodes

Docker Desktop Kubernetes = single-node only.

kubectl-
 kubectl is a CLI tool to talk to any Kubernetes cluster,Whether it’s:
  A Minikube cluster (local, runs on your machine),
  A Docker Desktop Kubernetes cluster (local),
  A cloud Kubernetes cluster (e.g., GKE, EKS, AKS),
    kubectl always talks to the API server of the respective cluster.

The **API server** is the entry point of a Kubernetes cluster. It allows communication with the cluster through various Kubernetes clients such as the **UI (Kubernetes Dashboard)**, **API (used in scripts)**, and **CLI tools**.

The **virtual network inside Kubernetes** enables nodes to communicate with each other, effectively turning all the nodes in the cluster into a single, powerful system.

* **Control plane nodes**: These are more important for managing the cluster but handle less workload and typically have fewer resources.
* **Worker nodes**: These handle the actual workloads, so they usually have more and larger resources.


### Main Kubernetes Components (Explained Simply and to the Point)

* **Node** = a virtual or physical machine inside the Kubernetes cluster.Nodes can be worker nodes (run your apps) or control plane nodes (manage the cluster).
* **Pod** = a wrapper around one or more containers. This layer makes us not dependent on a specific container runtime like Docker — so that Kubernetes interacts with the pod, not the container runtime directly — this makes it CRI (Container Runtime Interface) agnostic.

while pods can have multiple containers, they are typically used for a single primary container with optional sidecars/helpers.

Each pod (not container) gets its own internal IP, meaning containers in the same pod can communicate via localhost. Pods can talk to each other within the cluster via their IPs.
Now, if a **pod dies**, Kubernetes automatically creates a **new pod**, but the IP will be **different**.

So imagine I have a Node.js app running on one pod and it talks to the database pod using that pod's internal IP. If that DB pod dies, a new one is created with a **new IP**, and my app loses the connection — now I have to **readjust the IP** manually every time. Not ideal.

That’s why we use another Kubernetes component:

#### **Service**

A **Service** gives a **static IP** (and DNS name) to a pod. So, When a pod dies, the Service automatically connects to the new pod without changing the IP or DNS. You don’t have to do anything — it just works.

So now my app doesn’t need to worry about the new pod IP — it just connects to the Service. 
---

Now I want my app to be accessible from a **browser**. For that, I need:

* **External Service** = opens communication **from outside the cluster** (like for frontend apps).
* **Internal Service** = the default type. Only accessible **inside the cluster** (used for DB pods etc.).

But here’s the problem — even with an external service, the URL looks like this:

```
http://128.89.101.2:8080  
(http + node IP + service port)
```

That’s not ideal. What I want is:

```
https://myapp.com  
(secure + domain name)
```

For that, Kubernetes has another component:

#### **Ingress**

Ingress acts as a **router or gateway**. So:

1. The request comes to **Ingress**.
2. Ingress forwards it to the correct **Service**, which then routes it to the pod.

---

Now, inside my app, let’s say I have a **MongoDB endpoint** — the app connects to it using the **service name** (like `mongodb-service`).

But here's the catch: this endpoint is often **hardcoded inside the app image**. So if the service name changes, I now need to:

* Manually update the app code with the new DB service name,
* Rebuild the image,
* Push it to the repo,
* Pull the new image into the pod,
* And restart the pod.

All this just for changing the DB endpoint.

To avoid that, Kubernetes has:

#### **ConfigMap**

A **ConfigMap** stores **external configuration** for your app — like URLs of DB or other services. You connect the ConfigMap to a pod, and now the pod can access those values directly.

If the DB service name changes, just update the ConfigMap — no need to rebuild the app image or restart the whole pod.
but note that changing a ConfigMap does not auto-update running pods—you must restart them or use tools like Reloader
---

Now comes another case — what if I want to store **secrets** like DB usernames and passwords?

We should not put them in ConfigMaps (which are plain text). For that, Kubernetes has:

#### **Secret**

A **Secret** is like a ConfigMap, but meant for **sensitive data**. It stores the data in **Base64 encoded** format. Still not super secure, but better than plain text. Actual encryption is handled using cloud provider tools or custom libraries.

Just like ConfigMap, you attach Secrets to your pod, and the app can use them directly.

---

Now another issue — what if **my DB pod restarts or dies?** That would also **wipe the database data**, which is terrible.

To fix this, Kubernetes has:

#### **Volumes**

A **Volume** is storage attached to a pod. This storage can be:

* **Local** (on the same node as the pod),
* Or **Remote** (cloud storage or external persistent storage).

So even if the pod restarts, the data stays — because it lives in the volume.

But one important point — **Kubernetes does not manage volume backups or replication.** We, as users, have to handle data safety, replication, and backups ourselves.

---

Now think about this — what if **my app pod dies or is restarted** (maybe due to a new image being deployed)? That causes **downtime**.

To avoid this, we replicate pods across **multiple servers**.

**Multiple servers mean multiple worker nodes?**
Yes — when we say "multiple servers", it usually means **multiple worker nodes** in Kubernetes.

So now my app pod runs on multiple worker nodes — and each is connected to the **Service**.

A Service not only gives a static IP but also acts as a:

#### **Load Balancer**

So when a user accesses `my-app.com`, the Service sees there are multiple pods behind it, so it distributes the request to whichever pod is least busy.

That way, Service also handles **traffic balancing**.

But manually creating identical pods is tedious. Instead, we use: 

#### Deployment
A Deployment is a template for managing pods. You tell it:

What container image to run.
How many replicas you want (e.g., replicas: 3).

Deployments handle:

✅ Scaling (up/down).
✅ Rolling updates (zero-downtime deployments).Rolling Updates = No downtime because Kubernetes swaps pods one by one, not all at once
✅ Rollback if something goes wrong.

##### Rolling updates
You update code → build new image (v2).
Kubernetes kills one old pod (v1), starts one new pod (v2), waits for it to be ready, only then repeats.
so, At least one pod is always running → zero downtime.

We don’t create pods directly. We create **Deployments**, 

* **Pod** = abstraction over container.
* **Deployment** = abstraction over pods, and the main way we manage them in real apps.

If one pod dies, the Service simply forwards to the next.

---
Now what about the DB pod?

If the **database pod dies**, our app still breaks.

So we want to replicate DB pods too. But unlike app pods, DB pods are **stateful** — they read/write data and need **consistency**.

You can’t use **Deployment** for that — because multiple database pods writing to the same data can corrupt it.

So for that, Kubernetes has:

#### **StatefulSet**

This is made for **stateful apps** like MySQL, MongoDB, Elasticsearch, etc.

StatefulSet:

* Replicates DB pods,
* Ensures they share storage properly,
* Manages which pod reads or writes to storage to maintain data consistency.

But using StatefulSets can be **tedious**, especially compared to Deployments.

That’s why many teams just **host databases outside Kubernetes**, and only run stateless apps (like frontend/backend services) inside the cluster. These stateless apps can scale easily with Deployments and talk to the external DB over a network.

---
Deployment: Manages stateless pod replicas
StatefulSet: For stateful apps (e.g., databases).


### imp concepts
API server is the only entry point to the Kubernetes cluster.
So we send configuration requests via Kubernetes dashboard (GUI), CLI, or API to the API server.
These requests are either in JSON format or `.yml` format.
Configuration requests in Kubernetes are **declarative** —
we declare what our **desired outcome** is to Kubernetes,
and Kubernetes tries to **fulfill those desires**.
Example — if desired state = 2 pods and 1 pod crashes, Kubernetes spins up **automatically** another pod to meet that desire.

---

### k8s configuration file

### 🔸 Q: Now How Do we create each Kubernetes component?

we create a **separate YAML file** (or sometimes combine them in one file) for **each component** we want to use in our cluster.
so Each component in Kubernetes = needs its own config (YAML file), where you define exactly how that component should work. 🙌
---

### 🔹 Example 1: You want to use **Ingress**

* You write an **Ingress YAML file**
* In it, you define:

  * the **DNS name** (like `myapp.com`)
  * and which **Service** it should forward traffic to


### 🔹 Example 2: You want to use **Volume / Persistent Storage**

* You create a **PersistentVolumeClaim (PVC)** YAML file
* And in your **Pod/Deployment YAML**, you reference that PVC
* You tell K8s:

  > “I want this pod’s data stored in this volume path.”

📄 So you need:

1. `pvc.yaml` → to request storage
2. `deployment.yaml` → to mount that volume into a pod

---
### example 3: you want to use **Deployment**

* In a **Deployment config file**, you define:

  * **How many pods** you want (replicas)
  * **Which container image** to use (like nginx, MongoDB, etc.)
  * What to do **if a pod crashes** (K8s auto-restarts it)
  * Basically — “I want these many pods of my app/db running all the time.”

📦 So with **Deployment config**, you're saying:

> “Run 3 pods of my app and keep them alive always.”

✅ So yes — it’s for **defining and managing pods** of your app, database, etc.

### example 4: you want to use **Service**
In this config file, you define:

  * What **pods** it should point to (using `selector`)
  * What **port** to expose
  * What type of service (ClusterIP, NodePort, LoadBalancer)

📡 So with **Service config**, you're saying:

> “Create a service that always knows where my pods are — give it a fixed DNS/IP — and forward traffic to those pods.”


### ✅ So yes — this is how you think in K8s:

* Need a **Deployment**? → `deployment.yaml`
* Need a **Service**? → `service.yaml`
* Need an **Ingress**? → `ingress.yaml`
* Need a **Volume**? → `pvc.yaml`
* Need config/secrets? → `configmap.yaml`, `secret.yaml`

---

### 3 parts of K8s configuration file:

Deployment config file & Service config file — all types of config files have 3 parts.
First two lines of the configuration file declare **what you want to create** — Deployment/Service,
and then the **API version** for different components:

```yaml
apiVersion: apps/v1 
kind: Deployment
```

```yaml
apiVersion: v1
kind: Service
```

---

**1. Metadata**
Metadata of that component that you're creating — like **name** of the component:

```yaml
metadata:  
  name: nginx-deployment 
  labels: -
```

```yaml
metadata:
  name: nginx-service
```

---

**2. Specification**
Each component's configuration file will have a **specification**,
where you basically put every kind of configuration that you want to apply for that component:

```yaml
spec: 
  replicas: 2  
  selector: - 
  template: ---
```

```yaml
spec:
  selector: -
  ports: -
```

Inside the `spec`, the attributes depend on the kind of component we’re creating — whether **Deployment** or **Service**.

---

**3. Status**
Automatically generated by Kubernetes —
adds the **status of your Deployment** and updates continuously.
Kubernetes gets this status data from the **etcd** DB.
**etcd DB holds the current status** of any K8s component.

---

We store Kubernetes configuration files:
👉 Inside the project repo in a folder like this-- `/k8s/configs/deployments`
So it's version-controlled with your code- best practice ✅

 