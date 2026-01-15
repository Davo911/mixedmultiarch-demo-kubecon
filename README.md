# kubecon-multarch-demo

This demo showcases KubeVirt’s ability to schedule virtual machines to the correct architecture-specific nodes in a heterogeneous OpenShift/Kubernetes cluster.


## Overview

The demo consists of:
- **Landing Page**: A styled web page with navigation buttons (served from a regular pod)
- **s390x VM**: Fedora VM running on IBM Z architecture, serving a mainframe-themed page
- **x86_64 VM**: Fedora VM running on x86 architecture, serving an AI/ML-themed page

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                      OpenShift Cluster                      │
├─────────────────────────────────────────────────────────────┤
│                                                             │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐       │
│  │  s390x Node  │  │  s390x Node  │  │  x86 Node    │       │
│  │  t313lp06/07 │  │  t313lp08    │  │  compute-1   │       │
│  ├──────────────┤  ├──────────────┤  ├──────────────┤       │
│  │              │  │              │  │              │       │
│  │              │  │ ┌──────────┐ │  │ ┌──────────┐ │       │
│  │              │  │ │ fedora-  │ │  │ │ fedora-  │ │       │
│  │              │  │ │ s390x VM │ │  │ │ x86 VM   │ │       │
│  │              │  │ │ (nginx)  │ │  │ │ (nginx)  │ │       │
│  │              │  │ └──────────┘ │  │ └──────────┘ │       │
│  └──────────────┘  └──────────────┘  └──────────────┘       │
│                                                             │
│  ┌────────────────────────────────────────────────┐         │
│  │  Landing Page Pod (nginx)                      │         │
│  │  - Can run on any node (multi-arch image)      │         │
│  └────────────────────────────────────────────────┘         │
│                                                             │
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

## VM-Configs

### x86

```yaml
apiVersion: kubevirt.io/v1
kind: VirtualMachine
metadata:
  name: fedora-x86
  namespace: default
  labels:
    app: fedora-x86
    demo: multiarch
spec:
  running: true
  template:
    metadata:
      labels:
        app: fedora-x86
        demo: multiarch
    spec:
      nodeSelector:
        kubernetes.io/hostname: compute-1.ocp6.test.mainz
      terminationGracePeriodSeconds: 0
      domain:
        cpu:
          cores: 2
        resources:
          requests:
            memory: 2Gi
        devices:
          disks:
          - name: containerdisk
            disk: {}
          - name: cloudinitdisk
            disk: {}
          interfaces:
          - name: default
            masquerade: {}
            ports:
            - name: http
              port: 80
      networks:
      - name: default
        pod: {}
      volumes:
      - name: containerdisk
        containerDisk:
          image: quay.io/containerdisks/fedora:latest
      - name: cloudinitdisk
        cloudInitNoCloud:
          secretRef:
            name: fedora-x86-cloudinit

```

### s390x

```yaml
apiVersion: kubevirt.io/v1
kind: VirtualMachine
metadata:
  name: fedora-s390x
  namespace: default
  labels:
    app: fedora-s390x
    demo: multiarch
spec:
  running: true
  template:
    metadata:
      labels:
        app: fedora-s390x
        demo: multiarch
    spec:
      architecture: s390x
      nodeSelector:
        kubernetes.io/arch: s390x
      terminationGracePeriodSeconds: 0
      domain:
        cpu:
          cores: 2
        resources:
          requests:
            memory: 2Gi
        devices:
          disks:
          - name: containerdisk
            disk: {}
          - name: cloudinitdisk
            disk: {}
          interfaces:
          - name: default
            masquerade: {}
            ports:
            - name: http
              port: 80
      networks:
      - name: default
        pod: {}
      volumes:
      - name: containerdisk
        containerDisk:
          image: quay.io/containerdisks/fedora:latest
      - name: cloudinitdisk
        cloudInitNoCloud:
          secretRef:
            name: fedora-s390x-cloudinit

```

## Setup Instructions

### Step 1: Prepare Environment

```bash
# Clone or create the demo directorycd /path/to/MultiArchDemo
# Set your cluster domain and namespaceexport NAMESPACE=default
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
# Create ConfigMap with VM HTML filesoc create configmap vm-html-files \  --from-file=index_s390x.html \  --from-file=index_x86.html \  -n ${NAMESPACE}# Deploy HTML serveroc apply -f html-server.yaml
```

Get the HTML server’s ClusterIP:

```bash
HTML_SERVER_IP=$(oc get svc html-server -n ${NAMESPACE} -o jsonpath='{.spec.clusterIP}')echo "HTML Server IP: $HTML_SERVER_IP"
```

Create cloud-init secrets (update the IP in `update-cloudinit-curl.yaml`):

```bash
oc apply -f update-cloudinit-curl.yaml
```

### Step 4: Deploy VMs

```bash
# Apply VM manifestsoc apply -f s390x-vm.yaml
oc apply -f x86-vm.yaml
# Wait for VMs to be ready (takes 2-3 minutes)oc wait --for=condition=Ready vmi/fedora-s390x vmi/fedora-x86 \  -n ${NAMESPACE} --timeout=5m
# Verify VMs are running on correct nodesoc get vmi -n ${NAMESPACE} -o wide
```

Expected output:

```
NAME           AGE   PHASE     IP             NODENAME                    READY
fedora-s390x   5m    Running   10.130.0.127   t313lp08.ocp6.test.mainz    True
fedora-x86     5m    Running   10.130.2.80    compute-1.ocp6.test.mainz   True
```

### Step 5: Deploy Services

```bash
# Create ClusterIP servicesoc apply -f services.yaml
# Create NodePort services for direct accessoc apply -f nodeport-services.yaml
# Verify servicesoc get svc -n ${NAMESPACE} | grep svc-
```

### Step 6: Deploy Landing Page

```bash
# Create ConfigMap with landing page HTMLoc create configmap multiarch-landing \  --from-file=index.html \  -n ${NAMESPACE}# Deploy landing pageoc apply -f landing-deployment.yaml
# Wait for pod to be readyoc wait --for=condition=Ready pod \  -l app=multiarch-landing \  -n ${NAMESPACE} --timeout=2m
```

### Step 7: (Optional) Create OpenShift Routes

If you want to use DNS-based access instead of NodePort:

```bash
oc apply -f routes.yaml
oc get routes -n ${NAMESPACE}
```

## Architecture-Specific Notes

### s390x (IBM Z)

- Machine type: `s390-ccw-virtio-rhel9.6.0`
- Scheduled via `nodeSelector: kubernetes.io/arch: s390x`

### x86_64

- Machine type: Automatically selected (pc-q35 family)
- Scheduled via `nodeSelector: kubernetes.io/hostname: compute-1.ocp6.test.mainz`
