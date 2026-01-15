# mixedmultiarch-demo-kubecon

This demo showcases KubeVirt’s ability to schedule virtual machines to the correct architecture-specific nodes in a heterogeneous OpenShift/Kubernetes cluster.

## Overview

The demo consists of:

- **Landing Page**: A styled web page with navigation buttons (served from a regular pod)
- **s390x VM**: Fedora VM running on IBM Z architecture, serving a mainframe-themed page
- **x86_64 VM**: Fedora VM running on x86 architecture, serving an AI/ML-themed page

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                      OpenShift Cluster                       │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐     │
│  │  s390x Node  │  │  s390x Node  │  │  x86 Node    │     │
│  │  t313lp06/07 │  │  t313lp08    │  │  compute-1   │     │
│  ├──────────────┤  ├──────────────┤  ├──────────────┤     │
│  │              │  │              │  │              │     │
│  │              │  │ ┌──────────┐ │  │ ┌──────────┐ │     │
│  │              │  │ │ fedora-  │ │  │ │ fedora-  │ │     │
│  │              │  │ │ s390x VM │ │  │ │ x86 VM   │ │     │
│  │              │  │ │ (nginx)  │ │  │ │ (nginx)  │ │     │
│  │              │  │ └──────────┘ │  │ └──────────┘ │     │
│  └──────────────┘  └──────────────┘  └──────────────┘     │
│                                                              │
│  ┌────────────────────────────────────────────────┐         │
│  │  Landing Page Pod (nginx)                      │         │
│  │  - Can run on any node (multi-arch image)     │         │
│  └────────────────────────────────────────────────┘         │
│                                                              │
│  Access via NodePort:                                       │
│  - :30080 → Landing Page                                    │
│  - :30081 → s390x VM                                        │
│  - :30082 → x86 VM                                          │
└─────────────────────────────────────────────────────────────┘
```

## Prerequisites

- OpenShift cluster with:
  - At least one x86_64 node
  - At least one s390x node
  - KubeVirt operator installed (OpenShift Virtualization)
- `oc` CLI configured and authenticated
- `kubectl` (optional, oc is sufficient)
- Network access to the cluster nodes

## Files Structure

```
MultiArchDemo/
├── README.md                    # This file
├── index.html                   # Landing page HTML
├── index_s390x.html            # s390x VM page content
├── index_x86.html              # x86 VM page content
├── s390x-vm.yaml               # s390x VM definition
├── x86-vm.yaml                 # x86 VM definition
├── services.yaml               # ClusterIP services
├── routes.yaml                 # OpenShift Routes (optional)
├── nodeport-services.yaml      # NodePort services for direct IP access
├── landing-deployment.yaml     # Landing page deployment
├── landing-configmap.yaml      # ConfigMap with landing page HTML
├── html-server.yaml            # Internal HTTP server for VMs
└── update-cloudinit-curl.yaml  # VM cloud-init configs
```

## Setup Instructions

### Step 1: Prepare Environment

```bash
# Clone or create the demo directory
cd /path/to/MultiArchDemo

# Set your cluster domain and namespace
export NAMESPACE=default
export DOMAIN=apps.ocp6.test.mainz  # Adjust to your cluster
```

### Step 2: Create HTML Files

Ensure you have three HTML files:

- `index.html` - Landing page with navigation
- `index_s390x.html` - Content for s390x VM
- `index_x86.html` - Content for x86 VM

### Step 3: Create Cloud-Init Secrets

First, deploy an internal HTML server to serve the content:

```bash
# Create ConfigMap with VM HTML files
oc create configmap vm-html-files \
  --from-file=index_s390x.html \
  --from-file=index_x86.html \
  -n ${NAMESPACE}

