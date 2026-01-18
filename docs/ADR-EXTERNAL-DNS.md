# ADR: ExternalDNS for DNS Record Management

**Status:** Accepted
**Date:** 2024-10-01
**Updated:** 2026-01-17

## Context

Need automated DNS record management that:
- Syncs Kubernetes resources to external DNS providers
- Works with multiple DNS providers (Cloudflare, Hetzner, Route53, etc.)
- Integrates with k8gb for GSLB functionality
- Creates NS records for k8gb authoritative DNS delegation
- Supports multi-region deployments

## Decision

Use **ExternalDNS** for automated DNS record synchronization from Kubernetes to external DNS providers. ExternalDNS manages the **parent zone** and creates NS records delegating to **k8gb CoreDNS** for the GSLB zone.

## Architecture

```mermaid
flowchart TB
    subgraph K8s["Kubernetes Cluster"]
        SVC[Services]
        GW[Cilium Gateway]
        GSLB[Gslb CRD]
        EDNS[ExternalDNS]
        K8GB[k8gb CoreDNS<br/>Authoritative]
    end

    subgraph Providers["DNS Provider (Parent Zone)"]
        CF[Cloudflare / Hetzner DNS]
    end

    SVC -->|"Watch"| EDNS
    GW -->|"Watch"| EDNS
    GSLB -->|"Watch"| EDNS
    EDNS -->|"Create NS records"| CF
    CF -->|"Delegate gslb.example.com"| K8GB
```

## Integration with k8gb

### DNS Hierarchy

ExternalDNS and k8gb work together:

```
example.com                    → DNS Provider (managed by ExternalDNS)
  ├── app.example.com          → A record (direct, via ExternalDNS)
  ├── api.example.com          → A record (direct, via ExternalDNS)
  └── gslb.example.com (NS)    → k8gb CoreDNS (authoritative)
        ├── web.gslb.example.com   → Health-based IPs (k8gb)
        └── svc.gslb.example.com   → Health-based IPs (k8gb)
```

### Responsibilities

| Component | Responsibility |
|-----------|---------------|
| ExternalDNS | Parent zone records, NS delegation to k8gb |
| k8gb | Authoritative DNS for GSLB zone, health-based routing |

### Multi-Region Flow

```mermaid
flowchart TB
    subgraph Region1["Region 1"]
        K8GB1[k8gb CoreDNS]
        EDNS1[ExternalDNS]
    end

    subgraph Region2["Region 2"]
        K8GB2[k8gb CoreDNS]
        EDNS2[ExternalDNS]
    end

    subgraph DNS["DNS Provider"]
        Parent[Parent Zone]
        NS[NS Records]
    end

    K8GB1 <-->|"Health sync"| K8GB2
    EDNS1 -->|"Create NS"| NS
    EDNS2 -->|"Create NS"| NS
    NS -->|"Delegate"| K8GB1
    NS -->|"Delegate"| K8GB2
```

## Supported DNS Providers

| Provider | Availability | ExternalDNS Support |
|----------|--------------|---------------------|
| Cloudflare | Always | Built-in |
| Hetzner DNS | If Hetzner chosen | Built-in |
| AWS Route53 | If AWS chosen | Built-in |
| GCP Cloud DNS | If GCP chosen | Built-in |
| Azure DNS | If Azure chosen | Built-in |

## Configuration

### Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: external-dns
  namespace: external-dns
spec:
  replicas: 1
  selector:
    matchLabels:
      app: external-dns
  template:
    metadata:
      labels:
        app: external-dns
    spec:
      serviceAccountName: external-dns
      containers:
        - name: external-dns
          image: registry.k8s.io/external-dns/external-dns:v0.14.0
          args:
            - --source=service
            - --source=ingress
            - --source=crd
            - --crd-source-apiversion=k8gb.absa.oss/v1beta1
            - --crd-source-kind=Gslb
            - --provider=cloudflare  # or hetzner, aws, google, azure
            - --policy=sync
            - --registry=txt
            - --txt-owner-id=<tenant>-<region>
            - --txt-prefix=externaldns-
          env:
            - name: CF_API_TOKEN
              valueFrom:
                secretKeyRef:
                  name: cloudflare-credentials
                  key: api-token
```

### Provider-Specific Configuration

#### Cloudflare

```yaml
args:
  - --provider=cloudflare
  # Note: No --cloudflare-proxied (DDoS via cloud provider native)
env:
  - name: CF_API_TOKEN
    valueFrom:
      secretKeyRef:
        name: cloudflare-credentials
        key: api-token
```

#### Hetzner DNS

```yaml
args:
  - --provider=hetzner
env:
  - name: HETZNER_TOKEN
    valueFrom:
      secretKeyRef:
        name: hetzner-credentials
        key: api-token
```

#### AWS Route53

```yaml
args:
  - --provider=aws
  - --aws-zone-type=public
env:
  - name: AWS_ACCESS_KEY_ID
    valueFrom:
      secretKeyRef:
        name: aws-credentials
        key: access-key-id
  - name: AWS_SECRET_ACCESS_KEY
    valueFrom:
      secretKeyRef:
        name: aws-credentials
        key: secret-access-key
```

## NS Record Creation for k8gb

ExternalDNS creates NS records pointing to k8gb LoadBalancer IPs:

```yaml
# Managed by ExternalDNS in parent zone
gslb.example.com.        NS    ns1.gslb.example.com.
gslb.example.com.        NS    ns2.gslb.example.com.
ns1.gslb.example.com.    A     <k8gb-region1-lb-ip>
ns2.gslb.example.com.    A     <k8gb-region2-lb-ip>
```

## TXT Record Ownership

ExternalDNS uses TXT records to track ownership:

```
app.example.com.              A     1.2.3.4
externaldns-app.example.com.  TXT   "heritage=external-dns,external-dns/owner=tenant-region1"
```

This prevents:
- Multiple ExternalDNS instances conflicting
- Accidental deletion of manually created records

## Sync Policies

| Policy | Behavior |
|--------|----------|
| `sync` | Create, update, and delete records |
| `upsert-only` | Create and update, never delete |
| `create-only` | Only create new records |

**Recommended:** `sync` for full automation

## DDoS Protection

**Note:** Cloudflare proxy mode (`--cloudflare-proxied`) is NOT used. DDoS protection is handled by cloud provider native solutions (Hetzner Firewall, etc.).

## Consequences

**Positive:**
- Automated DNS management
- Multi-provider support
- Native k8gb integration
- GitOps-friendly (declarative)
- NS delegation enables k8gb authoritative DNS

**Negative:**
- Requires DNS provider API credentials
- TXT record overhead
- Provider-specific quirks

## Related

- [ADR-K8GB-GSLB](../../k8gb/docs/ADR-K8GB-GSLB.md)
- [SPEC-DNS-FAILOVER](../../handbook/docs/specs/SPEC-DNS-FAILOVER.md)
- [SPEC-SPLIT-BRAIN-PROTECTION](../../handbook/docs/specs/SPEC-SPLIT-BRAIN-PROTECTION.md)
