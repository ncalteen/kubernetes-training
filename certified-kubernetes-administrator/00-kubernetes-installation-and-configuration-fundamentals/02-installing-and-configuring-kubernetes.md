# Installing and Configuring Kubernetes

## Installation Considerations

Cloud

- **IaaS:** Deploy VMs and install Kubernetes yourself
- **PaaS:** Consume Kubernetes as a service

On-Premises

- Bare metal
- Virtual machines

The choice is heavily dependent on:

- Skill-set of the organization
- Current infrastructure
- Budget
- ...and more!

Cluster Networking

- Overlay
- Create layer 2 and 3 connectivity
- Network IP range overlaps

Scalability

- Sufficient nodes to handle peak workloads
- Sufficient nodes to handle failure
- High availability (e.g. multiple Control Plane Nodes)
- Disaster recovery

## Installation Methods

- Desktop installation for dev/test/learning
- `kubeadm` for installing and configuring a cluster for production clusters
- Cloud scenarios

## Installation Requirements

System requirements (minimum):

- Linux (Ubuntu/RHEL)
- 2 CPUs
- 2 GB RAM
- Swap disabled

Container runtime:

- Must be Container Runtime Interface (CRI) compliant
- `containerd` is preferred
- `docker` is deprecated

Networking:

- Connectivity between all nodes
- Each system has a unique hostname and MAC

## Understanding Cluster Networking Ports

Control Plane Node

| Component          | Ports      | Used By                          |
| ------------------ | ---------- | -------------------------------- |
| API Server         | 6443       | All                              |
| `etcd`             | 2379, 2380 | API Server / `etcd` Replicas     |
| Scheduler          | 10251      | Scheduler / `localhost`          |
| Controller Manager | 10252      | Controller Manager / `localhost` |
| Kubelet            | 10250      | Control Plane                    |

Node

| Component | Ports       | Used By       |
| --------- | ----------- | ------------- |
| Kubelet   | 10250       | Control Plane |
| NodePort  | 30000-32767 | All           |

> NodePort exposes services provided by pods.

## Getting Kubernetes

