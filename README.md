# OpenStack Resource Controller (ORC) GitOps Demo

This repository demonstrates **GitOps-based multi-cloud OpenStack infrastructure management** using [OpenStack Resource Controller (ORC)](https://github.com/k-orc/openstack-resource-controller) and [ArgoCD](https://argoproj.github.io/cd/). 

Demo video available through this link: https://youtu.be/mCYMpRV4yVw
## Overview

This demo showcases how to:
- Manage OpenStack infrastructure as Kubernetes Custom Resources
- Deploy the same infrastructure topology to multiple OpenStack clouds
- Use GitOps patterns for declarative infrastructure management
- Leverage Kustomize for environment-specific configuration

## What Gets Deployed

Each environment (PSI and Vexxhost) deploys an identical topology:

```
┌─────────────────────────────────────────┐
│  External Network (Public Internet)     │
└──────────────┬──────────────────────────┘
               │
        ┌──────▼──────┐
        │   Router    │
        └──────┬──────┘
               │
    ┌──────────┴───────────┐
    │                      │
┌───▼────┐          ┌──────▼──────┐
│Bastion │          │   Internal  │
│Server  │◄─────────┤   Network   │
│        │          │             │
└────────┘          └──────┬──────┘
                           │
                    ┌──────▼──────┐
                    │   Private   │
                    │   Server    │
                    └─────────────┘
```

**Resources Created:**
- 2 Servers (Bastion with external access, Private internal-only)
- 1 Internal Network + Subnet
- 1 Router (connects internal network to external)
- 1 Security Group (SSH access)
- References to existing external network, flavors, and default security group

## Architecture

```
GitHub Repo ──┐
              │
              ▼
         ┌─────────┐
         │ ArgoCD  │
         └────┬────┘
              │
              ▼
      ┌───────────────┐
      │  Kubernetes   │
      │  Cluster      │
      │               │
      │  ┌─────────┐  │
      │  │   ORC   │  │──────► PSI OpenStack Cloud
      │  │Controller│  │
      │  └─────────┘  │──────► Vexxhost OpenStack Cloud
      └───────────────┘
```

## Prerequisites

- **Kubernetes cluster** (minikube, kind, or any K8s cluster v1.25+)
- **kubectl** configured to access your cluster
- **OpenStack credentials** for PSI and/or Vexxhost clouds
- **Git** for version control

## Quick Start

### 1. Deploy Infrastructure Components

```bash
# Deploy ArgoCD
kubectl apply -k infrastructure/argocd

# Deploy OpenStack Resource Controller (ORC)
kubectl apply -k infrastructure/orc

# Wait for ArgoCD to be ready
kubectl wait --for=condition=available --timeout=600s deployment/argocd-server -n argocd
```

### 2. Access ArgoCD UI

Get the admin password:
```bash
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d && echo
```

Port-forward to access the UI:
```bash
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

Open your browser to `https://localhost:8080`
- **Username:** `admin`
- **Password:** (from the command above)

### 3. Configure OpenStack Cloud Credentials

Create cloud configuration files for your OpenStack clouds.

**For PSI (`~/.config/openstack/clouds-psi.yaml`):**
```yaml
clouds:
  openstack:
    auth:
      auth_url: https://psi-openstack-url:5000/v3
      username: your-username
      password: your-password
      project_name: your-project
      user_domain_name: Default
      project_domain_name: Default
    region_name: RegionOne
```

**For Vexxhost (`~/.config/openstack/clouds-vexxhost.yaml`):**
```yaml
clouds:
  vexxhost:
    auth:
      auth_url: https://vexxhost-openstack-url:5000/v3
      username: your-username
      password: your-password
      project_name: your-project
      user_domain_name: Default
      project_domain_name: Default
    region_name: RegionOne
```

Create Kubernetes secrets from these files:
```bash
# PSI cloud credentials
kubectl create secret generic cloud-config \
  --from-file=clouds.yaml=/path/to/clouds-psi.yaml \
  -n psi

# Vexxhost cloud credentials
kubectl create secret generic cloud-config \
  --from-file=clouds.yaml=/path/to/clouds-vexxhost.yaml \
  -n vexxhost
```

### 4. Deploy ArgoCD Applications

```bash
# Apply ArgoCD applications
kubectl apply -f argocd-app-psi.yaml
kubectl apply -f argocd-app-vexxhost.yaml

# Verify applications are created
kubectl get applications -n argocd
```

### 5. Sync Applications in ArgoCD UI

1. Open ArgoCD UI at `https://localhost:8080`
2. You'll see `psi-demo` and `vexxhost-demo` applications
3. Click on each application
4. Click the **"SYNC"** button to deploy resources
5. Watch as ORC creates the infrastructure in your OpenStack clouds!

### 6. Verify Resources

Check the Kubernetes Custom Resources:
```bash
# View all OpenStack resources
kubectl get servers,ports,networks,routers,securitygroups -n psi
kubectl get servers,ports,networks,routers,securitygroups -n vexxhost
```

Check status of specific resources:
```bash
# Get detailed status
kubectl describe server orcdemo-bastion -n psi

# Watch resources get created
kubectl get openstack -A -w
```

## Repository Structure

```
.
├── argocd-app-psi.yaml              # ArgoCD Application for PSI
├── argocd-app-vexxhost.yaml         # ArgoCD Application for Vexxhost
├── bases/
│   └── orcdemo/                     # Base OpenStack resources
│       ├── bastion-server.yaml      # Bastion server with external access
│       ├── private-server.yaml      # Private server (internal only)
│       ├── internal-network.yaml    # Internal network + subnet
│       ├── router.yaml              # Router connecting networks
│       ├── securitygroup.yaml       # SSH security group
│       └── image.yaml               # Cirros test image
├── components/
│   └── kustomizeconfig/             # Kustomize configuration
├── environments/
│   ├── psi/                         # PSI-specific config
│   │   ├── kustomization.yaml
│   │   ├── external-network.yaml    # PSI external network reference
│   │   ├── flavors.yaml             # PSI flavor mappings
│   │   └── default-securitygroup.yaml
│   └── vexxhost/                    # Vexxhost-specific config
│       ├── kustomization.yaml
│       ├── external-network.yaml    # Vexxhost external network reference
│       ├── flavors.yaml             # Vexxhost flavor mappings
│       └── default-securitygroup.yaml
└── infrastructure/
    ├── argocd/                      # ArgoCD installation
    └── orc/                         # ORC installation
```

## Key Concepts

### OpenStack Resource Controller (ORC)

ORC is a Kubernetes operator that:
- Extends Kubernetes with OpenStack Custom Resource Definitions (CRDs)
- Manages OpenStack resources (servers, networks, etc.) as Kubernetes objects
- Continuously reconciles desired state (in Git) with actual state (in OpenStack)
- Provides self-healing: if someone deletes a resource in OpenStack, ORC recreates it

### GitOps with ArgoCD

ArgoCD:
- Monitors this Git repository for changes
- Automatically syncs Kubernetes state with Git
- Provides visualization of deployment status
- Enables rollback, diff viewing, and sync management

### Kustomize Overlays

The repository uses Kustomize to:
- Define base resources once (`bases/orcdemo/`)
- Apply environment-specific overlays (`environments/psi/`, `environments/vexxhost/`)
- Manage different cloud credentials and resource IDs per environment
- Keep configuration DRY (Don't Repeat Yourself)

### Management Policies

Resources use different management policies:

- **`managed`**: ORC creates and manages the resource (servers, networks, etc.)
- **`unmanaged`**: Resource must already exist in OpenStack; ORC just imports a reference (external networks, flavors, default security groups)

## Customization

### Adding a New Environment

1. Create a new directory under `environments/`:
```bash
mkdir -p environments/new-cloud
```

2. Create a `kustomization.yaml`:
```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

components:
- ../../components/kustomizeconfig

resources:
- ../../bases/orcdemo
- external-network.yaml
- flavors.yaml
- default-securitygroup.yaml

patches:
- target:
    group: openstack.k-orc.cloud
    version: v1alpha1
  patch: |-
    - op: replace
      path: /spec/cloudCredentialsRef/cloudName
      value: new-cloud
```

3. Create environment-specific files (external-network.yaml, flavors.yaml, etc.)

4. Create an ArgoCD Application YAML

5. Create the cloud-config secret with credentials

### Modifying the Topology

Edit files in `bases/orcdemo/` to change the base topology. Changes will apply to all environments unless overridden.

For environment-specific changes, add patches in the environment's `kustomization.yaml`.

## Troubleshooting

### Resources stuck in "Waiting for Secret"
Ensure the `cloud-config` secret exists in the namespace:
```bash
kubectl get secret cloud-config -n psi
kubectl get secret cloud-config -n vexxhost
```

### Resources showing "no cloud named: X"
The `cloudName` in the resource doesn't match any cloud in your `clouds.yaml`. Verify:
```bash
kubectl get secret cloud-config -n <namespace> -o yaml
# Decode the clouds.yaml data and check cloud names
```

### ArgoCD shows "OutOfSync"
Click **"Refresh"** in the ArgoCD UI, then **"Sync"** to apply changes.

### ORC Controller not running
```bash
kubectl get pods -n orc-system
kubectl logs -n orc-system deployment/orc-controller-manager
```

### View ORC Controller Logs
```bash
kubectl logs -n orc-system deployment/orc-controller-manager -f
```

## Cleanup

To remove all resources:

```bash
# Delete ArgoCD applications (this will delete OpenStack resources)
kubectl delete -f argocd-app-psi.yaml
kubectl delete -f argocd-app-vexxhost.yaml

# Wait for resources to be cleaned up in OpenStack
kubectl get openstack -A -w

# Delete infrastructure
kubectl delete -k infrastructure/orc
kubectl delete -k infrastructure/argocd

# Delete namespaces
kubectl delete namespace psi vexxhost
```

## Demo Workflow

This demo is designed to showcase:

1. **Declarative Infrastructure**: Define infrastructure as code in Git
2. **Multi-Cloud Management**: Same topology deployed to different clouds
3. **Self-Healing**: Delete a server in OpenStack → ORC recreates it
4. **GitOps**: Change Git → ArgoCD syncs → ORC updates OpenStack
5. **Visualization**: ArgoCD UI shows real-time status of all resources

## Resources

- [OpenStack Resource Controller](https://github.com/k-orc/openstack-resource-controller)
- [ArgoCD Documentation](https://argo-cd.readthedocs.io/)
- [Kustomize Documentation](https://kustomize.io/)
