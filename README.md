# Traefik with HTTPS using cert-manager

This documentation explains how to set up Traefik as an ingress controller with HTTPS using cert-manager in a Kubernetes cluster managed by Flux CD.

## Components Overview

- **Traefik**: Modern HTTP reverse proxy and load balancer for microservices
- **cert-manager**: Kubernetes add-on to automate the management of TLS certificates
- **Flux CD**: GitOps operator that ensures the cluster state matches the desired state in Git

## Architecture

```
                                  ┌─────────────────┐
                                  │                 │
                                  │  Let's Encrypt  │
                                  │                 │
                                  └────────┬────────┘
                                           │
                                           │ ACME
                                           │
┌─────────────┐  HTTPS   ┌─────────────┐   │   ┌─────────────┐
│             │  ◄─────► │             │◄──┘   │             │
│   Users     │          │   Traefik   │       │cert-manager │
│             │          │             │◄─────►│             │
└─────────────┘          └──────┬──────┘       └─────────────┘
                                │
                                │
                         ┌──────▼──────┐
                         │             │
                         │ Application │
                         │             │
                         └─────────────┘
```

## Prerequisites

- Kubernetes cluster
- Flux CD installed
- kubectl access to the cluster

## Installation

### 1. Namespace Setup

Create the required namespaces:

```yaml
apiVersion: v1
kind: Namespace
metadata:
  name: internal
---
apiVersion: v1
kind: Namespace
metadata:
  name: istio-system
---
apiVersion: v1
kind: Namespace
metadata:
  name: cert-manager
  labels:
    istio-injection: disabled
```

### 2. Install cert-manager

cert-manager is installed via Flux HelmRelease:

```yaml
apiVersion: source.toolkit.fluxcd.io/v1beta2
kind: HelmRepository
metadata:
  name: cert-manager
  namespace: flux-system
spec:
  interval: 1h
  url: https://charts.jetstack.io
---
apiVersion: helm.toolkit.fluxcd.io/v2beta1
kind: HelmRelease
metadata:
  name: cert-manager
  namespace: cert-manager
spec:
  interval: 15m
  chart:
    spec:
      chart: cert-manager
      version: "1.18.0"
      sourceRef:
        kind: HelmRepository
        name: cert-manager
        namespace: flux-system
  values:
    crds:
      enabled: true
```

### 3. Install Traefik

Traefik is installed via Flux HelmRelease:

```yaml
apiVersion: source.toolkit.fluxcd.io/v1
kind: HelmRepository
metadata:
  name: traefik
  namespace: flux-system
spec:
  interval: 5m0s
  url: https://traefik.github.io/charts
---
apiVersion: helm.toolkit.fluxcd.io/v2
kind: HelmRelease
metadata:
  name: traefik-crd
  namespace: internal
spec:
  interval: 5m
  chart:
    spec:
      chart: traefik-crds
      sourceRef:
        kind: HelmRepository
        name: traefik
        namespace: flux-system
      version: "1.9.0-rc1"
  values:
    gatewayAPI: true
---
apiVersion: helm.toolkit.fluxcd.io/v2
kind: HelmRelease
metadata:
  name: traefik
  namespace: internal
spec:
  interval: 5m
  chart:
    spec:
      chart: traefik
      sourceRef:
        kind: HelmRepository
        name: traefik
        namespace: flux-system
      version: "36.2.0-rc1"
  values:
    ingressRoute:
      dashboard:
        enabled: true
```

## HTTPS Configuration

### 1. Create an ACME Issuer

Create an Issuer or ClusterIssuer to obtain certificates from Let's Encrypt:

```yaml
apiVersion: cert-manager.io/v1
kind: Issuer
metadata:
  name: acme
  namespace: internal
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    privateKeySecretRef:
      name: acme
    solvers:
      - http01:
          ingress:
            ingressClassName: traefik
```

For production use, ensure you're using the production Let's Encrypt server.

### 2. Request a Certificate

Request a certificate for your domain:

```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: domain-cert
  namespace: internal
spec:
  secretName: domain-tls
  dnsNames:
    - "your-domain.example.com"
  issuerRef:
    name: acme
    kind: Issuer
```

### 3. Configure Traefik IngressRoute with TLS

Create an IngressRoute with TLS enabled:

```yaml
apiVersion: traefik.io/v1alpha1
kind: IngressRoute
metadata:
  name: sample-app
  namespace: internal
spec:
  entryPoints:
    - websecure
  routes:
    - match: Host(`your-domain.example.com`)
      kind: Rule
      services:
        - name: sample-app
          port: 80
  tls:
    secretName: domain-tls
```

## Example Application

Here's a complete example of deploying an NGINX application with HTTPS:

```yaml
apiVersion: v1
kind: Service
metadata:
  name: sample-app
  namespace: internal
spec:
  selector:
    app: sample-app
  ports:
  - port: 80
    targetPort: 80
    protocol: TCP
    name: http 
---
apiVersion: traefik.io/v1alpha1
kind: IngressRoute
metadata:
  name: sample-app
  namespace: internal
spec:
  entryPoints:
    - websecure
  routes:
    - match: Host(`nginx.example.com`)
      kind: Rule
      services:
        - name: sample-app
          port: 80
  tls:
    secretName: domain-tls
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: sample-app
  namespace: internal
spec:
  replicas: 2
  selector:
    matchLabels:
      app: sample-app
  template:
    metadata:
      labels:
        app: sample-app
    spec:
      containers:
      - name: sample-app
        image: nginx:stable
        ports:
        - containerPort: 80
        resources:
          limits:
            cpu: 100m
            memory: 128Mi
          requests:
            cpu: 50m
            memory: 64Mi
---
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: nginx-cert
  namespace: internal
spec:
  secretName: domain-tls
  dnsNames:
    - "nginx.example.com"
  issuerRef:
    name: acme
    kind: Issuer
---
apiVersion: cert-manager.io/v1
kind: Issuer
metadata:
  name: acme
  namespace: internal
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    privateKeySecretRef:
      name: acme
    solvers:
      - http01:
          ingress:
            ingressClassName: traefik
```

## Troubleshooting

### Check Certificate Status

```bash
kubectl get certificates -n internal
kubectl get certificaterequests -n internal
kubectl get challenges -n internal
```

### Check Traefik Logs

```bash
kubectl logs -n internal -l app.kubernetes.io/name=traefik
```

### Check cert-manager Logs

```bash
kubectl logs -n cert-manager -l app=cert-manager
```

## Advanced Configuration

### Using ClusterIssuer

For cluster-wide certificate issuance:

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: acme-cluster-issuer
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    privateKeySecretRef:
      name: acme-cluster-issuer
    solvers:
      - http01:
          ingress:
            ingressClassName: traefik
```

### DNS Validation

For wildcard certificates:

```yaml
apiVersion: cert-manager.io/v1
kind: Issuer
metadata:
  name: acme-dns
  namespace: internal
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    privateKeySecretRef:
      name: acme-dns
    solvers:
      - dns01:
          # Configure your DNS provider here
          cloudflare:
            email: user@example.com
            apiTokenSecretRef:
              name: cloudflare-api-token
              key: api-token
```

### Rate Limiting

Be aware of Let's Encrypt rate limits:
- 50 certificates per registered domain per week
- 5 duplicate certificates per week
- 5 failed validations per account, per hostname, per hour
