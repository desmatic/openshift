# Technical Design Document: Multi-Tenant Secure OpenShift Platform with HSM-Backed Cryptography

## 1. Purpose & Scope

This document describes the target architecture for a secure multi-tenant OpenShift platform, where **secure by design** is treated as an enabler of developer velocity rather than a constraint on it: teams consume namespaces and never touch key management directly.

**Primary design goal:** Team A and Team B each receive a dedicated namespace. Neither team can interact with the other's namespace, and this isolation holds at both the access-control layer and the cryptographic layer — true zero trust, where the guarantee survives even a misconfigured policy.

**A note on doc type:** this is a **technical design document** — it describes *what* components are used, *how* they fit together, and *why*, at the level an architect or reviewer needs to sign off on the approach. It intentionally stops short of a **low-level design (LLD)** — exact YAML manifests, IP addressing, partition names, Helm values, or step-by-step build instructions. Once this design is agreed, those details belong in a follow-on implementation runbook / LLD, scoped separately (see Section 6).

## 2. Stack Components

| Component | Role |
| --- | --- |
| **Thales Luna Network HSM S750** | Hardware root of trust. FIPS 140-2 Level 3 validated. Provides PKCS#11 and KMIP interfaces, and partitioning for per-tenant key isolation. |
| **Red Hat OpenShift** on **HPE ProLiant DL360** workers | Compute/orchestration layer. Default CNI is OVN-Kubernetes. |
| **NetApp AFF A900** (ONTAP) | All-flash storage array. Multi-tenancy via Storage Virtual Machines (SVMs); volume encryption via NVE/NSE. |

**Integration stance — "the Red Hat way":** wherever a Red Hat-native or Red Hat-certified component exists, it is preferred over a community or third-party equivalent, to keep the stack inside a single support and lifecycle model:

- **OpenShift Service Mesh 3** (Sail Operator, upstream Istio) rather than raw community Istio or an alternative mesh.
- **cert-manager Operator for Red Hat OpenShift** (Red Hat product) rather than unmanaged community cert-manager.
- **OVN-Kubernetes IPsec** (built into OpenShift) rather than swapping the CNI.
- **NetApp Trident** (Red Hat-certified CSI Operator, OperatorHub) for storage provisioning.
- **HashiCorp Vault**, deployed via its certified Operator on OperatorHub, is the one non-Red-Hat component in the design — used because it is the de facto standard integration point between PKCS#11 HSMs and both etcd's KMS provider and cert-manager's issuer model. It remains inside the supported OperatorHub ecosystem rather than being a bolt-on.

## 3. Namespace Isolation Model

Team A and Team B isolation is enforced at two independent layers, deliberately overlapping so a failure in one does not compromise the other.

**Access-control layer (policy-enforced, not cryptographic):**

- NetworkPolicy / OVN-Kubernetes: default-deny ingress and egress per namespace.
- RBAC: namespace-scoped ServiceAccounts, Roles, RoleBindings.
- SCCs: restrict privileged containers, hostPath, host networking.
- ResourceQuotas/LimitRanges: prevent resource-exhaustion cross-talk.
- Optionally, dedicated node pools per team (taints/tolerations) to avoid a shared kernel.

This layer is necessary but insufficient on its own — a misconfigured policy silently weakens the boundary.

**Cryptographic layer (holds even if the access-control layer fails):**

- Per-tenant HSM partitions and per-tenant Key Encryption Keys (KEKs), so Team A's key material is physically inaccessible to Team B, independent of any Kubernetes policy.
- mTLS enforced by the service mesh, with workload identity certificates chained to an HSM-protected CA — so cross-namespace traffic is refused cryptographically, not just by NetworkPolicy.
- Storage-layer separation via distinct SVMs with distinct, HSM-managed encryption keys per team.

Section 4 details how each of these ties back to the Luna HSM.

## 4. HSM Integration by Layer