# Deploy HTML server
oc apply -f html-server.yaml
```

Get the HTML server's ClusterIP:

```bash
HTML_SERVER_IP=$(oc get svc html-server -n ${NAMESPACE} -o jsonpath='{.spec.clusterIP}')
echo "HTML Server IP: $HTML_SERVER_IP"
```

Create cloud-init secrets (update the IP in `update-cloudinit-curl.yaml`):

```bash
oc apply -f update-cloudinit-curl.yaml
```

### Step 4: Deploy VMs

```bash
# Apply VM manifests
oc apply -f s390x-vm.yaml
oc apply -f x86-vm.yaml

# Wait for VMs to be ready (takes 2-3 minutes)
oc wait --for=condition=Ready vmi/fedora-s390x vmi/fedora-x86 \
  -n ${NAMESPACE} --timeout=5m

# Verify VMs are running on correct nodes
oc get vmi -n ${NAMESPACE} -o wide
```

Expected output:

```
NAME           AGE   PHASE     IP             NODENAME                    READY
fedora-s390x   5m    Running   10.130.0.127   t313lp08.ocp6.test.mainz    True
fedora-x86     5m    Running   10.130.2.80    compute-1.ocp6.test.mainz   True
```

### Step 5: Deploy Services

```bash
# Create ClusterIP services
oc apply -f services.yaml

# Create NodePort services for direct access
oc apply -f nodeport-services.yaml

# Verify services
oc get svc -n ${NAMESPACE} | grep svc-
```

### Step 6: Deploy Landing Page

```bash
# Create ConfigMap with landing page HTML
oc create configmap multiarch-landing \
  --from-file=index.html \
  -n ${NAMESPACE}

# Deploy landing page
oc apply -f landing-deployment.yaml

# Wait for pod to be ready
oc wait --for=condition=Ready pod \
  -l app=multiarch-landing \
  -n ${NAMESPACE} --timeout=2m
```

### Step 7: (Optional) Create OpenShift Routes

If you want to use DNS-based access instead of NodePort:

```bash
oc apply -f routes.yaml
oc get routes -n ${NAMESPACE}
```

## Accessing the Demo

### Option A: Direct NodePort Access (Recommended for Behind-Gateway Clusters)

Use any cluster node IP with the NodePort:

```bash
# Get a node IP
NODE_IP=$(oc get nodes -o jsonpath='{.items[0].status.addresses[?(@.type=="InternalIP")].address}')

# Access URLs
echo "Landing Page: http://${NODE_IP}:30080"
echo "s390x VM:     http://${NODE_IP}:30081"
echo "x86 VM:       http://${NODE_IP}:30082"

# Open in browser (macOS)
open http://${NODE_IP}:30080
```

### Option B: OpenShift Routes (If DNS is configured)

```bash
echo "Landing Page: http://multiarch-demo-default.${DOMAIN}"
echo "s390x VM:     http://s390x-demo-default.${DOMAIN}"
echo "x86 VM:       http://x86-demo-default.${DOMAIN}"
```

### For Remote Access via SSH Tunnel

If accessing from a local machine through a gateway:

```bash
# On your local machine, create an SSH tunnel
sshuttle -r <gateway-host> 172.23.0.0/16 10.128.0.0/14 --dns

# Then access via node IPs
open http://172.23.239.6:30080
```

## Testing

```bash
# Test all endpoints
curl -I http://${NODE_IP}:30080  # Should return 200 OK
curl -I http://${NODE_IP}:30081  # Should return 200 OK
curl -I http://${NODE_IP}:30082  # Should return 200 OK

# Verify VM content
curl -s http://${NODE_IP}:30081 | grep '<title>'  # "Hello from IBM Z"
curl -s http://${NODE_IP}:30082 | grep '<title>'  # "Hello from x86"
```

## Troubleshooting

### VMs Not Starting

Check VM events:

```bash
oc describe vm fedora-s390x -n ${NAMESPACE}
oc describe vmi fedora-s390x -n ${NAMESPACE}
```

Common issues:

- **Machine type mismatch**: Ensure `architecture: s390x` is set in the VM spec
- **Node selector issues**: Check node labels match the VM's nodeSelector
- **Resource constraints**: Verify nodes have enough memory/CPU

### VMs Running but Not Serving Content

Check if cloud-init completed:

```bash
# Get virt-launcher pod logs
POD=$(oc get pods -n ${NAMESPACE} -l vm.kubevirt.io/name=fedora-s390x -o name)
oc logs -n ${NAMESPACE} $POD -c compute --tail=50
```

Check if HTML was downloaded:

```bash
# Console into VM (requires virtctl)
virtctl console fedora-s390x

