# Set Up ARC on Azure AKS

This document outlines how to configure ARC to run on AKS.

## Prerequisites

1. Install the
   [Azure CLI](https://learn.microsoft.com/en-us/cli/azure/install-azure-cli)

   ```bash
   brew install azure-cli
   ```

1. Authenticate to Azure

   ```bash
   # This example shows authenticating as a service principal
   az login --service-principal \
    -u=$AZURE_CLIENT_ID \
    -p=$AZURE_CLIENT_SECRET \
    -t=$AZURE_TENANT_ID

   az account set --subscription $AZURE_SUBSCRIPTION_ID
   ```

1. Install [`helm`](https://helm.sh/docs/intro/install/)

   ```bash
   brew install helm
   ```

1. Authenticate to GitHub Container Registry

   ```bash
   echo $GITHUB_TOKEN | docker login ghcr.io -u ncalteen --password-stdin
   ```

## Create an AKS Cluster

1. Create a resource group

   ```bash
   az group create --name <rg-name> --location <location>

   # e.g.
   az group create --name ncalteen --location eastus
   ```

1. Create a cluster

   ```bash
   az aks create \
    -g <rg-name> \
    -n <cluster-name> \
    --enable-managed-identity \
    --node-count 2 \
    --enable-addons monitoring \
    --generate-ssh-keys

   # e.g.
   az aks create \
    -g ncalteen \
    -n ncalteen-arc \
    --enable-managed-identity \
    --node-count 2 \
    --enable-addons monitoring \
    --generate-ssh-keys
   ```

   This command will output a `cluster.json` file and public/private keys.

## Connect to the Cluster

1. Configure `kubectl` to connect to the cluster

   ```bash
   az aks get-credentials \
    --resource-group <rg-name> \
    --name <cluster-name>

   # e.g.
   az aks get-credentials \
    --resource-group ncalteen \
    --name ncalteen-arc
   ```

1. Verify the connection

   ```bash
   kubectl get nodes
   ```

## (Optional) Create the Certificate ConfigMap

If you are using GHES and custom TLS certificates, you need to create a
ConfigMap that includes the certificate chain.

1. Concatenate the certificate chain

   ```bash
   cat root_cert > ca.crt && cat intermediate_cert_1 >> ca.crt && cat intermediate_cert_2 >> ca.crt
   ```

1. Create the ConfigMap

   ```bash
   k -n arc-runners-dev create configmap config-map-usps-dev --from-literal=ca.crt='-----BEGIN CERTIFICATE-----
   ABC123...
   -----END CERTIFICATE-----
   -----BEGIN CERTIFICATE-----
   DEF456...
   -----END CERTIFICATE-----'
   ```

## Create the Controller Configuration File

The following is an example of a `values.yaml` configuration file that can be
used to create the controller.

```yml
# Use this if the container image will be stored in another location
# besides ghcr.io
image:
  repository: 'artifactory.example.com/docker/gha-runner-scale-set-controller'
  pullPolicy: 'Always'
  tag: '0.7.0'

# Required for authentication to another image repository
imagePullSecrets:
  - 'artifactory-example-com'

# Use this to configure the proxy settings
env:
  - name: 'http_proxy'
    value: 'http://proxy.example.com:8080'
  - name: 'https_proxy'
    value: 'http://proxy.example.com:8080'
  - name: 'no_proxy'
    value: 'artifactory.example.com,10.0.0.0/8'

serviceAccount:
  create: true
  name: 'sa-arc-github'

flags:
  logLevel: 'debug'
  logFormat: 'text'
  watchSingleNamespace: 'arc-runners'
```

## Install the Controller

1. Install the operator and CRDs in the cluster

   ```bash
   helm install arc-controller
   --namespace arc-controller \
   --create-namespace \
   -f values.yaml \
   oci://ghcr.io/actions/actions-runner-controller-charts/gha-runner-scale-set-controller
   ```

1. Verify the controller is running

   ```bash
   kubectl -n arc-controller get pods
   ```

1. Check the logs in the running controller

   ```bash
   kubectl logs -n arc-controller -l app.kubernetes.io/name=gha-runner-scale-set-controller
   ```

## Configure the GitHub App

1. Follow
   [these instructions](https://docs.github.com/en/actions/hosting-your-own-runners/managing-self-hosted-runners-with-actions-runner-controller/authenticating-to-the-github-api#authenticating-arc-with-a-github-app)
1. Create the Kubernetes secret

   ```bash
   kubectl create secret generic github-app-secret \
   --namespace=arc-runners \
   --from-literal=github_app_id=<app-id> \
   --from-literal=github_app_installation_id=<installation-id> \
   --from-literal=github_app_private_key='-----BEGIN RSA PRIVATE KEY-----...'
   ```

## Create the Runner Configuration File

The following is an example of a `values.yaml` configuration file that can be
used to create the runner.

```yml
# Use this if you are authenticating to GitHub Enterprise Server
githubConfigUrl: 'https://github-dev.usps.gov/appdev'

githubConfigSecret: 'github-app-secret'

# Use this to configure the proxy settings
proxy:
  http:
    url: 'http://proxy.example.com:8080'
  https:
    url: 'http://proxy.example.com:8080'
  noProxy:
    - 'artifactory.example.com'
    - '10.0.0.0/8'

# Use this to configure the proxy settings
env:
  - name: 'http_proxy'
    value: 'http://proxy.example.com:8080'
  - name: 'https_proxy'
    value: 'http://proxy.example.com:8080'
  - name: 'no_proxy'
    value: 'artifactory.example.com,10.0.0.0/8'

minRunners: 1

runnerScaleSetName: 'arc-runners'

# Use this if you are using a custom cert chain for GHES
githubServerTLS:
  certificateFrom:
    configMapKeyRef:
      name: 'config-map-usps-dev'
      key: 'ca.crt'
  runnerMountPath: '/usr/local/share/ca-certificates/'

# Update the template if you are pulling images from another repository besides
# ghcr.io
template:
  spec:
    # Required for authentication to the image repository
    imagePullSecrets:
      - name: 'artifactory-example-com'
    containers:
      - name: 'runner'
        image: 'artifactory.example.com/docker/actions-runner:latest'
        command: ['/home/runner/run.sh']
        env:
          - name: 'ACTIONS_RUNNER_CONTAINER_HOOKS'
            value: '/home/runner/k8s/index.js'
          - name: 'ACTIONS_RUNNER_POD_NAME'
            valueFrom:
              fieldRef:
                fieldPath: 'metadata.name'
          - name: 'ACTIONS_RUNNER_REQUIRE_JOB_CONTAINER'
            value: 'false'
        volumeMounts:
          - name: 'work'
            mountPath: '/home/runner/_work'
    volumes:
      - name: 'work'
        ephemeral:
          volumeClaimTemplate:
            spec:
              accessModes: ['ReadWriteOnce']
              storageClassName: 'longhorn'
              resources:
                requests:
                  storage: '1Gi'

controllerServiceAccount:
  namespace: 'arc-controller'
  name: 'sa-arc-github'
```

## Install the Runner Scale Set

1. Install the runner scale set

   ```bash
   helm install arc-runners \
   --namespace arc-runners-dev \
   --create-namespace \
   -f values.yml \
   oci://ghcr.io/actions/actions-runner-controller-charts/gha-runner-scale-set
   ```