| Layer | Red Hat-native component | HSM integration |
| --- | --- | --- |
| **etcd / Secrets (control plane)** | OpenShift's KMS encryption provider (KMSv2, currently Tech Preview in OCP) | Calls out to an external KMS: HashiCorp Vault's Transit engine, deployed via the certified Vault Operator. Vault's Transit key is a Managed Key backed by a Luna HSM partition via PKCS#11 — the DEK wrapping every Secret in etcd is ultimately HSM-protected. |
| **Certificate issuance (PKI/CA)** | cert-manager Operator for Red Hat OpenShift | Issuer/ClusterIssuer points at Vault's PKI secrets engine. Vault's CA private key is a Managed Key on the Luna HSM, so every certificate issued cluster-wide is signed by a key that never leaves the HSM. |
| **Service mesh mTLS (namespace-to-namespace)** | OpenShift Service Mesh 3 (Sail Operator / upstream Istio) + istio-csr | istio-csr replaces istiod's self-signed root and requests workload certs from cert-manager. Every sidecar/ambient ztunnel identity cert authenticating (or refusing) Team A ↔ Team B traffic traces back to the HSM root. |
| **Node-to-node network encryption** | OVN-Kubernetes IPsec (native OCP CNI feature) | IKE negotiation via NSS, pointed at a PKCS#11 token (Luna client library) — node identity certs authenticating IPsec tunnels between ProLiant workers are HSM-protected, not on-disk. |
| **Storage encryption at rest** | NetApp Trident (Red Hat-certified CSI Operator) provisioning against ONTAP SVMs on the AFF A900 | ONTAP NVE/NSE encryption uses an external key manager over KMIP. The Luna Network HSM S750 natively serves as that KMIP server; each SVM (mappable 1:1 to a team) has its volume-encryption keys held in a dedicated HSM partition. |
| **HSM connectivity itself** | Luna Network HSM S750 client software (PKCS#11 / KMIP libraries) | Deployed wherever the integrations above live: inside Vault's pods (Transit/PKI Managed Keys), on OVN-Kubernetes nodes (IPsec/NSS), and configured directly as a KMIP endpoint on ONTAP (no OpenShift-side client needed there). |

**The thread that ties it together:** every layer independently calls back to the same HSM root of trust, mostly through Vault as the single OpenShift-side integration point (Transit for etcd, PKI for certs). NetApp is the one exception — ONTAP talks to the Luna S750 directly over KMIP, bypassing Vault entirely.

## 5. Cryptographic Guarantees Summary

**What this design does guarantee:**

- Data-at-rest confidentiality per tenant — distinct HSM-protected keys at both the etcd/Secrets layer and the storage/SVM layer, provided per-tenant KEKs are used rather than one cluster-wide key.
- FIPS 140-2 Level 3 hardware root of trust for all key material (HSM-resident, non-exportable private keys).
- Cryptographic workload identity — mTLS certs chained to an HSM-protected CA, enforcing namespace-to-namespace denial cryptographically, not just via network policy.
- A compromise of etcd, a mesh certificate, an IPsec node identity, or an AFF A900 volume all fail the same way: useless ciphertext without HSM access.

**What it does not automatically guarantee:**

- Isolation against a kernel-level or control-plane compromise. Team A and Team B still share the OpenShift control plane, and — unless dedicated node pools are used — the underlying kernel.
- "Zero trust between namespaces" as a cryptographic property requires the per-tenant KEK / HSM-partition design described in Section 4. Plain NetworkPolicy alone does not achieve this — it is policy enforcement, not cryptography.

## 6. Assumptions, Out of Scope, and Next Steps

**Assumptions:**

- Preference for Red Hat-native/certified components wherever one exists, even where a non-Red-Hat alternative might be marginally simpler (e.g. OpenShift Service Mesh 3 over a bare Istio install).
- HashiCorp Vault is acceptable as the single non-Red-Hat integration point, given it is deployed via a certified Operator and is the standard bridge between PKCS#11 HSMs and Kubernetes-native key consumers.
- OpenShift's KMSv2 etcd-encryption path is currently Tech Preview; this design should be revisited against its GA status before production commitment.

**Out of scope for this document:**

- Exact configuration: Vault Transit/PKI mount paths, Luna partition names and network topology, cert-manager Issuer manifests, Trident backend config, ONTAP KMIP server setup, OVN-Kubernetes IPsec policy YAML.
- HA/DR design for the Luna HSM pair, Vault cluster, and ONTAP array.
- Performance/latency budget for HSM-backed signing operations under production load.

**Next steps toward an LLD:**

1. Confirm per-tenant KEK model (dedicated HSM partition per team vs. shared partition with per-tenant key labels).
2. Validate KMSv2 etcd encryption Tech Preview status against target OCP version and support posture.
3. Produce the implementation runbook covering the exact configuration items listed above.