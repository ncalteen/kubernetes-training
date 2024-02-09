# Kubernetes Architecture

## Architecture

- Container orchestrator
  - Start and stop containers based on requirements
- Workload placement
  - Do containers need to reside alongside other containers
  - Are there networking/compute requirements
- Infrastructure abstraction
- Desired state
  - Define the state as code and let Kubernetes handle enforcement
- Benefits
  - Speed of deployment
  - Absorb and implement changes quickly
  - Recover quickly from failures
  - Hide complexity in the cluster (storage, networking, placement, etc.)
- Principles
  - Desired state/declarative configuration
  - Controllers/control loops
    - Monitor running state of the system to ensure it is in desired state
    - If not, controller handles enforcement
  - Kubernetes API/API server
    - Objects that can be used to build systems to deploy
    - Defines the desired state of the system
    - Implemented and available via the API server
    - Central information hub for a cluster
- Kubernetes API
  - API objects are primitives that represent the state of a system (pods,
    nodes, etc.)
  - Enables configuration of state (declarative or imperative)
- Kubernetes API server
  - RESTful API over HTTP using JSON
  - **Sole way** to interact with the cluster (both for you and Kubernetes
    itself)
  - Changes are serialized and persisted in the cluster datastore
- Kubernetes API Objects
  - Pods: Collection of containers that are deployed as a single unit
  - Controllers: Systems to keep the cluster in the desired state
  - Services: Persistent access point to applications that are deployed as pods
    - Pods change, services don't
  - Storage: Persistent storage for applications
