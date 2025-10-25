# Helm Chart Structure

Complete directory structure for the pattern-based Istio infrastructure chart.

```
istio-namespace-infrastructure/
│
├── Chart.yaml                          # Chart metadata
├── values.yaml                         # Default configuration with pattern definitions
├── README.md                           # User documentation
├── INSTALLATION.md                     # Installation guide
├── STRUCTURE.md                        # This file
│
├── templates/                          # Helm templates directory
│   │
│   ├── _helpers.tpl                   # Helper functions and common labels
│   │
│   ├── namespace.yaml                 # Namespace creation (direct config)
│   │
│   ├── patterns-renderer.yaml         # MAIN ORCHESTRATOR
│   │                                  # Loops through patterns and renders them
│   │
│   ├── patterns/                      # Pattern template definitions
│   │   │                              # Files starting with _ are named templates
│   │   │
│   │   ├── _ingress-simple-https.yaml    # Simple HTTPS ingress pattern
│   │   ├── _ingress-mtls.yaml            # Mutual TLS ingress pattern
│   │   ├── _ingress-jwt-auth.yaml        # JWT authentication pattern
│   │   │
│   │   ├── _egress-https-api.yaml        # Simple HTTPS API egress
│   │   ├── _egress-https-resilient.yaml  # Resilient HTTPS API with circuit breaker
│   │   ├── _egress-tcp-database.yaml     # TCP database connection
│   │   └── _egress-cloud-storage.yaml    # Cloud storage optimized pattern
│   │
│   └── security/                      # Security pattern templates
│       ├── peer-authentication.yaml   # mTLS configuration
│       ├── authorization-policies.yaml # AuthZ policies (basic/zero-trust)
│       └── sidecar.yaml              # Egress control configuration
│
└── examples/                          # Example configurations for users
    ├── simple-ingress.yaml           # Basic web app with HTTPS
    ├── production-complete.yaml      # Full production setup
    ├── mtls-example.yaml             # Mutual TLS API
    ├── egress-control.yaml           # External service management
    └── zero-trust.yaml               # Zero trust security model
```

## How It Works

### 1. User Configuration (values.yaml or custom file)

User defines high-level patterns:

```yaml
ingressPatterns:
  - name: "api"
    type: "jwt-auth"
    hosts: ["api.example.com"]
    # ...

egressPatterns:
  - name: "stripe"
    type: "https-api-resilient"
    host: "api.stripe.com"
    # ...
```

### 2. Pattern Renderer (patterns-renderer.yaml)

Orchestrates rendering:
- Loops through `ingressPatterns`
- Checks pattern type
- Calls appropriate pattern template
- Loops through `egressPatterns`
- Calls appropriate pattern template

### 3. Pattern Templates (templates/patterns/_*.yaml)

Each pattern template:
- Receives pattern configuration
- Generates ALL required Istio resources
- Applies best practices automatically

Example: `_egress-https-api.yaml` creates:
- ServiceEntry
- Gateway (if using egress gateway)
- VirtualService
- DestinationRule

### 4. Security Templates (templates/security/*.yaml)

Render based on `securityPatterns` configuration:
- Peer Authentication (mTLS)
- Authorization Policies (basic/zero-trust modes)
- Sidecar (egress control)

## Template Naming Conventions

### Files Starting with `_` (Underscore)
- **Named templates** - not rendered directly
- Used via `{{ include "template-name" . }}`
- Located in `templates/patterns/`

### Regular `.yaml` Files
- **Rendered directly** by Helm
- Can include named templates
- Examples: `patterns-renderer.yaml`, `namespace.yaml`

## Adding New Patterns

To add a new pattern:

1. **Create pattern template** in `templates/patterns/`:
   ```
   templates/patterns/_ingress-custom.yaml
   ```

2. **Define the template**:
   ```yaml
   {{- define "patterns.ingress.custom" -}}
   # Your Istio resources here
   {{- end -}}
   ```

3. **Add to renderer** (`patterns-renderer.yaml`):
   ```yaml
   {{- else if eq $pattern.type "custom" }}
   {{ include "patterns.ingress.custom" (dict "pattern" $pattern "root" $) }}
   ```

4. **Document in values.yaml**:
   ```yaml
   # Pattern X: Custom Pattern
   # - name: "myapp"
   #   type: "custom"
   #   # ... configuration
   ```

5. **Create example** in `examples/`:
   ```
   examples/custom-pattern.yaml
   ```

## File Purposes

| File | Purpose | Rendered? |
|------|---------|-----------|
| `Chart.yaml` | Chart metadata | No |
| `values.yaml` | Default configuration | No |
| `templates/_helpers.tpl` | Helper functions | No |
| `templates/namespace.yaml` | Creates namespace | Yes |
| `templates/patterns-renderer.yaml` | Orchestrates patterns | Yes |
| `templates/patterns/_*.yaml` | Pattern definitions | No (via include) |
| `templates/security/*.yaml` | Security configs | Yes |
| `examples/*.yaml` | User examples | No (documentation) |

## Development Workflow

### Testing a Pattern

```bash
# Render templates to see output
helm template my-app . -f examples/simple-ingress.yaml

# Install to cluster
helm install my-app . -f examples/simple-ingress.yaml

# Debug specific pattern
helm template my-app . -f examples/simple-ingress.yaml \
  --show-only templates/patterns-renderer.yaml
```

### Validating Configuration

```bash
# Lint the chart
helm lint .

# Validate against Kubernetes
helm template my-app . -f your-values.yaml | kubectl apply --dry-run=client -f -

# Use Istio validation
helm template my-app . -f your-values.yaml | istioctl analyze -
```

## Best Practices

1. **Keep patterns atomic** - Each pattern should be self-contained
2. **Use descriptive names** - Pattern names should indicate their purpose
3. **Document all parameters** - In values.yaml with comments
4. **Provide examples** - Show real-world usage in examples/
5. **Fail fast** - Use `fail` function for invalid configurations
6. **Version carefully** - Pattern changes may break existing configs

## Migration from Individual Resources

If you have existing charts using individual Istio resources:

1. **Identify the pattern** - What does the configuration achieve?
2. **Map to pattern type** - Find or create matching pattern
3. **Convert configuration** - Translate to pattern format
4. **Test thoroughly** - Ensure same resources are generated
5. **Update documentation** - Show old vs new approach

Example:
```yaml
# Old way (50+ lines of config)
serviceEntries: [...]
virtualServices: [...]
destinationRules: [...]

# New way (5 lines)
egressPatterns:
  - name: "api"
    type: "https-api"
    host: "api.example.com"
```