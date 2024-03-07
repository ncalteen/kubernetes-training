# Exploring the Kubernetes Architecture

## Introduction, Course, and Module Overview

## What is Kubernetes? Kubernetes Benefits and Operating Principles

- Container Orchestrator
  - Start/stop containers based on system requirements
- Workload Placement
  - Which servers should a container live on? Does a container need to reside
    next to other containers?
- Infrastructure Abstraction
- Desired State
  - Describe how the system should look

Key benefits:

- Speed of deployment
- Ability to absorb change quickly
- Ability to recover quickly
- Hide infrastructure complexity
  - Storage, networking, etc.

Kubernetes principles:

- Declarative configuration
  - Define deployment state in code. Kubernetes does the work to bring it
    online.
- Controllers/Control Loops
  - Constantly monitor the running state of the system. If it is not in the
    desired state, controllers will try to enforce it.
- Kubernetes API/API Server
  - Collection of objects we can use to build systems. Available via the API
    server.

## Introducing the Kubernetes API - Objects and API Server

- API Objects
  - Primitives that represent the state of the system. Pods, nodes, etc. Enables
    both declarative and imperative state configuration.
- RESTful API over HTTP using JSON
- The **only** way to interact with the cluster, both for you and Kubernetes
  itself.
- Cluster configuration is serialized and persisted into the cluster data store.

Kubernetes API objects:

- Pods
- Controllers
- Services
  - Persistent access point for applications running in pods
- Storage

## Understanding API Objects - Pods

- Represents one or more containers
- Most basic unit of work
- A unit of scheduling (Kubernetes schedules pods, not containers)
- Ephemeral
  - No pod is ever redeployed
- Atomic
  - They are there or they are not
  - Includes **all** containers in the same pod
- Pod state is tracked (whether it is up and running)
- Pod health tracks the health of the application running on a pod using
  **probes**
  - Probes can be configured for your application (e.g. an endpoint to query)

## Understanding API Objects - Controllers

- Defines the desired state for a cluster and applications
- Exposed as workload resource objects
- Creates and manages pods for you
- Monitor and respond to pod state/health

### `ReplicaSet` Controller

- Define a number of replicas for a pod that we want to have up and running
- E.g. have 3 pods for a particular web app running at all times
  - If a pod becomes unavailable/unhealthy, the controller will redeploy it
- Generally you won't create `ReplicaSet` objects directly

### `Deployment` Controller

- Manages the rollout of `ReplicaSet` objects

## Understanding API Objects - Services

- Add persistency to ephemeral pods
- Networking abstraction for access to pods
- Kuberntes allocates an IP and DNS name for the service
  - Dynamically updated as pods are replaced/launched
- Scale by adding and removing pods
- Provides load balancing across pods

## Understanding API Objects - Storage

- Volumes
  - Storage backed by physical media directly accessible to the pods
  - Tightly coupled to a Deployment
- Persistent Volume
  - Pod-independent storage
  - Defined at the cluster level
  - Pods create Persistent Volume Claims to get portions of the Persistent
    Volume storage

## Kubernetes Cluster Components Overview and Control Plane

Cluster components:

- Control Plane Node
  - Implements major control functions of a cluster
  - Monitoring, scheduling, orchestration
- Node
  - Where pods actually run
  - Start pods and underlying containers
  - Implement networking for reachability to pods
  - Can be virtual or physical

Control Plane Node components:

- API Server
  - Communication hub of the cluster
  - Stateless
- Cluster Store (`etcd`)
  - Persisting state
- Scheduler
  - Controls when and where pods are launched
- Controller Manager
  - Lifecycle functions of controllers
- `kubectl`
  - How we interact with the API server for administrative functions

API Server:

- Central to the cluster
- RESTful API
- Validate operations and persist state to `etcd`

`etcd`:

- Cluster data store
- Key-value pairs

Scheduler:

- Watch API server
- Schedules pods
- Determines where to schedule based on pod needs and admin constraints
  - E.g. pod affinity

Controller Manager:

- Execute controller loops
  - Lifecycle functions and desired state of pods
- Watches the current state and updates API server

## Nodes

- Where application pods run
- Starts pods and ensures containers are running
- Implement networking to provide reachability to pods
- One node can have many running pods
- Can be virtual or physical
- Kubelet, Kube-proxy, and Container Runtime run on all nodes, including the
  Control Plane Node

Kubelet

- Monitors the API Server for changes
- Manages pod lifecycle (start/stop pods and containers)
- Reports node and pod state to the API Server
- Executes pod probs

Kube-proxy

- Networking components for nodes
  - Commonly implemented with `iptables`
- Implements services
- Routes traffic to pods
- Load balances traffic to multiple pods

Container Runtime

- Pulls container images
- Starts and runs containers
- Uses a **Container Runtime Interface (CRI)** to abstract the container runtime
- Commonly implements `containerd` as the container runtime
  - Previously `docker`, but replaced after v1.20
- Containers must be CRI compliant

## Cluster Add-On Pods

- Pods that provide special services to a cluster
  - E.g. DNS via core-dns
    - Pods register with the DNS server
    - Commonly used for service discovery
  - E.g. Ingress controllers
  - E.g. Dashboard for web-based administration
  - E.g. Network overlays

## Pod Operations

1. Use `kubectl` to instruct the cluster to create a deployment with 3 replicas
   of a pod
1. Request is submitted to API Server
1. API Server stores information in `etcd`
1. Controller Manager creates 3 replicas in a `ReplicaSet`
1. Request is submitted to Scheduler
1. Scheduler tells API Server
1. Kubelet on the nodes will ask API Server for work
1. Kubelet will launch pods on the nodes

What happens if a node goes down?

1. Controller Manager sees we aren't in desired state
1. Controller Manager tells Scheduler to bring up more nodes
1. Kubelet on the nodes will ask API Server for work
1. Kubelet will launch pods on the nodes

- Control Plane Node is "tainted" so that workloads other than administration
  cannot be run on it

## Service Operations

- Access to pods is exposed via a service endpoint (e.g. HTTP over port 80)
- Requests are load balanced to underlying pods
- If a pod goes down, the `ReplicaSet` controller will remove that pod and
  create a new one
  - The Service will not attempt to route requests to the unhealthy pod

## Kubernetes Networking Fundamentals

- Every pod will be assigned its own IP address

Networking requirements:

- Pods on a node can communicate with all pods on all nodes without NAT
  - They can use their real IPs
- Agents on a node can communicate with all pods on that node

Fundamentals:

- Containers in the same pod can communicate with one another via `localhost`
- Containers in different pods on the same node can communicate with one another
  via layer 2 bridge using the pod IP addresses
- Containers in different pods on different nodes can communicate with one
  another via layer 2 or 3 bridge using the pod IP addresses
  - May require networking assistance to ensure layer 2/3 connectivity
  - Can also be done using an overlay network
- External services can communicate with pods via Kube-proxy