[`kubernetes/kubernetes`](https://github.com/kubernetes/kubernetes)

- `yum` or `apt`

## Building Your Own Cluster

1. Install and configure Container Runtime and Kubernetes packages
1. Bootstrap a cluster with `kubeadm`
1. Configure pod networking
1. Join nodes to the cluster

Required packages (install on **all** nodes in the cluster):

- `containerd`
- `kubelet`
- `kubeadm`
- `kubectl`

## Installing Kubernetes on Ubuntu VMs

This should be done on **all** nodes

1. Install the Container Runtime

   ```bash
   sudo apt-get install -y containerd
   ```

1. Add the Kubernetes repository to the apt repository list

   ```bash
   curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -

   cat <<EOF >/etc/apt/sources.list.d/kubernetes.list
   deb https://apt.kubernetes.io/ kubernetes-xenial main
   EOF
   ```

1. Update the package list

   ```bash
   apt-get update
   ```

1. Install the remaining packages

   ```bash
   apt-get install -y kubelet kubeadm kubectl
   ```

1. Hold the package versions (so you can control when they are updated)

   ```bash
   apt-mark hold kubelet kubeadm kubectl containerd
   ```

> Kubernetes has a well-defined upgrade path.

## Lab Environment Overview

- One Control Plane Node
- Three Worker Nodes
- Ubuntu 22.04
- Kubernetes 1.26
- Hostnames are manually set in the `/etc/hosts` on each node

## Demo: Installing and Configuring `containerd`

```bash
# Disable swap
swapoff -a
vi /etc/fstab # Comment out or remove the swap line

# Install containerd prerequisites
sudo modprobe overlay
sudo modprobe br_netfilter

# Load modules on startup
cat <<EOF | sudo tee /etc/modules-load.d/containerd.conf
overlay
br_netfilter
EOF

# Configure sysctl params
cat <<EOF | sudo tee /etc/sysctl.d/99-kubernetes-cri.conf
net.bridge.bridge-nf-call-iptables  = 1
net.ipv4.ip_forward                 = 1
net.bridge.bridge-fn-call-ip6tables = 1
EOF

# Apply sysctl params (without reboot)
sudo sysctl --system

# Install containerd
sudo apt-get update
sudo apt-get install -y containerd

# Create a configuration file and generate a default configuration file
sudo mkdir -p /etc/containerd
sudo containerd config default | sudo tee /etc/containerd/config.toml

# Set the cgroup driver for containerd to systemd (required for kubelet)
sudo vi /etc/containerd/config.toml
# [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc]
#   [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
#     SystemdCgroup = true

# Restart containerd
sudo systemctl restart containerd
```

## Demo: Installing and Configuring Kubernetes Packages

```bash
# Add the Kubernetes apt repository
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
cat <<EOF >/etc/apt/sources.list.d/kubernetes.list
deb https://apt.kubernetes.io/ kubernetes-xenial main
EOF

# Update the package list
sudo apt-get update

# Check what versions are available
apt-cache policy kubelet | head -n 20

# Pin to a specific version and install
VERSION=1.20.1-00
sudo apt-get install -y kubelet=$VERSION kubeadm=$VERSION kubectl=$VERSION

# Hold the packages from being upgraded automatically
sudo apt-mark hold kubelet kubeadm kubectl containerd
```

Initially, `kubelet` will enter a crashloop until a cluster is created or the
node is joined to a cluster.

```bash
sudo systemctl status kubelet.service
sudo systemctl status containerd.service
```

Set the services to start automatically.

```bash
sudo systemctl enable kubelet.service
sudo systemctl enable containerd.service
```

## Boostrapping a Cluster with `kubeadm`

```bash
kubeadm init
```

- Pre-flight checks
  - Various checks to ensure the node will create successfully
  - Memory, connectivity, container runtime, etc.
- Creates a Certificate Authority
  - Authentication and encryption
- Generates `kubeconfig` files
- Generates static pod manifests
  - Kubelet monitors that location for pods to start
- Waits for the control plan pod(s) to start
- Taints the Control Plane Node(s) to prevent user pods from being scheduled
- Generates a bootstrap token
- Starts add-on components (e.g. DNS and kube-proxy)

Bootstrapping is highly customizable!

## Understanding the Certificate Authority's Role in Your Cluster

- `kubeadm` will create a self-signed CA by default
- Can be configured to use an external PKI
- Used to secure cluster communications
- Used to authenticate users and cluster components
- CA is distributed to each node in `/etc/kubernetes/pki`

## `kubeadm` Created `kubeconfig` Files and Static Pod Manifests

- `kubeconfig` is a configuration file used to connect to a cluster
  - Client certificates
  - Cluster API Server network location
- `kubeadm` creates several `kubeconfig` files in `/etc/kubernetes`
  - `admin.conf` - An administrator user
  - `kubelet.conf`
  - `controller-manager.conf`
  - `scheduler.conf`

Static Pod Manifests

- Describes configuration for the the cluster control plane components (`etcd`,
  API Server, Controller Manager, Scheduler)
- `/etc/kubernetes/manifests`
- `kubelet` watches this directory for changes and starts pods as needed
- Also used during startup
- Enables startup of the cluster components without the cluster actually being
  up and running

## Pod Networking Fundamentals

- Single, un-NATed IP address per pod
- Direct routing
  - Infrastructure must support IP reachability between pods and nodes
- Overlay networking
  - Tunneling and encapsulation of the packets
  - **Flannel** - Layer 3 virtual network
  - **Calico** - L3 and policy-based traffic management
  - **Weave Net** - Multi-host network

## Creating a Cluster Control Plane Node and Adding a Node

- Download a YAML manifest with the cloud network to deploy

  ```bash
  # Calico
  wget https://docs.projectcalico.org/manifests/calico.yaml
  ```

- Create a cluster configuration file

  ```bash
  # Configuration defaults for a cluster
  kubeadm config print init-defaults | tee ClusterConfiguration.yaml
  ```

- Initialize the cluster init process

  ```bash
  sudo kubeadm init \
    --config=ClusterConfiguration.yaml \
    --cri-socket /run/containerd/containerd.sock
  ```

  This will print out instructions for other commands to run such as joining new
  nodes.

- Create a directory for `kubeconfig` and copy the admin `kubeconfig`

  ```bash
  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config
  ```

- Apply the cluster networking configuration

  ```bash
  kubectl apply -f calico.yaml
  ```

At this point, you should have a fully-functioning Control Plane Node.

- Install packages
- Join the node to the cluster

  ```bash
  kubeadm join <api-server-ip>:6443 \
    --token <bootstrap-token> \
    --discovery-token-ca-cert-hash <cert-hash> \
  ```

  The node will download cluster information and submit a CSR to the API Server.
  This is used to join the node to the cluster. The CA will sign the CSR
  automatically and the signed certificate will be saved on the new node. A
  `kubeconfig` will be generated for the node.

## Demo: Creating a Cluster Control Plane Node

```bash
# Get the networking deployment manifest
wget https://docs.projectcalico.org/manifests/calico.yaml

# Update the CALICO_IPV4POOL_CIDR with the IPv4 IP range you want to use. This
# shouldn't overlap with other networks in the infrastructure.
vi calico.yaml

# Create a kubeadm configuration file.
kubeadm config print init-defaults | tee ClusterConfiguration.yaml

# Change the following in the configuration file:
# - IP endpoint for API server `localAPIEndpoint.advertiseAddress` to the
#   Control Plan Node IP
# - `nodeRegistration.criSocket` to `/run/containerd/containerd.sock`
# - Set cgroup driver for kubelet to systemd
# - Set the `kubernetesVersion` to the one you want to use
vi ClusterConfiguration.yaml

# Bootstrap the cluster!
sudo kubeadm init \
  --config=ClusterConfiguration.yaml \
  --cri-socket /run/containerd/containerd.sock
```

When the bootstrap process completes, it will provide information on how to use
`admin.conf` and join nodes to the cluster. Follow the steps there to configure
your account for Control Plane Node access using `admin.conf`.

```bash
mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config
```

Deploy the pod network.

```bash
kubectl apply -f calico.yaml
```

Verify pods and resources have been created to set up the pod network.

```bash
kubectl get pods --all-namespaces
kubectl get pods --all-namespaces --watch
kubectl get nodes
```

Initially, when the kubelet starts it will be stuck in a crashloop because there
are no static pod manifests to start the cluster with. Now it should be OK.

```bash
sudo systemctl status kubelet.service

# List the manifests
ls /etc/kubernetes/manifests
sudo more /etc/kubernetes/manifests/etcd.yaml
sudo more /etc/kubernetes/manifests/kube-apiserver.yaml
```

The kubeconfig files for the control plane pods are in `/etc/kubernetes`.

```bash
ls /etc/kubernetes
```

## Demo: Adding a Node to Your Cluster

Very similar to setting up a Control Node, except using `kubeadm join`. Do the
following on the node that will be joined.

```bash
# Disable swap
swapoff -a

# Install containerd prerequisites
sudo modprobe overlay
sudo modprobe br_netfilter

# Load modules on startup
cat <<EOF | sudo tee /etc/modules-load.d/containerd.conf
overlay
br_netfilter
EOF

# Configure sysctl params
cat <<EOF | sudo tee /etc/sysctl.d/99-kubernetes-cri.conf
net.bridge.bridge-nf-call-iptables  = 1
net.ipv4.ip_forward                 = 1
net.bridge.bridge-fn-call-ip6tables = 1
EOF

# Apply sysctl params (without reboot)
sudo sysctl --system

# Install containerd
sudo apt-get update
sudo apt-get install -y containerd

# Create a configuration file and generate a default configuration file
sudo mkdir -p /etc/containerd
sudo containerd config default | sudo tee /etc/containerd/config.toml

# Set the cgroup driver for containerd to systemd (required for kubelet)
sudo vi /etc/containerd/config.toml
# [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc]
#   [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
#     SystemdCgroup = true

# Restart containerd
sudo systemctl restart containerd

# Add the Kubernetes apt repository
curl -s https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo apt-key add -
cat <<EOF >/etc/apt/sources.list.d/kubernetes.list
deb https://apt.kubernetes.io/ kubernetes-xenial main
EOF

# Update the package list
sudo apt-get update

# Check what versions are available
apt-cache policy kubelet | head -n 20

# Pin to a specific version and install
VERSION=1.20.1-00
sudo apt-get install -y kubelet=$VERSION kubeadm=$VERSION kubectl=$VERSION

# Hold the packages from being upgraded automatically
sudo apt-mark hold kubelet kubeadm kubectl containerd

# Check the status of kubelet and containerd
sudo systemctl status kubelet.service
sudo systemctl status containerd.service

# Enable them on startup
sudo systemctl enable kubelet.service
sudo systemctl enable containerd.service
```

Next, do the following on the Control Plane node.

```bash
# Get the bootstrap token
kubeadm token list

# If the token is missing or expiring, generate one
kubeadm token create

# Get the CA cert hash
openssl x509 -pubkey -in /etc/kubernetes/pki/ca.crt | openssl rsa -pubin -outform der 2>/dev/null | openssl gfst -sha256 -hex | sed 's/^.* //'

# Generate the join command (this includes the token and cert automatically)
kubeadm token create --print-join-command
```

On the node, run the join command.

```bash
sudo kubeadm join <controller-ip>:6443 \
  --token <token> \
  --discovery-token-ca-cert-hash <cert-hash>
```

On the control plane node, you can follow the status of the node.

```bash
# Get the node status
kubectl get nodes

# Get the pod status
kubectl get pods --all-namespaces --watch
```

## Managed Cloud Deployment Scenarios: AKS, EKS, and GKE

## Demo: Creating a Cluster in the Cloud with AKS
