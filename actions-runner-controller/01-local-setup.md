# Set Up ARC Locally

This document outlines how to configure ARC to run locally using `minikube`.

## Prerequisites

1. Install [`minikube`](https://minikube.sigs.k8s.io/docs/start/)

   ```bash
   brew install minikube
   ```

1. Start `minikube`

   ```bash
   minikube start
   ```

1. Install [`helm`](https://helm.sh/docs/intro/install/)

   ```bash
   brew install helm
   ```

1. Authenticate to GitHub Container Registry

   ```bash
   echo $GITHUB_TOKEN | docker login ghcr.io -u ncalteen --password-stdin
   ```

## Install the Controller

1. Install the operator and CRDs in the cluster

   ```bash
   helm install arc-controller-dev \
     --namespace "arc-controller-dev" \
     --create-namespace \
     --set serviceAccount.name="sa-github-runner-dev" \
     oci://ghcr.io/actions/actions-runner-controller-charts/gha-runner-scale-set-controller
   ```

1. Verify the controller is running

   ```bash
   kubectl -n arc-controller-dev get pods
   ```

1. Check the logs in the running controller

   ```bash
   kubectl logs -n arc-controller-dev -l app.kubernetes.io/name=gha-runner-scale-set-controller
   ```

## Configure the GitHub App

1. Follow
   [these instructions](https://docs.github.com/en/actions/hosting-your-own-runners/managing-self-hosted-runners-with-actions-runner-controller/authenticating-to-the-github-api#authenticating-arc-with-a-github-app)
1. Create the Kubernetes secret

   ```bash
   kubectl create secret generic github-app-secret \
   --namespace=arc-runners-dev \
   --from-literal=github_app_id=658277 \
   --from-literal=github_app_installation_id=44538880 \
   --from-literal=github_app_private_key='-----BEGIN RSA PRIVATE KEY-----...'
   ```

## Install the Runner Scale Set

1. Install the runner scale set

   ```bash
   helm install arc-runner-dev \
   --namespace arc-runner-dev \
   --create-namespace \
   --set githubConfigUrl="https://github.com/ncalteen-avocado" \
   --set githubConfigSecret="github-app-secret" \
   --set runnerScaleSetName="arc-runner-dev" \
   --set controllerServiceAccount.namespace="arc-controller-dev" \
   --set controllerServiceAccount.name="sa-github-runner-dev" \
   oci://ghcr.io/actions/actions-runner-controller-charts/gha-runner-scale-set
   ```
