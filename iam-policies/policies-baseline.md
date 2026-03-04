# IAM Policies Baseline — OKE Virtual Node Pool Cross-Compartment

> **All policies validated empirically during PoC in eu-milan-1.**  
> Sources: Oracle official documentation + Oracle Support SR evidence.

---

## Architectural Context — Why IAM Is Complex Here

```
OCI Tenancy (Customer)
│
├── cmp-collaudo        ← Cluster Compartment  (OKE cluster lives here)
└── cmp-network         ← Network Compartment  (VCN, subnets live here)
         ↑
         Sibling compartments — no shared parent below root
```

```
OKE Service Tenancy (Oracle-managed)
│
└── ske_compartment     ← Virtual Nodes and Container Instances live HERE
                           NOT in the customer tenancy
```

> **This is the fundamental difference from Managed Nodes.**  
> With Managed Nodes, compute instances are in YOUR tenancy → Instance Principal works naturally.  
> With Virtual Nodes, the Container Instance is in the **Oracle OKE service tenancy** → the endorse
> cross-tenancy mechanism is required to allow the Virtual Node principal to attach VNICs to YOUR subnets.

---

## Policy Layer 1 — Endorse (Cross-Tenancy) — MANDATORY FOR ALL TENANCIES

> 📌 Source: [Getting Started and Best Practices with OKE Virtual Nodes — Oracle Blog](https://blogs.oracle.com/cloud-infrastructure/getting-started-best-practices-oke-virtual-nodes)  
> 📌 Source: [Oracle University Podcast — Working with OKE Virtual Nodes](https://oracleuniversitypodcast.libsyn.com/working-with-oke-virtual-nodes)

This policy layer is **required regardless of whether you are a tenancy administrator or not**.  
It must be created in the **root compartment** of the customer tenancy.  
The OCIDs of `ske` tenancy and `ske_compartment` are **fixed Oracle values — do not change them**.

### What it does
The Virtual Node (running in the Oracle OKE service tenancy) needs to attach a VNIC to a subnet in YOUR VCN and associate NSG rules defined in YOUR tenancy.  
Without these endorse statements, the Virtual Node **cannot create Container Instances** because it has no authorization to touch your network resources.

### Policy statements (root compartment — customer tenancy)

```
define tenancy ske as ocid1.tenancy.oc1..aaaaaaaacrvwsphodcje6wfbc3xsixhzcan5zihki6bvc7xkwqds4tqhzbaq
define compartment ske_compartment as ocid1.compartment.oc1..aaaaaaaa2bou6r766wmrh5zt3vhu2rwdya7ahn4dfdtwzowb662cmtdc5fea

endorse any-user to associate compute-container-instances
  in compartment ske_compartment of tenancy ske
  with subnets in tenancy
  where ALL { request.principal.type = 'virtualnode',
              request.operation      = 'CreateContainerInstance' }

endorse any-user to associate compute-container-instances
  in compartment ske_compartment of tenancy ske
  with vnics in tenancy
  where ALL { request.principal.type = 'virtualnode',
              request.operation      = 'CreateContainerInstance' }

endorse any-user to associate compute-container-instances
  in compartment ske_compartment of tenancy ske
  with network-security-group in tenancy
  where ALL { request.principal.type = 'virtualnode',
              request.operation      = 'CreateContainerInstance' }
```

> ⚠️ **These three endorse statements are non-negotiable.**  
> Without them, every Pod scheduling attempt fails — the Virtual Node cannot attach to your VCN.  
> The `ske` OCID and `ske_compartment` OCID are Oracle-managed constants, not customer-specific.

---

## Policy Layer 2 — User Group Policies (Non-Administrator Users)

> 📌 Source: [Required IAM Policies for Using Virtual Nodes](https://docs.oracle.com/en-us/iaas/Content/ContEng/Tasks/contengvirtualnodes-Required_IAM_Policies.htm)

### 2.1 Cluster Compartment (cmp-collaudo)

```
Allow group <domain>/<group> to manage cluster-virtualnode-pools
  in compartment <cluster-compartment>

Allow group <domain>/<group> to read virtual-network-family
  in compartment <cluster-compartment>

Allow group <domain>/<group> to manage vnics
  in compartment <cluster-compartment>
```

Alternatively, using the aggregate resource type (broader but simpler):

```
Allow group <domain>/<group> to manage cluster-family
  in compartment <cluster-compartment>
```

> `cluster-family` includes `cluster-virtualnode-pools` — covers VNP create/update/delete operations.

---

### 2.2 Network Compartment (cmp-network) — ⚠️ UNDOCUMENTED — Confirmed via Oracle Support SR

**This policy is NOT present in the official Oracle documentation.**  
It was identified as mandatory through a formal Oracle Support Service Request.

**SR evidence — backend OCI IAM evaluation logs:**
```
requestedPermissions: [CLUSTER_VIRTUAL_NODE_POOL_CREATE]
authorizedPermissions: []
```

**SR Support confirmation:**
> *"yes, we do think they need this permission in all compartments that are involved  
> with creating the Cluster/VNP, that is indeed the reason why it was failing"*

**Why it happens:**  
`CLUSTER_VIRTUAL_NODE_POOL_CREATE` is a multi-resource IAM operation. OCI IAM evaluates  
authorization in **every compartment that contains resources referenced by the operation**.  
Since the VCN and subnets are in `cmp-network`, the IAM engine checks authorization there too.  
Because `cmp-collaudo` and `cmp-network` are sibling compartments with no shared parent below root,  
the `cluster-family` policy in `cmp-collaudo` does NOT propagate to `cmp-network`.

**Required policy — network compartment:**

```
Allow group <domain>/<group> to manage cluster-family
  in compartment <network-compartment>
```

> This is the key undocumented finding of this PoC — validated empirically and confirmed by Oracle Support.

---

## Policy Layer 3 — Cluster Principal Policies (OCI CNI Runtime)

> 📌 Source: [OCI VCN-Native Pod Networking CNI Plugin — Additional IAM Policy (Cross-Compartment)](https://docs.oracle.com/en-us/iaas/Content/ContEng/Concepts/contengpodnetworking_topic-OCI_CNI_plugin.htm)

These policies allow the OKE cluster principal to manage compute and network resources at Pod scheduling time — required for VCN-Native Pod Networking (OCI CNI).

### 3.1 Cluster Compartment

```
Allow any-user to manage instances in compartment <cluster-compartment>
  where all { request.principal.type = 'cluster',
               request.principal.id  = '<cluster-ocid>' }
```

### 3.2 Network Compartment

```
Allow any-user to use private-ips in compartment <network-compartment>
  where all { request.principal.type = 'cluster',
               request.principal.id  = '<cluster-ocid>' }

Allow any-user to use network-security-groups in compartment <network-compartment>
  where all { request.principal.type = 'cluster',
               request.principal.id  = '<cluster-ocid>' }
```

> The Oracle documentation also shows a broader `in tenancy` variant — the above compartment-scoped  
> version is preferred in production for least-privilege compliance.

---

## Policy Layer 4 — Workload Identity (Pod-to-OCI Authentication)

> 📌 Source: [Granting Workloads Access to OCI Resources](https://docs.oracle.com/en-us/iaas/Content/ContEng/Tasks/contenggrantingworkloadaccesstoresources.htm)  
> 📌 Source: [OKE Policies — Disaster Recovery Example (Virtual Node Pool)](https://docs.oracle.com/en-us/iaas/disaster-recovery/doc/oke-policies.html)

### Why Workload Identity is different on Virtual Nodes vs Managed Nodes

| | Managed Nodes | Virtual Nodes |
|---|---|---|
| Compute in | Customer tenancy | **Oracle OKE service tenancy** |
| Instance Principal | ✅ Works (VM in your tenancy) | ❌ Not applicable (VM not in your tenancy) |
| Dynamic Group on compute | ✅ Works | ❌ Container Instance not visible in your tenancy |
| Workload Identity | ✅ Works | ✅ **Only supported mechanism** |

> **Key insight:** On Virtual Nodes, you cannot use Instance Principal or Dynamic Group on  
> `computecontainerinstance` because those Container Instances are in the **Oracle tenancy**,  
> not yours. They are not visible as resources in your tenancy.  
> Workload Identity uses the **Kubernetes cluster + namespace + ServiceAccount** as the IAM principal —  
> this works regardless of where the underlying compute runs.

### Requirements

- Enhanced Cluster ✅ (your setup)
- OCI SDK in the application code (`OkeWorkloadIdentityConfigurationProvider`)
- `automountServiceAccountToken: true` in the Pod spec

### IAM Policy

```
Allow any-user to manage objects in compartment <target-compartment>
  where all {
    request.principal.type         = 'workload',
    request.principal.namespace    = '<k8s-namespace>',
    request.principal.service_account = '<serviceaccount-name>',
    request.principal.cluster_id   = '<cluster-ocid>'
  }
```

> Note: `request.principal.type = 'workload'` — NOT `'cluster'`, NOT `'virtualnode'`, NOT a Dynamic Group.

### ServiceAccount (Kubernetes)

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: sa-app
  namespace: poc-oke
imagePullSecrets:
- name: ocir-pullsecret
```

### Deployment spec requirement

```yaml
spec:
  serviceAccountName: sa-app
  automountServiceAccountToken: true
```

> ⚠️ **Bootstrap constraint:** Workload Identity can only be configured AFTER the cluster is  
> operational. The cluster becomes operational only after the `docker-registry` Secret bootstrap.  
> Sequence is mandatory: Secret bootstrap first → cluster operational → Workload Identity configured.

---

## What Does NOT Work on Virtual Nodes (Validated)

| Mechanism | Why it fails |
|-----------|-------------|
| `Allow any-user to read repos` with `cluster` principal | OCIR pull not supported via cluster principal |
| `Allow any-user to read repos` with `virtualnode` principal | OCIR pull not supported via virtualnode principal |
| Dynamic Group on `computecontainerinstance` | Container Instances are in Oracle tenancy — not visible in customer tenancy |
| Instance Principal | No VM in customer tenancy to act as principal |
| Workload Identity without bootstrap Secret | Cluster not yet operational — chicken-and-egg |

> **OCIR image pull on Virtual Nodes requires exclusively a `docker-registry` Kubernetes Secret.**

---

## Complete Policy Checklist

### Root Compartment (always required)
- [ ] `endorse` subnets — virtualnode principal → CreateContainerInstance
- [ ] `endorse` vnics — virtualnode principal → CreateContainerInstance
- [ ] `endorse` network-security-group — virtualnode principal → CreateContainerInstance

### Cluster Compartment (cmp-collaudo)
- [ ] `manage cluster-virtualnode-pools` (user group) — or `manage cluster-family`
- [ ] `read virtual-network-family` (user group)
- [ ] `manage vnics` (user group)
- [ ] `manage instances` — cluster principal (OCI CNI)

### Network Compartment (cmp-network) — cross-compartment sibling
- [ ] `manage cluster-family` (user group) — **undocumented, SR-confirmed**
- [ ] `use private-ips` — cluster principal (OCI CNI)
- [ ] `use network-security-groups` — cluster principal (OCI CNI)

### Workload Identity (post-bootstrap)
- [ ] IAM policy with `request.principal.type = 'workload'`
- [ ] `automountServiceAccountToken: true` in Pod spec
- [ ] OCI SDK with `OkeWorkloadIdentityConfigurationProvider` in application code

---

## References

| Document | URL |
|----------|-----|
| Required IAM Policies for Virtual Nodes | https://docs.oracle.com/en-us/iaas/Content/ContEng/Tasks/contengvirtualnodes-Required_IAM_Policies.htm |
| OCI CNI Cross-Compartment IAM | https://docs.oracle.com/en-us/iaas/Content/ContEng/Concepts/contengpodnetworking_topic-OCI_CNI_plugin.htm |
| Workload Identity | https://docs.oracle.com/en-us/iaas/Content/ContEng/Tasks/contenggrantingworkloadaccesstoresources.htm |
| OKE Virtual Nodes Best Practices (Oracle Blog) | https://blogs.oracle.com/cloud-infrastructure/getting-started-best-practices-oke-virtual-nodes |
| Comparing Virtual Nodes vs Managed Nodes | https://docs.oracle.com/en-us/iaas/Content/ContEng/Tasks/contengcomparingvirtualwithmanagednodes_topic.htm |
