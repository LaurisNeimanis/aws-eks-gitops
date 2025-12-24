# Low-Level Architecture — AWS EKS GitOps Platform

This document provides a **low-level, implementation-oriented view** of the GitOps platform,
complementing the high-level architecture described in the main README.

The focus here is on **control flow, reconciliation boundaries, and runtime responsibilities**
across Terraform, ArgoCD, ApplicationSets, and Kubernetes workloads.

---

## Control Plane & Reconciliation Flow

```mermaid
flowchart TD
    subgraph AWS
        NLB[AWS Network Load Balancer<br/>ACM TLS Termination]
        EKS[EKS Cluster]
    end

    subgraph Git
        REPO[aws-eks-gitops<br/>Git Repository]
    end

    subgraph ArgoCD
        ROOT[Root Application<br/>App-of-Apps]
        APPSET[ApplicationSets]
        ARGO[ArgoCD Controller]
    end

    subgraph Platform
        TRAEFIK[Traefik<br/>Ingress Controller]
        EXTDNS[ExternalDNS]
    end

    subgraph Workloads
        WHOAMI[whoami<br/>Deployment + Service]
        CCORE[ccore-ai<br/>Frontend + Backend]
    end

    REPO --> ROOT
    ROOT --> APPSET
    APPSET --> ARGO

    ARGO --> TRAEFIK
    ARGO --> EXTDNS
    ARGO --> WHOAMI
    ARGO --> CCORE

    NLB --> TRAEFIK
    TRAEFIK --> WHOAMI
    TRAEFIK --> CCORE
```

---

## Ingress & Traffic Flow

```mermaid
sequenceDiagram
    participant Client
    participant DNS as Cloudflare DNS
    participant NLB as AWS NLB (ACM)
    participant Traefik
    participant App as Kubernetes Service

    Client->>DNS: Request demo.ccore.ai
    DNS-->>Client: CNAME → ingress.ccore.ai

    Client->>NLB: HTTPS request (443)
    NLB->>Traefik: Forward decrypted traffic

    Traefik->>Traefik: EntryPoint redirect (HTTP → HTTPS)
    Traefik->>App: Route via IngressRoute
    App-->>Client: HTTP response
```

---

## GitOps Responsibility Boundaries

### Terraform (out of scope for this repo)
- VPC, subnets, routing
- EKS cluster provisioning
- IAM, IRSA, load balancer integration
- ACM certificate provisioning

### ArgoCD
- Continuous reconciliation engine
- Drift detection and self-healing
- ApplicationSet expansion
- Namespace lifecycle (CreateNamespace)

### ApplicationSets
- Environment-aware application instantiation
- Horizontal scalability across apps and environments
- Single source of truth for application definitions

### Platform Layer
- Traefik ingress controller
- Global HTTP → HTTPS enforcement at entryPoint level
- ExternalDNS integration with Cloudflare
- Shared ingress endpoint (`ingress.ccore.ai`)

### Workload Layer
- Deployments and Services
- IngressRoute definitions only
- No platform logic duplication
- No infrastructure awareness

---

## Design Characteristics

- No circular dependencies
- Clear ownership boundaries
- Global ingress behavior enforced once
- Application definitions remain minimal and explicit
- GitOps reconciliation is the only mutation path

This low-level view reflects the **actual runtime behavior** of the platform,
not just conceptual intent.
