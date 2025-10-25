# Installation Guide

This guide walks you through installing and configuring the Istio Namespace Infrastructure Helm chart.

## Table of Contents

1. [Prerequisites](#prerequisites)
2. [Quick Start](#quick-start)
3. [Step-by-Step Installation](#step-by-step-installation)
4. [Configuration](#configuration)
5. [Verification](#verification)
6. [Common Issues](#common-issues)

## Prerequisites

### Required Components

Before installing this chart, ensure you have:

1. **Kubernetes Cluster** (1.19+)
   ```bash
   kubectl version --short
   ```

2. **Helm** (3.2.0+)
   ```bash
   helm version
   ```

3. **Istio** (1.14+) installed in your cluster
   ```bash
   kubectl get namespace istio-system
   istioctl version
   ```

4. **cert-manager** (1.0+) for certificate management
   ```bash
   kubectl get namespace cert-manager
   ```

### Installing Prerequisites

#### Install Istio

```bash
# Download Istio
curl -L https://istio.io/downloadIstio | sh -
cd istio-*

# Install with default profile
istioctl install --set profile=default -y

# Verify installation
kubectl get pods -n istio-system
```

#### Install cert-manager

```bash
# Add cert-manager Helm repository
helm repo add jetstack https://charts.jetstack.io
helm repo update

# Install cert-manager
helm install cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --version v1.13.0 \
  --set installCRDs=true

# Verify installation
kubectl get pods -n cert-manager
```

#### Create Certificate Issuer

Create a ClusterIssuer for Let's Encrypt:

```yaml
# letsencrypt-issuer.yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: your-email@example.com
    privateKeySecretRef:
      name: letsencrypt-prod
    solvers:
      - http01:
          ingress:
            class: istio
```

Apply it:
```bash
kubectl apply -f letsencrypt-issuer.yaml
```

## Quick Start

### 1. Download the Chart

```bash
git clone <repository-url>
cd istio-namespace-infrastructure
```

### 2. Install with Default Values

```bash
helm install my-app . \
  --namespace my-app \
  --create-namespace
```

### 3. Install with Custom Values

```bash
# Create your values file
cat > my-values.yaml <<EOF
namespace:
  create: true
  name: my-app

ingressGateway:
  enabled: true
  servers:
    - port:
        number: 443
        name: https
        protocol: HTTPS
      hosts:
        - "myapp.example.com"
      tls:
        mode: SIMPLE

certificate:
  enabled: true
  dnsNames:
    - "myapp.example.com"
  issuerRef:
    name: "letsencrypt-prod"
    kind: "ClusterIssuer"
EOF

# Install
helm install my-app . \
  --namespace my-app \
  --create-namespace \
  --values my-values.yaml
```

## Step-by-Step Installation

### For Beginners: Simple Web Application

**Scenario**: You want to deploy a web application with HTTPS using Let's Encrypt.

1. **Create your values file**:

```yaml
# simple-app-values.yaml
namespace:
  create: true
  name: "my-web-app"
  istioInjection: true

ingressGateway:
  enabled: true
  name: "my-app-gateway"
  servers:
    - port:
        number: 80
        name: http
        protocol: HTTP
      hosts:
        - "www.myapp.com"
      tls:
        httpsRedirect: true
    - port:
        number: 443
        name: https
        protocol: HTTPS
      hosts:
        - "www.myapp.com"
      tls:
        mode: SIMPLE

certificate:
  enabled: true
  dnsNames:
    - "www.myapp.com"
  issuerRef:
    name: "letsencrypt-prod"
    kind: "ClusterIssuer"

peerAuthentication:
  enabled: true
  mode: STRICT

egressGateway:
  enabled: false
```

2. **Install the chart**:

```bash
helm install my-web-app . \
  --namespace my-web-app \
  --create-namespace \
  --values simple-app-values.yaml
```

3. **Wait for certificate**:

```bash
# Watch certificate status
kubectl get certificate -n istio-system -w

# Check if ready
kubectl get certificate tls-certificate -n istio-system -o jsonpath='{.status.conditions[?(@.type=="Ready")].status}'
```

4. **Deploy your application** (example):

```yaml
# my-app-deployment.yaml
apiVersion: v1
kind: Service
metadata:
  name: my-app
  namespace: my-web-app
spec:
  selector:
    app: my-app
  ports:
    - port: 8080
      targetPort: 8080
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
  namespace: my-web-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      containers:
        - name: app
          image: your-image:latest
          ports:
            - containerPort: 8080
---
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: my-app
  namespace: my-web-app
spec:
  hosts:
    - "www.myapp.com"
  gateways:
    - my-app-gateway
  http:
    - route:
        - destination:
            host: my-app
            port:
              number: 8080
```

Apply it:
```bash
kubectl apply -f my-app-deployment.yaml
```

### For Advanced Users: Production Setup with Egress Control

**Scenario**: Production application with controlled egress to external APIs.

1. **Use the production example**:

```bash
helm install prod-app . \
  --namespace production \
  --create-namespace \
  --values examples/production-complete.yaml
```

2. **Customize for your needs**:

Edit `examples/production-complete.yaml` to:
- Update DNS names
- Add your external service entries
- Configure authorization policies
- Set resource quotas

3. **Deploy in stages**:

```bash
# First, install without egress restrictions
helm install prod-app . \
  --namespace production \
  --create-namespace \
  --values production-values.yaml \
  --set sidecar.outboundTrafficPolicy.mode=ALLOW_ANY

# Test your application

# Then, enable strict egress control
helm upgrade prod-app . \
  --namespace production \
  --values production-values.yaml \
  --set sidecar.outboundTrafficPolicy.mode=REGISTRY_ONLY
```

## Configuration

### Updating Configuration

```bash
# Edit your values file, then:
helm upgrade my-app . \
  --namespace my-app \
  --values my-values.yaml
```

### Adding External Services After Installation

1. **Edit your values file** to add service entries:

```yaml
serviceEntries:
  - name: "new-external-api"
    hosts:
      - "api.new-service.com"
    location: MESH_EXTERNAL
    resolution: DNS
    ports:
      - number: 443
        name: https
        protocol: HTTPS
```

2. **Upgrade the release**:

```bash
helm upgrade my-app . --values my-values.yaml
```

### Enabling Features Incrementally

Start simple and add features as needed:

```bash
# 1. Start with basic ingress
helm install my-app . --set ingressGateway.enabled=true

# 2. Add certificates
helm upgrade my-app . --set certificate.enabled=true

# 3. Enable mTLS
helm upgrade my-app . --set peerAuthentication.enabled=true

# 4. Add egress control
helm upgrade my-app . --set egressGateway.enabled=true

# 5. Enable authorization policies
helm upgrade my-app . --set authorizationPolicies.defaultDeny.enabled=true
```

## Verification

### Check Deployment Status

```bash
# Check all resources
kubectl get all -n my-app

# Check Istio resources
kubectl get gateway,virtualservice,destinationrule -n my-app

# Check certificates
kubectl get certificate -n istio-system

# Check policies
kubectl get peerauthentication,authorizationpolicy -n my-app
```

### Test Ingress

```bash
# Get ingress gateway IP
export ING