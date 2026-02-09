# Application Manifests

This directory contains Kubernetes manifests for the application managed by ArgoCD.  


## Files

### deployment.yaml
Defines the application deployment with:
- **Namespace**: `dev`
- **Image**: `svvoii/iot-app:v1` (public Docker Hub repository)
- **Container port**: 8888
- **Replicas**: 1 pod

This file controls which version of the application runs in the cluster. Change the image tag (v1 → v2) to update the application version.

### service.yaml
Exposes the application using a NodePort service:
- **Type**: NodePort
- **NodePort**: 30080 (mapped to localhost:8888 via K3d)
- **Target port**: 8888 (container port)

This service makes the application accessible at `http://localhost:8888` from your host machine.

---

## ArgoCD GitOps Workflow

ArgoCD continuously monitors this GitHub repository and automatically syncs (every 3 minutes) the cluster state to match these manifests:

1. **Current State**: These files define the desired state
2. **Detection**: ArgoCD polls this repo every 3 minutes
3. **Sync**: When changes are detected, ArgoCD updates the cluster
4. **Self-Heal**: Manual changes to the cluster are automatically reverted

## Accessing ArgoCD UI

Login to ArgoCD in the browser:

1. Port forward ArgoCD API server (run in terminal from project root):
```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

2. URL: `http://localhost:8080`
3. Username: `admin`
4. Password: (from `argocd-initial-admin-secret`)
```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d && echo
```

---

## Docker Image

This is a simple Flask application that is packaged into a Docker image and hosted on Docker Hub.  
The app returns JSON responses with version information.    
The image is used in the `deployment.yaml` manifest to run the application in the Kubernetes cluster.

The application image on Docker Hub:
- **Repository**: `https://hub.docker.com/r/svvoii/iot-app`
- **Versions**: 
  - `svvoii/iot-app:v1` - Returns `{"status":"ok", "message":"v1", ...}`
  - `svvoii/iot-app:v2` - Returns `{"status":"ok", "message":"v2", ...}`

---

## Version Switching

To switch versions:

1. Edit `deployment.yaml`:
```yaml
image: svvoii/iot-app:v2  # Changed from v1 to v2
```

2. Commit and push:
```bash
git add deployment.yaml
git commit -m "Update to v2"
git push
```

3. Wait 1-3 minutes for ArgoCD to sync

4. Verify:
```bash
curl http://localhost:8888/
```

---

## ArgoCD Application Setup

!!! *NOTE*: The ArgoCD Application resource is defined in `argocd-application.yaml` at the project root and it is NOT a part of this repo. It is applied separately to the cluster to tell ArgoCD where to find these manifests and how to manage them.  

### `argocd-application.yaml` example:
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: iot-app
  namespace: argocd
spec:
  project: default
  source:
    repoURL: https://github.com/svvoii/sbocanci-iot-argocd.git
    targetRevision: main
    path: .
  destination:
    server: https://kubernetes.default.svc
    namespace: dev
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - CreateNamespace=true
```

### Key points:
- **repoURL**: Points to this GitHub repository
- **path**: Root of the repository (where `deployment.yaml` and `service.yaml` are located)
- **destination**: Deploys to `dev` namespace in the cluster
- **syncPolicy**: Enables automated syncing and self-healing

---
