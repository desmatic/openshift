Act as a Senior Infrastructure & Security Architect specializing in Red Hat OpenShift. 

I need you to perform a technical deep-dive on observability for a proposed zero-trust, multi-tenant AI platform. Include support for metrics, logs, and traces. Solutions should be grounded in vendor best practices and what is reasonably feasible. If a desired feature imposes unreasonable administrative burden or is unsupported, highlight it.

### STACK SPECIFICATION:

1. Compute & Acceleration:
   - HPE ProLiant Compute DL380 Gen12 (Dual Intel Xeon 6, 144 total cores, 4TB DDR5 RAM) worker nodes
   - 3x NVIDIA L4 GPUs (Single-wide, 24GB GDDR6 VRAM each, Ada Lovelace) per worker node
   - Intel NUMA architecture tuning (Single-NUMA-Node policy via OpenShift Topology Manager)

2. Storage & Data Platform:
   - NetApp AFF A900 Trident CSI ONTAP storage array.

3. Orchestration & Databases:
   - Red Hat OpenShift (Latest release)
   - CloudNativePG (CNPG) Operator with PostgresML & pgvector extensions
   - NVIDIA GPU Operator (Time-slicing configured for multi-tenant inference)

4. Identity, Security & Cryptography:
   - Luna Network S750 HSM (Hardware Root of Trust via PKCS#11 and KMIP)
   - HashiCorp Vault Enterprise (Auto-unscaled via Luna HSM, managing dynamic database credentials and Istio Intermediate CA)
   - Active Directory (LDAP Group Sync to OpenShift RBAC, user authentication)
   - OpenShift Service Mesh / Istio in Ambient Mode (STRICT mTLS, Layer 7 AuthorizationPolicies for cross-tenant Pub/Sub)
   - OpenShift OAuth Proxy (Sidecar pattern enforcing AD-based HTTP access)
   - Red Hat Advanced Cluster Security (ACS) for runtime security & container image scanning

5. Design Principles & Architecture Choices:
   - Zero-Trust / Default-Deny posture between namespaces.
   - Self-service multitenancy (ResourceQuotas, LimitRanges, local `admin` RBAC).
   - Storage-level data reduction preservation (No application-layer TDE; encryption enforced at storage array layer backed by HSM).
   - All persistant volume claims are on the storage array.
   - API Limitations: End-users/developers must only have access to standard Kubernetes/OpenShift APIs (PVCs, StorageClasses) and must never have direct access to storage array APIs or management interfaces.
   - "The Red Hat way": wherever a Red Hat-native or Red Hat-certified component exists, it is preferred over a community or third-party equivalent, to keep the stack inside a single support and lifecycle model.

6. HSM integration with
   - etcd
   - Certificate issuance (PKI/CA)
   - Service mesh mTLS (namespace-to-namespace)
   - Node-to-node network encryption
   - Storage array encryption at rest
   - Hashicorp Vault

### RESEARCH OBJECTIVES:

Please conduct a thorough technical analysis and provide a structured report addressing the following:

1. Single Pane of Glass:
   - a multi-tenant namespace single pane of glass.
   - a place for for multiple teams to publish SLIs, SLOs, and metrics, and dashboards.
   - cluster wide metrics like available cpu, memory, disk, etc.

2. Application Team Pane of Glass:
   - a single tenant namespace pane of glass.
   - Provide a step-by-step verification process to ensure Team A cannot read Team B's pane of glass.
   - It should support logs, metrics, and application traces.

2. Cluster Administrators Pane of Glass:
   - An administrative pane of glass for certificates, traffic flows, and cluster wide diagnostics.
   - Options, risk analysis, and feasibility of providing applications teams read only access.
   - Options, risk analysis, and feasibility of providing administrators access to application teams pane of glass.

Tone & Style: Technical, authoritative, highly specific, and actionable. Avoid high-level marketing language. Focus on real-world engineering constraints, CLI configurations, YAML snippets, and architectural edge cases.