# Inside VM:
ls -la /usr/share/nginx/html/
systemctl status nginx
```

### Services Not Accessible

Check service endpoints:

```bash
oc get endpoints -n ${NAMESPACE} | grep svc-
```

If no endpoints exist, the service selector may be wrong. Verify pod labels:

```bash
oc get pods -n ${NAMESPACE} -l vm.kubevirt.io/name=fedora-s390x --show-labels
```

### HTML Server Issues

```bash
# Check HTML server
oc get pods -n ${NAMESPACE} -l app=html-server
oc logs -n ${NAMESPACE} -l app=html-server

# Test HTML server from within cluster
oc run test-curl --image=curlimages/curl --rm -i --restart=Never -- \
  curl -I http://html-server.${NAMESPACE}.svc.cluster.local/index_s390x.html
```

## Cleanup

Remove all demo resources:

```bash
# Delete VMs
oc delete vm fedora-s390x fedora-x86 -n ${NAMESPACE}

# Delete services
oc delete svc svc-s390x svc-x86 svc-landing \
  svc-s390x-nodeport svc-x86-nodeport svc-landing-nodeport \
  html-server -n ${NAMESPACE}

# Delete routes (if created)
oc delete route s390x-demo-default x86-demo-default multiarch-demo-default -n ${NAMESPACE}

# Delete deployments and ConfigMaps
oc delete deployment multiarch-landing html-server -n ${NAMESPACE}
oc delete configmap multiarch-landing vm-html-files html-files-server -n ${NAMESPACE}

# Delete secrets
oc delete secret fedora-s390x-cloudinit fedora-x86-cloudinit -n ${NAMESPACE}

# Delete any leftover jobs
oc delete job copy-html-files -n ${NAMESPACE} --ignore-not-found
```

## Key Concepts Demonstrated

1. **Architecture-Aware Scheduling**: VMs are automatically scheduled to nodes matching their architecture
2. **Multi-Architecture Support**: Shows x86_64 and s390x running side-by-side
3. **KubeVirt Integration**: VMs managed as Kubernetes resources
4. **Cloud-Init**: VMs are configured at boot time with cloud-init
5. **Service Mesh**: Both VMs exposed via Kubernetes Services and Routes

## Important Notes

- The `architecture: s390x` field in the VM spec is critical for proper scheduling
- Cloud-init can take 2-3 minutes to install packages and configure the VM
- NodePort services (30080-30082) provide direct access without DNS
- The demo uses Fedora container disk images from `quay.io/containerdisks/fedora:latest`
- VMs use masquerade networking (NAT) for external connectivity

## Architecture-Specific Notes

### s390x (IBM Z)

- Machine type: `s390-ccw-virtio-rhel9.6.0`
- Scheduled via `nodeSelector: kubernetes.io/arch: s390x`
- Ideal for COBOL workloads, mainframe applications

### x86_64

- Machine type: Automatically selected (pc-q35 family)
- Scheduled via `nodeSelector: kubernetes.io/hostname: compute-1.ocp6.test.mainz`
- Ideal for GPU workloads, AI/ML applications

## References

- [KubeVirt Documentation](https://kubevirt.io/user-guide/)
- [OpenShift Virtualization](https://docs.openshift.com/container-platform/latest/virt/about_virt/about-virt.html)
- [Cloud-Init Documentation](https://cloudinit.readthedocs.io/)
