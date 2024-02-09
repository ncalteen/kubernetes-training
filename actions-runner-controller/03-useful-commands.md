# Useful Troubleshooting Commands

## Get the ARC Helm Charts

```bash
helm pull oci://ghcr.io/actions/actions-runner-controller-charts/gha-runner-scale-set-controller
helm pull oci://ghcr.io/actions/actions-runner-controller-charts/gha-runner-scale-set
```

## Copy the Images to a Repository

1. Pull the images

   ```bash
   # Update the version number as needed
   docker pull ghcr.io/actions/gha-runner-scale-set-controller:X.Y.Z
   docker pull ghcr.io/actions/gha-runner-scale-set-controller:latest
   ```

1. Tag the images

   ```bash
   # Update the version number as needed
   docker tag ghcr.io/actions/gha-runner-scale-set-controller:X.Y.Z artifactory.example.com/docker/gha-runner-scale-set-controller:X.Y.Z
   docker tag ghcr.io/actions/gha-runner-scale-set-controller:latest artifactory.example.com/docker/gha-runner-scale-set-controller:latest
   ```

1. Push the images

```bash
docker push artifactory.example.com/docker/gha-runner-scale-set-controller:latest
```
