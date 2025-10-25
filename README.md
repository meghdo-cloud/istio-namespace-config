# Pattern-Based Istio Infrastructure Helm Chart

**Stop configuring individual Istio resources!** This chart uses high-level patterns to automatically generate all required Istio configurations.

## 🎯 Why Patterns?

Instead of manually configuring ServiceEntries, VirtualServices, DestinationRules, Gateways, and Certificates separately (error-prone and complex), you define **what you want to achieve**, and the chart creates everything for you.

### Before (Manual Configuration)
```yaml
# You had to create: ServiceEntry, Gateway, VirtualService, DestinationRule...
# Easy to miss something or misconfigure!
serviceEntries:
  - name: stripe
    hosts: ["api.stripe.com"]
    # ... 20 more lines

virtualServices:
  - name: stripe
    # ... 30 more lines
    
destinationRules:
  - name: stripe
    # ... 25 more lines
```

### After (Pattern-Based)
```yaml
# Just say what you need!
egressPatterns:
  - name: "stripe"
    type: "https-api"
    host: "api.stripe.com"
```

**The chart automatically creates ServiceEntry, Gateway, VirtualService, and DestinationRule with best practices!**

## 🚀 Quick Start

### Example 1: Simple Web Application

```yaml
# simple-app.yaml
namespace:
  create: true
  name: "my-app"

ingressPatterns:
  - name: "webapp"
    type: "simple-https"
    hosts:
      - "www.myapp.com"
    certificate:
      issuer: "letsencrypt-prod"
```

Install:
```bash
helm install my-app ./istio-namespace-infrastructure -f simple-app.yaml
```

**That's it!** You get:
- ✅ Gateway with HTTP→HTTPS redirect
- ✅ TLS Certificate from Let's Encrypt
- ✅ Automatic certificate renewal
- ✅ mTLS between services

### Example 2: API with External Services

```yaml
# api-with-external.yaml
namespace:
  create: true
  name: "my-api"

ingressPatterns:
  - name: "api"
    type: "jwt-auth"
    hosts:
      - "api.example.com"
    certificate:
      issuer: "letsencrypt-prod"
    jwt:
      issuer: "https://auth.example.com"
      jwksUri: "https://auth.example.com/.well-known/jwks.json"

egressPatterns:
  - name: "stripe"
    type: "https-api-resilient"
    host: "api.stripe.com"
    resilience:
      timeout: "30s"
      circuitBreaker:
        consecutive5xxErrors: 5
```

You get:
- ✅ JWT authentication configured
- ✅ Authorization policies
- ✅ Egress gateway routing
- ✅ Circuit breaker for Stripe
- ✅ Automatic retries
- ✅ TLS configuration

## 📋 Available Patterns

### Ingress Patterns

#### 1. Simple HTTPS (`simple-https`)
**Use for:** Public web applications

```yaml
ingressPatterns:
  - name: "webapp"
    type: "simple-https"
    hosts:
      - "www.myapp.com"
    certificate:
      issuer: "letsencrypt-prod"
      issuerKind: "ClusterIssuer"
```

**Creates:**
- Gateway with HTTP→HTTPS redirect
- HTTPS server on port 443
- Certificate from cert-manager
- TLS 1.2+ configuration

#### 2. Mutual TLS (`mtls`)
**Use for:** APIs requiring client certificates

```yaml
ingressPatterns:
  - name: "api"
    type: "mtls"
    hosts:
      - "api.secure.com"
    certificate:
      issuer: "company-ca"
      issuerKind: "Issuer"
    mtls:
      clientCaCertificate: "LS0tLS1..." # base64 encoded
```

**Creates:**
- Gateway with MUTUAL TLS mode
- Server certificate
- Client CA certificate secret
- Authorization policy requiring valid client cert

#### 3. JWT Authentication (`jwt-auth`)
**Use for:** APIs with OAuth2/OIDC

```yaml
ingressPatterns:
  - name: "api"
    type: "jwt-auth"
    hosts:
      - "api.example.com"
    certificate:
      issuer: "letsencrypt-prod"
    jwt:
      issuer: "https://auth.example.com"
      jwksUri: "https://auth.example.com/.well-known/jwks.json"
      audiences:
        - "api.example.com"
    authorization:
      allowedRoles: ["user", "admin"]  # Optional
```

**Creates:**
- Gateway with HTTPS
- Certificate
- RequestAuthentication with JWT validation
- AuthorizationPolicy requiring valid JWT
- Optional role-based access control

### Egress Patterns

#### 1. Simple HTTPS API (`https-api`)
**Use for:** Standard external APIs

```yaml
egressPatterns:
  - name: "github"
    type: "https-api"
    host: "api.github.com"
```

**Creates:**
- ServiceEntry for external host
- Egress Gateway (if enabled)
- VirtualService routing through gateway
- DestinationRule with TLS

#### 2. Resilient HTTPS API (`https-api-resilient`)
**Use for:** Critical external services

```yaml
egressPatterns:
  - name: "payment"
    type: "https-api-resilient"
    host: "api.paymentprovider.com"
    resilience:
      timeout: "30s"
      retries: