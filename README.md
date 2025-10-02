# Online Boutique Application - GitOps Configuration

Kubernetes manifests for the Google Cloud microservices demo application, configured with Istio service mesh, External Secrets, and OPA Gatekeeper policies.

## Overview

This repository contains the Kubernetes configuration for deploying a cloud-native microservices application using GitOps principles with ArgoCD. The application demonstrates:

- **Microservices Architecture**: 11 microservices in multiple languages (Go, Java, Python, Node.js, C#)
- **Kustomize**: Environment-specific configuration management
- **Service Mesh**: Istio for traffic management, mTLS, and observability
- **Secrets Management**: External Secrets Operator with AWS Secrets Manager
- **Policy Enforcement**: OPA Gatekeeper for security policies
- **Security**: Pod Security Standards, AuthorizationPolicies, non-root containers

## Repository Structure

```
online-boutique-app/
├── base/                           # Base Kubernetes manifests
│   ├── adservice.yaml             # Advertisement service
│   ├── cartservice.yaml           # Shopping cart service
│   ├── checkoutservice.yaml       # Checkout processing
│   ├── currencyservice.yaml       # Currency conversion
│   ├── emailservice.yaml          # Email notifications
│   ├── frontend.yaml              # Web frontend
│   ├── paymentservice.yaml        # Payment processing
│   ├── productcatalogservice.yaml # Product catalog
│   ├── recommendationservice.yaml # Product recommendations
│   ├── redis.yaml                 # Redis cache
│   ├── shippingservice.yaml       # Shipping calculations
│   └── kustomization.yaml         # Base kustomization
│
├── overlays/
│   └── dev/
│       └── kustomization.yaml     # Development overlay with image tags
│
└── platform/                       # Platform-level resources
    ├── external-secrets/
    │   ├── aws-secrets-store.yaml        # AWS Secrets Manager integration
    │   ├── stripe-external-secret.yaml   # Stripe API key secret
    │   └── kustomization.yaml
    │
    ├── gatekeeper/
    │   ├── block-privileged-containers.yaml    # Policy template
    │   ├── privileged-container-constraint.yaml # Policy enforcement
    │   └── kustomization.yaml
    │
    └── istio/
        ├── gateway.yaml                         # Istio ingress gateway
        ├── frontend-virtual-service.yaml        # Frontend routing
        ├── istio-external-secret.yaml           # TLS certificates
        ├── argocd-authorization-policy.yaml     # ArgoCD network policy
        ├── frontend-authorization-policy.yaml   # Frontend access control
        ├── mesh-peer-authentication.yaml        # mTLS configuration
        └── kustomization.yaml
```

## Application Architecture

### Microservices

| Service | Language | Description |
|---------|----------|-------------|
| **frontend** | Go | Web UI and API gateway |
| **cartservice** | C# | Shopping cart management |
| **productcatalogservice** | Go | Product inventory |
| **currencyservice** | Node.js | Currency conversion |
| **paymentservice** | Node.js | Payment processing with Stripe |
| **shippingservice** | Go | Shipping cost calculation |
| **emailservice** | Python | Email notifications |
| **checkoutservice** | Go | Order processing |
| **recommendationservice** | Python | ML-based recommendations |
| **adservice** | Java | Contextual advertisements |
| **redis-cart** | Redis | Session and cart data |

### Service Dependencies

```
frontend
├── productcatalogservice
├── currencyservice
├── cartservice
│   └── redis-cart
├── recommendationservice
│   └── productcatalogservice
├── checkoutservice
│   ├── cartservice
│   ├── productcatalogservice
│   ├── currencyservice
│   ├── shippingservice
│   ├── emailservice
│   └── paymentservice
└── adservice
```

## Security Features

### Pod Security Standards

All microservices are configured with:
- **Non-root user**: UID/GID 1000
- **Read-only root filesystem**: Prevents runtime modifications
- **Dropped capabilities**: Removes all Linux capabilities
- **No privilege escalation**: `allowPrivilegeEscalation: false`
- **Security context**: `privileged: false`

Example from any service:
```yaml
securityContext:
  fsGroup: 1000
  runAsGroup: 1000
  runAsNonRoot: true
  runAsUser: 1000
containers:
- name: server
  securityContext:
    allowPrivilegeEscalation: false
    capabilities:
      drop: ["ALL"]
    privileged: false
    readOnlyRootFilesystem: true
```

### Istio Service Mesh

#### mTLS (Mutual TLS)
- **Strict mode**: All service-to-service communication encrypted
- **Automatic certificate rotation**: Managed by Istio CA
- **Zero-trust networking**: Services authenticate each other

#### Authorization Policies
- **Frontend isolation**: Blocks direct service-to-service access to frontend
- **ArgoCD protection**: Prevents application pods from accessing ArgoCD namespace
- **Least privilege**: Each service can only access required dependencies

#### Gateway Configuration
- **HTTP to HTTPS redirect**: Enforced at ingress
- **TLS termination**: Using certificates from AWS Secrets Manager
- **Single entry point**: All external traffic through Istio gateway

### OPA Gatekeeper Policies

**Privileged Container Prevention**:
- Blocks any pod with `privileged: true`
- Applies to all namespaces except `kube-system`
- Prevents container escape vulnerabilities

### External Secrets Integration

**Secrets are never stored in Git**:
- Stripe API key synced from AWS Secrets Manager
- TLS certificates for Istio gateway
- Automatic rotation support (1-minute refresh interval)
- IRSA (IAM Roles for Service Accounts) authentication

## Deployment

### Prerequisites

1. EKS cluster with infrastructure from `eks-online-boutique` repository
2. ArgoCD installed and configured
3. External Secrets Operator installed
4. OPA Gatekeeper installed
5. Istio installed

### Deploy Platform Resources

```bash
# Apply platform-level configuration (Istio, policies, secrets)
kubectl apply -f platform/
```

### Deploy Application via ArgoCD

Create ArgoCD Application resources:

```yaml
# Application
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: online-boutique
  namespace: argocd
spec:
  project: default
  destination:
    server: https://kubernetes.default.svc
    namespace: online-boutique
  source:
    repoURL: https://github.com/bosingva/online-boutique-app.git
    path: overlays/dev
    targetRevision: main
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
    - CreateNamespace=true
```

```yaml
# Platform
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: platform
  namespace: argocd
spec:
  project: default
  destination:
    server: https://kubernetes.default.svc
    namespace: default
  source:
    repoURL: https://github.com/bosingva/online-boutique-app.git
    path: platform
    targetRevision: main
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
```

### Manual Deployment (for testing)

```bash
# Deploy platform resources first
kubectl apply -k platform/

# Deploy application
kubectl apply -k overlays/dev/

# Verify deployment
kubectl get pods -n online-boutique
kubectl get pods -n istio-ingress
```

## Configuration Management

### Kustomize Overlays

The `overlays/dev` directory customizes base manifests for the development environment:

- **Image tags**: Pins specific versions (v0.8.0)
- **Image sources**: Uses Google Container Registry
- **Environment-specific configs**: Can add ConfigMaps, resource limits, etc.

### Adding New Environments

Create a new overlay:

```bash
mkdir -p overlays/production
```

Create `overlays/production/kustomization.yaml`:

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization
namespace: online-boutique
resources:
  - ../../base

# Production-specific configurations
images:
  - name: gcr-frontend
    newName: your-registry/frontend
    newTag: v1.0.0

# Add production resources
patchesStrategicMerge:
  - production-resources.yaml
```

## Secrets Configuration

### Required Secrets in AWS Secrets Manager

1. **Stripe API Key** (`stripe-api-key`):
```json
{
  "stripe-key": "sk_test_your_stripe_secret_key"
}
```

2. **Istio TLS Certificate** (`dev/istio-tls-cert`):
```
-----BEGIN CERTIFICATE-----
Your TLS certificate
-----END CERTIFICATE-----
```

3. **Istio TLS Private Key** (`dev/istio-tls-key`):
```
-----BEGIN PRIVATE KEY-----
Your TLS private key
-----END PRIVATE KEY-----
```

### Creating Secrets

```bash
# Stripe API key
aws secretsmanager create-secret \
  --name stripe-api-key \
  --secret-string '{"stripe-key":"sk_test_your_key"}' \
  --region us-east-1

# TLS certificate
aws secretsmanager create-secret \
  --name dev/istio-tls-cert \
  --secret-string file://tls.crt \
  --region us-east-1

# TLS private key
aws secretsmanager create-secret \
  --name dev/istio-tls-key \
  --secret-string file://tls.key \
  --region us-east-1
```

## Istio Traffic Management

### Accessing the Application

1. **Get the Load Balancer DNS**:
```bash
kubectl get svc -n istio-ingress
```

2. **Access via HTTPS**:
```bash
# If using custom domain with TLS
https://your-domain.com

# If using NLB DNS directly (will show certificate warning)
https://<nlb-dns-name>
```

### Traffic Routing

All HTTP traffic is redirected to HTTPS:
```yaml
servers:
  - port:
      number: 80
      name: http-80
      protocol: HTTP
    hosts:
      - "*"
    tls:
      httpsRedirect: true  # Automatic redirect
```

## Monitoring and Observability

### Health Checks

All services implement:
- **Readiness probes**: gRPC or HTTP health checks
- **Liveness probes**: Automatic restart on failure

### Istio Observability

With Istio sidecars, you automatically get:
- **Distributed tracing**: Request flow across services
- **Metrics**: Latency, error rates, request volume (RED metrics)
- **Service graph**: Visual representation of dependencies

### Recommended Additions

Add to `platform/` directory:
- Prometheus for metrics collection
- Grafana for visualization
- Jaeger or Zipkin for distributed tracing
- Kiali for service mesh visualization

## Testing

### Verify Deployment

```bash
# Check pod status
kubectl get pods -n online-boutique

# Check Istio sidecar injection
kubectl get pods -n online-boutique -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{.spec.containers[*].name}{"\n"}{end}'

# Should show: server, istio-proxy
```

### Verify mTLS

```bash
# Check peer authentication
kubectl get peerauthentication -n online-boutique

# Verify mTLS status
istioctl authn tls-check <pod-name>.<namespace>
```

### Verify Gatekeeper Policies

```bash
# List constraints
kubectl get constraints

# Test privileged container (should be denied)
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: test-privileged
  namespace: online-boutique
spec:
  containers:
  - name: test
    image: nginx
    securityContext:
      privileged: true
EOF
# Should fail with policy violation
```

### Verify External Secrets

```bash
# Check secret sync status
kubectl get externalsecrets -n online-boutique
kubectl get externalsecrets -n istio-ingress

# Verify secrets were created
kubectl get secret stripe-api-key -n online-boutique
kubectl get secret gateway-tls -n istio-ingress
```

## Troubleshooting

### Pods Not Starting

```bash
# Check pod events
kubectl describe pod <pod-name> -n online-boutique

# Common issues:
# 1. Image pull errors - check image names in kustomization
# 2. Secrets missing - verify External Secrets sync
# 3. Istio sidecar issues - check istio-proxy logs
```

### Service Communication Issues

```bash
# Check Istio configuration
kubectl get virtualservices -n online-boutique
kubectl get destinationrules -n online-boutique

# Check authorization policies
kubectl get authorizationpolicies -A

# Test connectivity from pod
kubectl exec -it <pod-name> -n online-boutique -c server -- /bin/sh
# Then: curl http://productcatalogservice:3550
```

### External Secrets Not Syncing

```bash
# Check ExternalSecret status
kubectl describe externalsecret stripe-api-secret -n online-boutique

# Check SecretStore
kubectl describe clustersecretstore aws-secret-store

# Verify IRSA permissions
kubectl describe sa externalsecrets-sa -n online-boutique
```

### Gateway/Ingress Issues

```bash
# Check gateway configuration
kubectl describe gateway istio-gateway -n istio-ingress

# Check TLS secret
kubectl get secret gateway-tls -n istio-ingress

# Check Istio gateway logs
kubectl logs -n istio-ingress -l istio=ingressgateway
```

## Development Workflow

### Making Changes

1. **Update manifests** in `base/` or create overlay patches
2. **Commit changes** to Git
3. **Push to repository**
4. **ArgoCD auto-syncs** (if configured) or manual sync:
```bash
argocd app sync online-boutique
argocd app sync platform
```

### Testing Changes Locally

```bash
# Preview changes with Kustomize
kubectl kustomize overlays/dev

# Dry-run deployment
kubectl apply -k overlays/dev --dry-run=client

# Apply to test namespace
kubectl apply -k overlays/dev -n test
```

## Security Best Practices

- Store secrets in AWS Secrets Manager, never in Git
- Use least-privilege RBAC for service accounts
- Enable Pod Security Standards
- Regularly update container images
- Review and update OPA Gatekeeper policies
- Monitor authorization policy violations
- Rotate secrets periodically

## Performance Optimization

### Resource Limits (Currently Commented Out)

Uncomment and tune resource requests/limits in base manifests:

```yaml
resources:
  requests:
    cpu: 100m
    memory: 64Mi
  limits:
    cpu: 200m
    memory: 128Mi
```

Start with conservative limits and adjust based on actual usage.

### Horizontal Pod Autoscaling

Add HPA resources:

```yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: frontend
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: frontend
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
```

## Related Repositories

- **Infrastructure**: [eks-online-boutique](https://github.com/bosingva/eks-online-boutique) - Terraform code for EKS cluster

## Contributing

This is a demonstration repository for portfolio purposes. For the original application, see:
- [Google Cloud Microservices Demo](https://github.com/GoogleCloudPlatform/microservices-demo)

## License

Apache 2.0 (inherited from Google Cloud Microservices Demo)

---

**Author**: [Your Name]  
**Contact**: [GitHub](https://github.com/bosingva) | [LinkedIn](your-linkedin)