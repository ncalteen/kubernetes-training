# Working with your Kubernetes Cluster

## Introducting and Using `kubectl`

- Primary CLI tool to interact with the API Server.
- Perform **operations** on **resources** in the cluster.

Core operations:

- `apply`/`create` - Create resource(s)
- `run` - Start a pod from an image
- `explain` - Documentation of resources and fields needed to construct an API
  object
- `delete` - Delete resource(s)
- `get` - Display information about multiple resource(s)
- `describe` - Display detailed information about a resource
- `exec` - Execute a command on a container
- `logs` - View stdout logs from a running container
- There are many more...

Core resources to perform operations on:

- `nodes` (`no`)
- `pods` (`po`)
- `services` (`svc`)
- There are many more...

You can specify the output format:

- `wide` - Output additional information
- `yaml` / `json` - Formatted API objects
- `dry-run` - Outputs an object without sending it to the API Server
  - Useful to generate YAML for deployments, services, etc. before creating them

## A Closer Look at `kubectl`

Command format:

```bash
kubectl [command] [type] [name] [flags]

# Get `my-pod` and output as YAML
kubectl get pods my-pod --output=yaml

# Create a deployment named nginx using the nginx image
kubectl create deployment nginx --image=nginx
```

## Demo: Using `kubectl`: Nodes, Pods, API Resources and bash Auto-Completion

```bash
# List and inspect your cluster
kubectl cluster-info

# Get current nodes
kubectl get nodes

# Output additional information
kubectl get nodes --output wide

# Get Pods in current namespace
kubectl get pods

# Get Pods in another namespace
kubectl get pods --namespace kube-system

# Output additional information
kubectl get pods --namespace kube-system --output wide

# List everything in all namespaces
kubectl get all --all-namespaces | more

# List API resources that the cluster is aware of (this can vary depending on
# what is installed on the cluster). Provides information such as the name,
# shortname, kind, and whether or not the resource is namespace-bound.
kubectl api-resources | more
kubectl api-resources | grep pod

# Explain a resource in detail
kubectl explain pod | more

# Explain a resource property in detail
kubectl explain pod.spec | more

# Output all the fields of an API resource
kubectl explain pod --recursive | more

# Describe a resource in the cluster
kubectl describe nodes my-node | more

# Get help information, along with examples of commands
kubectl -h | more
kubectl get -h | more
kubectl create -h | more

# Enable autocompletion
brew install bash-completion
# Add the following to your ~/.zprofile
# source <(kubectl completion zsh)
```

## Application and Pod Deployment in Kubernetes and Working with YAML Manifests

- Using `kubectl` is **imperative.** You can only perform one action at a time
  on one object at a time.
- Manifests can be used to **declaratively** configure Kubernetes.

Basic Deployment manifest:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: nginx
  name: nginx
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx
  strategy: {}
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
        - image: us-docker.pkg.dev/google-samples/containers/gke/hello-app:1.0
          name: hello-app
          resources: {}
status: {}
```

You can also generate the manifest using `dry-run`:

```bash
kubectl create deployment nginx \
  --image us-docker.pkg.dev/google-samples/containers/gke/hello-app:1.0 \
  --dry-run=client -o yaml > nginx-deployment.yaml
```

If you want to deploy this manifest, you need to ensure there are node(s) that
are labeled using the selector defined above.

```bash
# Label a node
kubectl label nodes <node-name> app=nginx

# List labels on nodes
kubectl get nodes --show-labels
```

## Demo: Imperative Deployments and Working with Resources in your Cluster

```bash
# Create a basic deployment (this will launch a pod)
kubectl create deployment nginx --image=nginx:latest

# Deploy a single pod that is not managed by a deployment
kubectl run nginx-pod --image=nginx:latest
```

Pods associated with deployments will have unique names.

- E.g. `<deployment>-<pod-template-hash>-<unique-id>`
- Pod template hash is unique amongst ReplicaSets in a Deployment
- The ID is unique amon pods in the same ReplicaSet

```bash
# Open an ssh connection into a node (in this case, using minikube)
minikube ssh --node minikube-m02

# (containerd) Use crictl to list running containers
sudo crictl --runtime-endpoint unix:///run/containerd/containerd.sock ps

# (Docker) List running containers
sudo docker ps
```

Useful troubleshooting:

```bash
# Get logs
kubectl logs <pod-name>

# Attach a shell to a running pod
kubectl exec -it <pod-name> -- /bin/sh
```

Useful deployment-related commands:

```bash
# Get ReplicaSets
kubectl get replicaset

# Get pods
kubectl get pods

# Get more information about a deployment or replicaset
kubectl describe deployment nginx | more
kubectl describe replicaset <replicaset> | more
```

## Demo: Exposing and Accessing Services in Your Cluster

```bash
# Expose a Deployment as a Service. The port parameter is the port the Service
# listens on. The target-port is the port the Pod is listening on.
kubectl expose deployment nginx \
  --port=80 \
  --target-port=80

# This will display the IP and port the pod is listening on.
kubectl get service nginx

# This will describe the IP and port pairs (Endpoints) for the service
kubectl describe service nginx
kubectl get endpoints nginx

# SSH into a node and curl the exposed endpoint
minikube ssh --node <node-name>
curl http://<endpoint>:<target-port>
```

## Demo: Declarative Deployments and Accessing and Modifying Existing Resources

```bash
# Create a deployment manifest file
kubectl create deployment nginx \
  --image=nginx:latest \
  --dry-run=client -o yaml > deployment.yaml

# Create the deployment described in the file
kubectl apply -f deployment.yaml

# Create a service manifest file
kubectl expose deployment nginx \
  --port=80 --target-port=80 \
  --dry-run=client -o yaml > service.yaml

# Create the service
kubectl apply -f service.yaml

# Get the running resources
kubectl get all

# Scale up the deployment
# Update deployment.yaml to set replicas to 10
# Deploy the change
kubectl apply -f deployment.yaml

# Now there should be 10 pods creating/running
kubectl get all

# SSH into a node and curl the exposed endpoint. This should be load
# balanced across all 10 pods.
minikube ssh --node <node-name>
curl http://<endpoint>:<target-port>
```

If you don't have the deployment manifest, you can also edit existing resources
on the fly.

```bash
# Open the existing API resource in an editor (vim). Saving this will apply
# the changes immediately.
kubectl edit deployment nginx

# There should be 5 pods creating/running
kubectl get all

# You can also scale using `kubectl scale`
kubectl scale deployment nginx --replicas=10

# There should be 10 pods creating/running
kubectl get all

# Go back to 1 replica
kubectl scale deployment nginx --replicas=1
```
