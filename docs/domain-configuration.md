# Domain Configuration Guide

This repository contains **example domains owned by the author**
and is not intended to be used with those domains as-is.

To run this platform in another environment,
all domain-related values must be updated.

---

## Overview

Domain-related configuration affects:

- ExternalDNS behavior
- Ingress routing
- TLS termination
- Public service exposure

These values are **environment-specific** and must be aligned with:
- your DNS provider
- your domain ownership
- your ACM certificates (if applicable)

---

## Files requiring domain changes

### 1. ExternalDNS domain filters

**File**
```
apps/platform/external-dns/values/values.yaml
```

**Field**
```yaml
domainFilters:
  - ccore.ai
```

**Action**
Replace `ccore.ai` with a domain you own and manage
in your DNS provider.

Example:
```yaml
domainFilters:
  - example.com
```

---

### 2. Traefik service DNS hostname

**File**
```
apps/platform/traefik/values/values.yaml
```

**Field**
```yaml
external-dns.alpha.kubernetes.io/hostname: ingress.ccore.ai
```

**Action**
Update to the ingress hostname used in your environment.

Example:
```yaml
external-dns.alpha.kubernetes.io/hostname: ingress.example.com
```

This hostname becomes the **single shared ingress endpoint**
for all workloads.

---

### 3. Workload IngressRoute host rules

Each workload defines its own hostnames.

#### whoami example

**File**
```
apps/workloads/whoami/base/ingressroute-https.yaml
```

```yaml
match: Host(`whoami.ccore.ai`)
```

Replace with:
```yaml
match: Host(`whoami.example.com`)
```

---

#### ccore-ai example

**File**
```
apps/workloads/ccore-ai/base/ingressroute-https.yaml
```

```yaml
match: Host(`demo.ccore.ai`)
```

Replace with:
```yaml
match: Host(`demo.example.com`)
```

---

### 4. AWS ACM certificate (TLS termination on NLB)

This platform assumes that **TLS is terminated at the AWS Network Load Balancer**
using a **pre-existing ACM certificate**.

The certificate **must exist**:
- in the **same AWS account** as the EKS cluster
- in the **same AWS region**
- and cover the **ingress hostname** used by Traefik

#### File

```
apps/platform/traefik/values/values.yaml
```

#### Field

```yaml
service.beta.kubernetes.io/aws-load-balancer-ssl-cert: "arn:aws:acm:eu-central-1:277679348320:certificate/18a3f75f-1fb2-461d-9aa2-c3b60591d773"
```

#### Action

Replace the placeholder ACM certificate ARN with **your own certificate ARN**.

Example:
```yaml
service.beta.kubernetes.io/aws-load-balancer-ssl-cert: "arn:aws:acm:eu-central-1:123456789012:certificate/xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
```

⚠️ **This value is mandatory**

If the ARN is invalid, missing, or belongs to another account:
- the AWS NLB will **fail to provision**
- HTTPS ingress will **not function**

#### Certificate requirements

The ACM certificate **must cover**:
- the ingress hostname (e.g. `ingress.example.com`)
- and optionally wildcard subdomains (e.g. `*.example.com`)

Certificate issuance and DNS validation are handled **outside of this repository**
by the Terraform infrastructure layer.

---

## DNS ownership requirements

You must control the DNS zone for the chosen domain.

This setup assumes:
- ExternalDNS manages records automatically
- DNS provider credentials are supplied via Kubernetes Secret
- No manual DNS changes are required after bootstrap

---

## What NOT to change here

- No TLS certificates are created in this repository
- No ACM resources are provisioned here
- No DNS zones are created here

All infrastructure-level DNS and certificate lifecycle
is handled by the **Terraform infrastructure repository**.

---

## Summary

Before bootstrap, ensure that:

- All `ccore.ai` references are replaced
- Domain ownership matches your environment
- ExternalDNS credentials are valid
- A valid ACM certificate ARN is configured for Traefik

This keeps the GitOps layer portable, predictable,
and safe to reuse across clusters.
