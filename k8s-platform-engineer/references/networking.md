# Kubernetes Networking Patterns

Detailed patterns for CNI, service mesh, Ingress, and Gateway API. Read this when
you're actively configuring networking for a Kubernetes cluster.

## Table of Contents

1. [CNI Configuration](#cni-configuration)
2. [Cilium Patterns](#cilium-patterns)
3. [Service Mesh Patterns](#service-mesh-patterns)
4. [Gateway API](#gateway-api)
5. [Ingress Patterns](#ingress-patterns)
6. [cert-manager Integration](#cert-manager-integration)
7. [Network Policies](#network-policies)

## CNI Configuration

### AKS: Azure CNI Powered by Cilium

This is the default for new AKS clusters. Configure via Terraform:

```hcl
resource "azurerm_kubernetes_cluster" "this" {
  network_profile {
    network_plugin      = "none"  # Required for Cilium overlay
    network_policy      = "cilium"
    pod_cidr            = "10.244.0.0/16"
    service_cidr        = "10.96.0.0/16"
    dns_service_ip      = "10.96.0.10"
  }
}
```

Or with Azure CNI overlay (recommended for large clusters):

```hcl
resource "azurerm_kubernetes_cluster" "this" {
  network_profile {
    network_plugin      = "azure"
    network_plugin_mode = "overlay"  # Enables Cilium with Azure CNI
    network_policy      = "cilium"
    pod_cidr            = "10.244.0.0/16"
  }
}
```

### EKS: AWS VPC CNI

```yaml
# EKS addon configuration
apiVersion: addons.aws.amazonaws.com/v1
kind: EKSAddon
spec:
  addonName: vpc-cni
  configurationValues: |
    enablePrefixDelegation: "true"
    warmPrefixTarget: "1"
    minIPAddressCount: "4"
```

Key settings:
- `enablePrefixDelegation: true` — assigns /28 prefixes instead of individual IPs,
  dramatically reducing IP exhaustion risk
- `warmPrefixTarget` — number of warm prefixes to keep allocated

## Cilium Patterns

### Hubble (Cilium Observability)

Hubble provides deep network observability on top of Cilium:

```yaml
# values for cilium Helm chart
hubble:
  enabled: true
  relay:
    enabled: true
  ui:
    enabled: true
  metrics:
    enabled:
      - dns
      - drop
      - tcp
      - flow
      - port-distribution
      - icmp
      - http
```

### Cilium Cluster Mesh

For connecting multiple clusters:

```yaml
clusterMesh:
  enabled: true
  clusters:
    - name: cluster1
      endpoint: "https://cluster1-api-server:6443"
    - name: cluster2
      endpoint: "https://cluster2-api-server:6443"
```

### Cilium Replacing kube-proxy

```yaml
kubeProxyReplacement: "strict"
k8sServiceHost: "your-api-server-ip"
k8sServicePort: "6443"
```

Use `"strict"` for full replacement (no kube-proxy running). Use `"disabled"` to
keep kube-proxy but not use it for Cilium-managed traffic.

### Cilium L7 Network Policy

```yaml
apiVersion: cilium.io/v2
kind: CiliumNetworkPolicy
metadata:
  name: l7-rule
spec:
  endpointSelector:
    matchLabels:
      app: myapp
  egress:
    - toEndpoints:
        - matchLabels:
            app: api
      toPorts:
        - ports:
            - port: "8080"
          rules:
            http:
              - method: GET
                path: "/api/.*"
```

This gives you L7 (HTTP-level) policy without a sidecar proxy.

## Service Mesh Patterns

### Istio Installation

```yaml
# istioctl profile: default
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
spec:
  profile: default
  meshConfig:
    accessLogFile: /dev/stdout
    defaultConfig:
      tracing:
        zipkin:
          address: tempo.observability:9411
  values:
    pilot:
      resources:
        requests:
          cpu: 500m
          memory: 2048Mi
    global:
      proxy:
        resources:
          requests:
            cpu: 100m
            memory: 128Mi
          limits:
            cpu: 500m
            memory: 512Mi
```

### Istio mTLS (PeerAuthentication)

```yaml
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: default
  namespace: istio-system
spec:
  mtls:
    mode: STRICT
```

This enforces mTLS across the entire mesh. Use `PERMISSIVE` during migration.

### Istio Traffic Management (Canary)

```yaml
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: myapp
spec:
  hosts:
    - myapp
  http:
    - route:
        - destination:
            host: myapp
            subset: stable
          weight: 90
        - destination:
            host: myapp
            subset: canary
          weight: 10
---
apiVersion: networking.istio.io/v1beta1
kind: DestinationRule
metadata:
  name: myapp
spec:
  host: myapp
  trafficPolicy:
    connectionPool:
      http:
        h2UpgradePolicy: UPGRADE
  subsets:
    - name: stable
      labels:
        version: v1
    - name: canary
      labels:
        version: v2
```

### Linkerd Installation

```yaml
# Linkerd via Helm
installNamespace: true

proxy:
  resources:
    requests:
      cpu: 50m
      memory: 64Mi
    limits:
      cpu: 200m
      memory: 256Mi

identity:
  issuer:
    scheme: kubernetes.io/tls
```

### Linkerd mTLS (Automatic)

Linkerd enables mTLS automatically by default — no PeerAuthentication needed.
Just inject the sidecar:

```yaml
annotations:
  linkerd.io/inject: enabled
```

Or at the namespace level:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: myapp
  annotations:
    linkerd.io/inject: enabled
```

## Gateway API

### Gateway Class

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: GatewayClass
metadata:
  name: cilium
spec:
  controllerName: io.cilium/gateway
```

### Gateway

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: main
  namespace: infrastructure
  annotations:
    cert-manager.io/issuer: letsencrypt-prod
spec:
  gatewayClassName: cilium
  listeners:
    - name: http
      port: 80
      protocol: HTTP
      allowedRoutes:
        namespaces:
          from: All
    - name: https
      port: 443
      protocol: HTTPS
      tls:
        mode: Terminate
        certificateRefs:
          - name: wildcard-cert
      allowedRoutes:
        namespaces:
          from: All
```

### HTTPRoute with Traffic Splitting

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: myapp
spec:
  parentRefs:
    - name: main
      namespace: infrastructure
  hostnames:
    - "myapp.example.com"
  rules:
    - backendRefs:
        - name: myapp-stable
          port: 8080
          weight: 90
        - name: myapp-canary
          port: 8080
          weight: 10
```

### GRPCRoute

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: GRPCRoute
metadata:
  name: grpc-service
spec:
  parentRefs:
    - name: main
      namespace: infrastructure
  hostnames:
    - "grpc.example.com"
  rules:
    - backendRefs:
        - name: grpc-service
          port: 50051
```

## Ingress Patterns

### NGINX Ingress Controller (AKS)

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
    external-dns.alpha.kubernetes.io/hostname: myapp.example.com
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/force-ssl-redirect: "true"
    nginx.ingress.kubernetes.io/proxy-body-size: "50m"
    nginx.ingress.kubernetes.io/configuration-snippet: |
      more_set_headers "X-Frame-Options: DENY";
      more_set_headers "X-Content-Type-Options: nosniff";
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - myapp.example.com
      secretName: myapp-tls
  rules:
    - host: myapp.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: myapp
                port:
                  number: 8080
```

### AKS Application Routing Addon

For AKS, Microsoft provides a managed NGINX ingress:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: myapp
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
spec:
  ingressClassName: webapprouting.kubernetes.azure.com
  tls:
    - hosts:
        - myapp.example.com
      secretName: myapp-tls
  rules:
    - host: myapp.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: myapp
                port:
                  number: 8080
```

## cert-manager Integration

### ClusterIssuer: Let's Encrypt with DNS-01 (Azure DNS)

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: platform@myorg.com
    privateKeySecretRef:
      name: letsencrypt-prod
    solvers:
      - dns01:
          azureDNS:
            clientIDSecretRef:
              name: cert-manager-azure-credentials
              key: CLIENT_ID
            clientSecretSecretRef:
              name: cert-manager-azure-credentials
              key: CLIENT_SECRET
            subscriptionID: "{{ .Values.azure.subscriptionId }}"
            resourceGroupName: "{{ .Values.azure.dnsResourceGroup }}"
            hostedZoneName: "example.com"
```

### ClusterIssuer: Let's Encrypt with DNS-01 (Route53)

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-prod
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: platform@myorg.com
    privateKeySecretRef:
      name: letsencrypt-prod
    solvers:
      - dns01:
          route53:
            region: us-east-1
            role: arn:aws:iam::123456789012:role/cert-manager
```

### Certificate Resource

```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: wildcard-cert
spec:
  secretName: wildcard-cert
  duration: 2160h    # 90 days
  renewBefore: 360h  # 15 days before expiry
  issuerRef:
    name: letsencrypt-prod
    kind: ClusterIssuer
  dnsNames:
    - "example.com"
    - "*.example.com"
```

## Network Policies

### Default Deny All (Namespace Level)

This is the most important network policy — start with it and then add allow rules:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: default-deny-all
  namespace: myapp
spec:
  podSelector: {}
  policyTypes:
    - Ingress
    - Egress
```

### Allow Ingress from Ingress Controller

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-from-ingress
  namespace: myapp
spec:
  podSelector: {}
  policyTypes:
    - Ingress
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              name: ingress-nginx
        - namespaceSelector:
            matchLabels:
              name: infrastructure
      ports:
        - port: 8080
          protocol: TCP
```

### Allow Egress to External Services

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-egress-external
  namespace: myapp
spec:
  podSelector: {}
  policyTypes:
    - Egress
  egress:
    # DNS (always needed)
    - to:
        - namespaceSelector: {}
          podSelector:
            matchLabels:
              k8s-app: kube-dns
      ports:
        - port: 53
          protocol: UDP
        - port: 53
          protocol: TCP
    # External HTTPS
    - ports:
        - port: 443
          protocol: TCP
    # External HTTP (if needed)
    - ports:
        - port: 80
          protocol: TCP
```

### Allow Same-Namespace Communication

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-same-namespace
  namespace: myapp
spec:
  podSelector: {}
  policyTypes:
    - Ingress
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              name: myapp
```