- Pods
  - One or more containers
  - Your application or service
  - The most basic unit of work
  - The main unit for scheduling
  - Ephemeral (no pod is ever redeployed, just replaced)
  - Atomicity (they either exist or don't)
    - In a multi-container pod, if one container dies, the entire pod is
      considered failed
  - Kubernetes' job is to keep pods running and in the desired state
  - Pod State: Is the pod up and running?
  - Pod Health: Is the application in the pod up and running?
  - Probes: Check the health of the application (e.g. send a request to a
    `/health-check` endpoint)

How does Kubernetes manage pod state?

- Controllers
  - Defines desired state
  - Exposed as a workload resource API object
  - Create and manage pods for you
  - Monitor and respond to pod state and health
- `ReplicaSet` controller
  - Define a number of replicas for a particular pod to keep running
  - If a pod becomes unavailable, the controller replaces the failed pod
  - Generally you won't create `ReplicaSets` directly
- `Deployment` controller
  - Manages rollout of `ReplicaSets`
  - Manages transition between `ReplicaSets` (e.g. transition between two
    versions)
- Many more controllers available, not just ones for pods

How does Kubernetes add persistency?

- Services
- Add persistency to ephemeral pods
- Networking abstraction for pod access
- Allocates IP and DNS name for the collection of pods that back the service
- Dynamically updated based on pod lifecycle
- Users/systems only access via the IP and DNS of the service
- Kubernetes handles routing from the service to the health pod(s)
- Scaling can be done by adding and removing pods
- Distributes load across pod(s)

Where is data stored?

- Volumes
  - Storage backed by physical media
  - Directly accessible to pods
  - Tightly coupled, lacks flexibility
- Persisten Volume
  - Pod-independent storage
  - Defined by the administrator at the cluster level
- Persistent Volume Claim
  - Pod definition "claims" storage from the persistent volume
  - Decouples pod from storage

Cluster Architecture

- Cluster components
  - Control plane node
    - Major control functions of a cluster
    - Operations, monitoring, scheduling
  - Node / Worker node
    - Start pods and underlying containers
    - Implement networking for reachability
    - Contributes to compute capacity of the cluster
    - Generally 2+ nodes per cluster
    - Virtual or physical machines
- Control Plan Node / Master Node
  - API server
    - Stateless
  - `etcd`
    - Cluster state storage and persistence
  - Scheduler
    - Tells Kubernetes which nodes to start pods on
  - Controller Manager
    - Implementing lifecycle functions of controllers
- `kubectl`
  - Interaction with the API server
  - Retrieve information, make changes, etc.
- API server
  - Central to the control of the cluster
  - Responsible for validating operations and persisting data to `etcd`
- `etcd`
  - Persists state as key/value pairs
- Scheduler
  - Watch API server for unscheduled pods and schedule them
  - Evaluates resource needs for pods
  - Must respect administrative constraints (e.g. affinity/anti-affinity)
- Controller Manager
  - Run controller loops
  - Implement lifecycle functions of pods, enforce desired state
  - Watch current state and update API server as needed
  - Ex: `ReplicaSet` controller
- Nodes
  - Runs pods
  - Implements networking
  - Kubelet
    - Starts pods on the node
  - Kube-proxy
    - Networking and services abstraction
  - Container Runtime
    - Pulls container images, provides execution environment and pod abstraction
  - Kubelet and kube-proxy interact with API Server for changes to implement
- Kubelet, kube-proxy, and container runtime also run on control plane nodes
- Kubelet
  - Monitors API server for changes
  - As pods are scheduled, Kubelet handles pod lifecycle changes
  - Reports node and pod state to API Server
  - Executes pod probes
- Kube-proxy
  - All network components
  - Commonly uses `iptables`
  - Implements services abstraction
  - Routes traffic to pods
  - Load balances traffic to pods
- Container Runtime
  - Download container images
  - Start and run containers
  - Container runtime interface (CRI)
    - Swap container runtimes dynamically
    - Default is `containerd`, many others available...
    - Docker was deprecated in v1.20 in favor of `containerd`
    - Docker containers are supported as long as they are CRI compliant
- Cluster Add-On Pods
  - Provides additional services to clusters
  - Examples
    - Primary example is DNS
    - Ingress controllers (layer 7 load balancers)
    - Dashboard (visual management of a cluster)
- Pod Operations
  - Use `kubectl` to create a Deployment
  - Request is submitted to API server
  - API Server stores information in `etcd`
  - Controller Manager creates the replicas in the `ReplicaSet`
  - Request is sent to Scheduler
  - Scheduler tells API server which nodes to schedule on
  - API server stores scheduler information in `etcd`
  - Kubelets on nodes ask API server for workloads
  - Kubelets create pods
  - _Node goes down_
  - Controller Manager notices we are outside of desired state
  - Controller Manager requests a new pod to be scheduled
  - Scheduler finds new node to schedule the pod on
  - Kubelet on the node creates the pod
- In the default configuration, the Scheduler node "taints" the control plane
  node so it will **only** run _system_ nodes (non-user workloads)

### Networking Basics

- Every pod gets its own unique IP address
- Pods on a node can communicate with **all pods on all nodes** without NAT
- Agents on a node can communicate with **all pods on that node**
- Containers in the same pod communicate over `localhost`
- Containers in different pods communicate over the pod IP adresses (layer 2)
- Containers in a different node communicate over the node IP addresses (layer 2
  or 3)
  - If you don't control the networking infrastructure, you can use an **overlay
    network** to simulate this
- Kube-proxy implements network access for external users

## Install and Configure

### Installation Considerations

Where to install?

- Cloud
  - IaaS
    - Create VMs
    - Install Kubernetes on the VMs
    - Manage patching/updates/etc.
    - Full control over everything
  - PaaS
    - Kubernetes as a managed services
    - Don't need to manage underlying infrastructure
    - Lose flexibility in versioning and features
- On-Premises
  - Bare Metal
  - Virtual Machines

Which to choose?

- Skill set
- Strategy of the organization

Cluster Networking

- Consider how pods will communicate
- Overlay network vs. layer 2/3 configuration
- Network IP range overlaps

Scalability

- Are there enough nodes and enough resources on each node
- Can we support node failure

High Availability

- Should there be multiple control plan node(s)

Disaster Recovery

- Backup and restore process (especially `etcd`)

### Installation Methods

Desktop

- Good for development
- Comes with Docker Desktop

`kubeadm`

- Package to bootstrap and manage clusters

Cloud

- Deploy as IaaS or PaaS

### Installation Requirements

System

- Linux (Ubuntu/RHEL/etc)
- 2+ CPU
- 2 GB RAM
- Swap must be disabled

Container Runtime

- Container Runtime Interface (CRI) compatible
- `containerd` (preferred), Docker (deprecated), CRI-O

Networking

- Connectivity between nodes
- Unique hostname/MAC for each node

Ports

| Component          | Ports (TCP)    | Used By       |
| ------------------ | -------------- | ------------- |
| API Server         | 6443           | All           |
| `etcd`             | 2379, 2380     | API/`etcd`    |
| Scheduler          | 10251          | Self          |
| Controller Manager | 10252          | Self          |
| Kubelet            | 10250          | Control Plane |
| NodePort           | 30000 to 32767 | All           |

- NodePort exposes services and ports on each node in the cluster.

### Getting Kubernetes

- [`kubernetes/kubernetes`](https://github.com/kubernetes/kubernetes)
- Linux distribution repositories (`yum` and `apt`)

Building a cluster

- Install and configure packages
- Create the cluster using `kubeadm`
- Configure pod networking
- Join nodes to the cluster

Required packages

- `containerd`
- `kubelet`
- `kubeadm`
- `kubectl`
- These should be installed on all nodes in the cluster, not just control plane
  nodes

Getting and installing Kubernetes on Ubuntu VMs (all nodes)

```bash
# Install containerd
sudo apt-get install -y containerd

# Add Kubernetes repos
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
cat <<EOF >/etc/apt/sources.list.d/kubernetes.list
deb https://apt.kubernetes.io/ kubernetes-xenial main
EOF
apt-get update

# Install Kubernetes packages
apt-get install -y kubelet kubeadm kubectl

# Control when we move between versions
apt-mark hold kubelet kubeadm kubectl containerd
```

### Install and Configure Packages

These need to be run on each node and control plane node. Check Kubernetes docs
for most updated steps.

- Disable swap
- Install `containerd` modules
- Set required `sysctl` params
- Apply `sysctl` params
- Install `containerd`
- Create a `containerd` configuration file
- Set the `cgroup` driver from `containerd` to `systemd` (required for
  `kubelet`)
- Restart `containerd`
- Install `kubeadm`, `kubelet`, and `kubectl`

### Install a Cluster with `kubeadm`

### Create a Cluster in the Cloud

## Basic Operation
