# OKE Virtual Node Pool: What the Docs Don't Tell You

## A production deep dive into cross-compartment IAM, Guaranteed QoS, Pod Pending root causes, zero-downtime subnet migration, and application logging.

*By Alessandro Moccia — Oracle Solution Architect | Oracle ACE Program*

---

There is a specific kind of frustration that only cloud engineers know: you follow the official documentation to the letter, you configure everything exactly as described, and the platform returns a cryptic authorization error with no Work Request, no trace, and no explanation.

That is how this story begins.

This article covers OKE Virtual Node Pool in a fully private, cross-compartment enterprise architecture — no Internet Gateway, no NAT Gateway, Service Gateway only. The findings are organized into two types: empirically validated behaviors discovered during hands-on deployment and supported in several cases by Oracle Support Service Request evidence, and documented behaviors from Oracle official sources cited inline where referenced.

The full reference implementation — IAM policies, manifests, architecture diagrams, screenshots, and the complete technical whitepaper — is at:
**[github.com/alexmocciaoci/oke-virtual-nodepool-deep-dive](https://github.com/alexmocciaoci/oke-virtual-nodepool-deep-dive)**

---

## Before We Start: Virtual Nodes Are Fundamentally Different

Most engineers approach OKE Virtual Nodes as "managed nodes but lighter." That mental model will cause problems in production.

Virtual Nodes are architecturally closer to a serverless compute abstraction than to Kubernetes worker nodes. According to [Oracle's comparison documentation](https://docs.oracle.com/en-us/iaas/Content/ContEng/Tasks/contengcomparingvirtualwithmanagednodes_topic.htm), virtual nodes run in the Kubernetes Engine tenancy — not in the customer tenancy. This single fact drives every IAM, networking, and operational decision that follows.

| | Managed Nodes | Virtual Nodes |
|---|---|---|
| Worker VM in your tenancy | ✅ Yes | ❌ No — runs in Oracle OKE service tenancy |
| DaemonSet support | ✅ Yes | ❌ No |
| Instance Principal | ✅ Works | ❌ Not applicable |
| Pod networking | VCN-Native or Flannel | VCN-Native only |
| Resource model | Node-level | Pod-level — each Pod = 1 VNIC + 1 real IP |
| Pod-to-OCI auth | Instance Principal or Dynamic Group | Workload Identity only |
| IAM model | Standard | Cross-tenancy endorse required |

One hard prerequisite: Virtual Nodes are only supported on Enhanced Clusters, as documented in [Comparing Enhanced Clusters with Basic Clusters](https://docs.oracle.com/en-us/iaas/Content/ContEng/Tasks/contengcomparingenhancedwithbasicclusters_topic.htm).

---

## The Architecture: Private-Only, Cross-Compartment

The deployment was designed under enterprise constraints that reflect a common pattern in large OCI tenancies: strict compartmental separation between application and network resources.

```
OCI Tenancy
│
├── Cluster Compartment     ← OKE Enhanced Cluster
│
└── Network Compartment     ← VCN, subnets, NSGs
         ↑
         Sibling compartments — no shared parent below root
```

Three private regional subnets, no public egress:

| Subnet | Purpose | Notes |
|--------|---------|-------|
| /29 | K8s API Endpoint | Oracle-controlled sizing |
| /27 | Internal Load Balancer | Minimum recommended |
| /28 → /27 | Pod / Virtual Node IPs | Each Pod = 1 real IP — see Finding 4 |

In VCN-Native Pod Networking, every Pod consumes one real VCN IP address via a dedicated VNIC. As noted in the [Virtual Nodes networking documentation](https://docs.oracle.com/en-us/iaas/Content/ContEng/Concepts/contengnetworkconfig-virtualnodes.htm), the number of pods available in the cluster is limited by the number of IP addresses available in the subnet. A /28 provides ~11 usable IPs — adequate for limited testing, insufficient for production. Use at minimum /26 for real workloads, /24 for enterprise scale.

All outbound traffic — OCIR image pulls, OCI API calls — routes exclusively via Service Gateway.

---

## Finding 1: The IAM Behavior Nobody Documents

**This is the most important section. Bookmark it.**

### What happened

During Virtual Node Pool creation, the OCI Console returned an immediate synchronous failure:

```
Authorization failed or requested resource not found
```

No Work Request was generated. No retryable error. Silent, immediate failure.

The group had all policies from the [official Oracle IAM policy page for Virtual Nodes](https://docs.oracle.com/en-us/iaas/Content/ContEng/Tasks/contengvirtualnodes-Required_IAM_Policies.htm) applied correctly to the cluster compartment. Everything appeared correct.

### What Oracle Support found

After opening a formal Service Request, Oracle Support retrieved the following from backend IAM evaluation logs:

```
requestedPermissions: [CLUSTER_VIRTUAL_NODE_POOL_CREATE]
authorizedPermissions: []
```

Zero permissions authorized. Despite the policy being in place.

### Root cause: multi-scope IAM evaluation across disjoint compartments

`CLUSTER_VIRTUAL_NODE_POOL_CREATE` is not a single-resource operation. When a Virtual Node Pool references subnets in a different compartment, the OCI IAM engine evaluates authorization across every compartment that contains resources referenced by the operation.

The Cluster Compartment and the Network Compartment are siblings — their only common ancestor is the tenancy root. A policy in the Cluster Compartment has zero effect in the Network Compartment. The IAM engine evaluated the operation in the Network Compartment context, found no matching policy, and failed the entire operation.

Oracle Support's formal confirmation from the SR:

> *"Based on what we saw, yes, we do think they need this permission in all compartments that are involved with creating the Cluster/VNP, that is indeed the reason why it was failing, as we can see in the logs."*

This is by design, not a bug. And it is not documented in Oracle's official IAM policy page for Virtual Nodes.

### The fix

The SR log identifies exactly which permission was missing: `CLUSTER_VIRTUAL_NODE_POOL_CREATE`, which maps to the resource type `cluster-virtualnode-pools`. The fix is one statement in the Network Compartment:

```hcl
# Network Compartment — SR-confirmed, not in Oracle docs
allow group <domain>/<group> to manage cluster-virtualnode-pools
  in compartment <network-compartment>
```

Oracle Support suggested `manage cluster-family` during the SR — which works but is over-permissioned. The log identifies only `CLUSTER_VIRTUAL_NODE_POOL_CREATE` as missing. `cluster-virtualnode-pools` is the least-privilege fix consistent with the evidence.

---

## The Complete IAM Baseline: Four Layers

Each layer serves a distinct purpose and cannot substitute for another. Missing any one produces a different, equally opaque failure.

### Layer 1 — Endorse (Cross-Tenancy): the structural foundation

Container Instances backing your Pods do not run in your tenancy. They run in the Oracle OKE service tenancy (`ske`). When a Virtual Node schedules a Pod, OKE needs to attach a VNIC to your subnet and associate your NSG rules — a cross-tenancy operation that requires an explicit `endorse` policy in your tenancy's root compartment.

The following statements must be created exactly as shown. The OCIDs are Oracle-managed constants — identical for every OCI customer, must not be changed. Source: [Oracle Blog — Getting started and best practices with OKE virtual nodes](https://blogs.oracle.com/cloud-infrastructure/getting-started-best-practices-oke-virtual-nodes).

```hcl
define tenancy ske as ocid1.tenancy.oc1..aaaaaaaacrvwsphodcje6wfbc3xsixhzcan5zihki6bvc7xkwqds4tqhzbaq
define compartment ske_compartment as ocid1.compartment.oc1..aaaaaaaa2bou6r766wmrh5zt3vhu2rwdya7ahn4dfdtwzowb662cmtdc5fea

endorse any-user to associate compute-container-instances
  in compartment ske_compartment of tenancy ske
  with subnets in tenancy
  where ALL { request.principal.type='virtualnode',
              request.operation='CreateContainerInstance',
              request.principal.subnet=request.subnet.id }

endorse any-user to associate compute-container-instances
  in compartment ske_compartment of tenancy ske
  with vnics in tenancy
  where ALL { request.principal.type='virtualnode',
              request.operation='CreateContainerInstance',
              request.principal.subnet=request.subnet.id }

endorse any-user to associate compute-container-instances
  in compartment ske_compartment of tenancy ske
  with network-security-group in tenancy
  where ALL { request.principal.type='virtualnode',
              request.operation='CreateContainerInstance' }
```

**Understanding `request.principal.type='virtualnode'`**

This condition identifies the OKE Virtual Node as the acting IAM principal — not the cluster, not a workload, not a user. Together, the three `where` conditions form a precise authorization scope: only a Virtual Node (`type='virtualnode'`), only during Pod scheduling (`operation='CreateContainerInstance'`), only on the subnet declared in its own configuration (`request.principal.subnet=request.subnet.id`).

**Critical:** `request.principal.type='virtualnode'` is an infrastructure-level principal. Its sole purpose is to authorize the cross-tenancy VNIC attachment during Pod creation. It cannot be used to grant Pods access to OCI services such as Object Storage, Functions, or Vault. For Pod-to-OCI-service authentication, the only supported mechanism on Virtual Nodes is Workload Identity — see Layer 4.

Without these three endorse statements, no Pod can ever be scheduled. No workaround exists.

### Layer 2 — User Group Policies: cluster + network compartment

**Cluster Compartment** — from [Oracle docs: Required IAM Policies for Using Virtual Nodes](https://docs.oracle.com/en-us/iaas/Content/ContEng/Tasks/contengvirtualnodes-Required_IAM_Policies.htm):

```hcl
allow group <domain>/<group> to manage cluster-virtualnode-pools
  in compartment <cluster-compartment>

allow group <domain>/<group> to read virtual-network-family
  in compartment <cluster-compartment>

allow group <domain>/<group> to manage vnics
  in compartment <cluster-compartment>
```

If users also need to manage the OKE cluster itself, add:

```hcl
allow group <domain>/<group> to manage cluster-family
  in compartment <cluster-compartment>
```

**Network Compartment** — not in Oracle docs, identified via SR backend log:

```hcl
allow group <domain>/<group> to manage cluster-virtualnode-pools
  in compartment <network-compartment>
```

OCI IAM policies are compartment-scoped. A policy in the Cluster Compartment has zero effect in the Network Compartment. When cluster and VCN live in sibling compartments disjoint to root, explicit policies are required in both. There is no inheritance, no shortcut.

### Layer 3 — Cluster Principal (OCI CNI Runtime)

These statements allow the OKE cluster principal to manage VCN resources at Pod scheduling time. Required for VCN-Native Pod Networking in cross-compartment deployments — from [Oracle docs: OCI VCN-Native Pod Networking CNI plugin](https://docs.oracle.com/en-us/iaas/Content/ContEng/Concepts/contengpodnetworking_topic-OCI_CNI_plugin.htm):

```hcl
Allow any-user to manage instances in compartment <cluster-compartment>
  where all { request.principal.type = 'cluster',
               request.principal.id  = '<cluster-ocid>' }

Allow any-user to use private-ips in compartment <network-compartment>
  where all { request.principal.type = 'cluster',
               request.principal.id  = '<cluster-ocid>' }

Allow any-user to use network-security-groups in compartment <network-compartment>
  where all { request.principal.type = 'cluster',
               request.principal.id  = '<cluster-ocid>' }
```

### Layer 4 — Workload Identity (Pod-to-OCI Authentication)

On Managed Nodes, a Dynamic Group matching `resource.type='computecontainerinstance'` grants Pods access to OCI services. On Virtual Nodes, this does not work. The Container Instances backing Pods run in the Oracle `ske` tenancy — invisible to customer IAM. Dynamic Groups cannot reference resources in a tenancy they cannot see.

The only supported mechanism for Pod-to-OCI-service authentication on Virtual Nodes is Workload Identity — from [Oracle docs: Granting Workloads Access to OCI Resources](https://docs.oracle.com/en-us/iaas/Content/ContEng/Tasks/contenggrantingworkloadaccesstoresources.htm):

```hcl
Allow any-user to <verb> <resource> in compartment <target-compartment>
  where all {
    request.principal.type            = 'workload',
    request.principal.namespace       = '<k8s-namespace>',
    request.principal.service_account = '<serviceaccount-name>',
    request.principal.cluster_id      = '<cluster-ocid>'
  }
```

`request.principal.type='workload'` identifies the IAM principal as a Kubernetes workload — cluster + namespace + ServiceAccount. This identity is compute-independent: it does not depend on where the underlying Container Instance runs, which is precisely why it works on Virtual Nodes when Instance Principal does not.

Oracle docs constraints: works only with OCI SDKs (Go ≥v65.32.0, Java ≥v2.54.0, Python ≥v2.111.0, .NET ≥v87.3.0, Ruby ≥v2.19.0), cannot be combined with Dynamic Groups, create one provider instance per process.

Workload Identity activates after the Pod is running — it plays no role in Pod creation or scheduling. The Pod spec must declare the ServiceAccount explicitly:

```yaml
spec:
  serviceAccountName: <serviceaccount-name>
  automountServiceAccountToken: true
```

---

## Finding 2: The Mandatory Bootstrap Sequence

There is a chicken-and-egg problem at cluster initialization that surfaces only when you hit it.

For a Pod to be scheduled, the cluster must be operational. For the cluster to be operational, at least one Pod must pull its image successfully. Image pull from OCIR requires authentication. On Virtual Nodes, neither the cluster principal nor the `virtualnode` principal can pull from OCIR — confirmed during deployment. The only supported mechanism for OCIR image pull on Virtual Nodes is a `docker-registry` Kubernetes Secret.

This Secret must exist before any workload is deployed:

```bash
kubectl -n <namespace> create secret docker-registry ocir-pullsecret \
  --docker-server=<region>.ocir.io \
  --docker-username='<tenancy-namespace>/<identity-domain>/<username>' \
  --docker-password='<auth-token>'
```

Create a dedicated ServiceAccount referencing it. Never patch the `default` ServiceAccount:

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: sa-app
  namespace: <namespace>
imagePullSecrets:
- name: ocir-pullsecret
```

Workload Identity can only be configured once the cluster is operational and Pods are running. The sequence is not negotiable:

```
Secret bootstrap → Cluster operational → Workload Identity configured
```

### Cluster access: Identity Domains and kubeconfig

On Identity Domain tenancies, kubeconfig generation requires an additional flag:

```bash
oci ce cluster create-kubeconfig \
  --cluster-id <cluster-ocid> \
  --file $HOME/.kube/config \
  --region <region> \
  --token-version 2.0.0
```

Without `--token-version 2.0.0`, authentication fails silently. Both Cloud Shell attached to a private subnet and Git Bash via corporate VPN were used successfully during the deployment.

### Build vs runtime environments

Cloud Shell attached to a private VCN subnet cannot reach public image registries (docker.io, gcr.io) without a NAT Gateway or Internet Gateway. Use standard Cloud Shell with public egress for `podman build` and `podman push` to OCIR. The OKE cluster pulls from OCIR at runtime via Service Gateway — no internet path needed.

---

## Finding 3: Guaranteed QoS Is Not Optional

In Managed Node environments, `requests == limits` is a recommended practice. On Virtual Nodes, it is a structural requirement.

When a Pod is scheduled on a Virtual Node, OKE provisions a dedicated OCI Container Instance. The Container Instance shape — CPU and memory — must be determined at scheduling time. If `resources.requests` and `resources.limits` are absent, mismatched, or partially declared, shape resolution fails and the provisioning layer enters retry loops. This does not surface as a configuration error. It presents as non-deterministic, inexplicable Pending.

As noted in [Oracle's best practices for Virtual Nodes](https://blogs.oracle.com/cloud-infrastructure/getting-started-best-practices-oke-virtual-nodes), specifying CPU and memory requests in pod specs is crucial because this specification is the only way to ensure that pods get the CPU and memory required.

```yaml
# ❌ Non-deterministic Pending on Virtual Nodes
containers:
- name: app
  image: <image>
  # No resources declared

# ✅ Guaranteed QoS — deterministic shape resolution
containers:
- name: app
  image: <image>
  resources:
    requests:
      cpu: "250m"
      memory: "256Mi"
    limits:
      cpu: "250m"       # Must equal requests
      memory: "256Mi"
```

`requests == limits` produces Kubernetes QoS class **Guaranteed** — the only class that results in immediate, deterministic provisioning on Virtual Nodes. Burstable and BestEffort produce non-deterministic scheduling behavior.

This rule applies to every container in the Pod, including sidecars. A sidecar without matching resources drops the entire Pod from Guaranteed QoS.

---

## Finding 4: VNIC GC Creates Predictable Pending Windows

When a Pod is deleted, its VNIC does not immediately return to the subnet address pool. There is an OCI garbage collection window during which the IP remains reserved but unavailable. Rapid Pod deletion cycles — during force deletes or rolling updates on small subnets — produce a window where all IPs appear allocated but no Pods are running.

This is not a bug. It is expected VCN-Native networking behavior, with a recognizable signature:

```bash
kubectl describe pod <pod-name>
# Events: "0/2 nodes are available: 2 Insufficient cpu."
# On Virtual Nodes this message is misleading.
# The actual cause is often IP exhaustion in the pod subnet.
```

**For constrained subnets** — scale sequentially:
```bash
kubectl apply -f deployment.yaml   # replicas: 1 — wait for Running
kubectl scale deployment <name> --replicas=2
```

**After force delete** — wait 60–90 seconds before reprovisioning to allow VNIC GC to complete.

**For production** — size the pod subnet at minimum /26. For enterprise workloads, /24.

---

## Finding 5: Load Balancer Resets Are a Reconciliation Artifact

Intermittent `connection reset by peer` errors appeared when routing traffic through the internal OCI Load Balancer — not systematic, concentrated during Pod rollouts.

### Root cause: three-layer async reconciliation

```
Layer 1: Kubernetes EndpointSlice   ← sub-second update on Pod readiness change
Layer 2: OKE Cloud Controller       ← consumes EndpointSlice events, seconds latency
Layer 3: OCI LB Backend Set         ← updated by Cloud Controller, seconds to tens of seconds
```

**Window A** — new Pod not yet in backend set: Kubernetes declares the Pod Ready and routes traffic to it, but the OCI LB backend set has not yet registered it. Result: TCP RST.

**Window B** — terminating Pod still in backend set: Pod receives SIGTERM, EndpointSlice updates immediately, but OCI LB backend set takes seconds to propagate. In-flight connections arrive at a shutting-down Pod. Result: TCP RST or incomplete response.

Two solutions exist.

---

### Solution A: readinessProbe + preStop (validated during deployment)

Addresses both windows through timing coordination. Full implementation at [github.com/alexmocciaoci/oke-virtual-nodepool-deep-dive](https://github.com/alexmocciaoci/oke-virtual-nodepool-deep-dive).

```yaml
readinessProbe:
  httpGet:
    path: /health
    port: 8080
  initialDelaySeconds: 5
  periodSeconds: 5
  timeoutSeconds: 2
  failureThreshold: 3

lifecycle:
  preStop:
    exec:
      command: ["/bin/sh", "-c", "sleep 5"]

terminationGracePeriodSeconds: 15  # Must exceed preStop sleep
```

On Virtual Nodes, each Pod holds a real VCN IP. `externalTrafficPolicy: Local` eliminates the SNAT hop introduced by `Cluster` policy:

```yaml
spec:
  type: LoadBalancer
  externalTrafficPolicy: Local
```

**Constraint:** `externalTrafficPolicy: Local` and Pod Readiness Gates are mutually exclusive. Do not combine them on the same Service.

### Solution B: Pod Readiness Gate (Oracle official approach)

Documented in [Oracle docs: Pod Readiness Gates](https://docs.oracle.com/en-us/iaas/Content/ContEng/Tasks/contengusingloadbalancers-podreadinessgates.htm). OKE injects an additional readiness condition into each Pod, preventing it from being declared Ready until OCI confirms it is a healthy LB backend — closing Window A at the source.

```bash
kubectl label namespace <namespace> \
  loadbalancer.oci.oraclecloud.com/pod-readiness-gate-inject=enabled
```

The Pod remains `0/1 READY` until the OCI LB backend set has registered the Pod IP and the health check has passed.

| Scenario | Recommended |
|---|---|
| Rolling deployments, production traffic | Solution B — Pod Readiness Gate |
| Source IP preservation required | Solution A — readinessProbe + preStop |
| Both concerns | Separate Services |

---

## Finding 6: DaemonSet-Based Logging Is Structurally Incompatible. Here Is What Works.

Every standard Kubernetes logging guide prescribes a Fluent Bit or Fluentd DaemonSet on worker nodes. On Virtual Nodes, there are no worker nodes. DaemonSets do not schedule on Virtual Nodes — full stop.

This requires a deliberate architectural decision.

### The sidecar model: Fluent Bit per Pod

[Oracle docs — Viewing Application Logs on Virtual Nodes](https://docs.oracle.com/en-us/iaas/Content/ContEng/Tasks/contengviewingapplicationlogs-virtualnodes.htm) documents the correct approach: since virtual nodes do not support the Kubernetes DaemonSet, a lightweight logging agent runs as a sidecar for each Pod. The sidecar and the application container share a volume via `emptyDir`, with the application writing logs to a shared path and Fluent Bit reading and forwarding them.

The following example is adapted from Oracle docs, which uses OpenSearch as the remote log server destination. Replace the `[OUTPUT]` block with the persistent log destination appropriate to your architecture — OCI Logging, OpenSearch, or another supported Fluent Bit output target:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: fluentbit-config
  namespace: <namespace>
data:
  fluent-bit.conf: |
    [SERVICE]
        flush  1
        daemon Off

    [INPUT]
        Name  tail
        Path  /mnt/log/*

    [OUTPUT]
        # Example from Oracle docs uses OpenSearch as remote log server.
        # Replace with your target: OCI Logging, OpenSearch, or other Fluent Bit output.
        name       opensearch
        match      *
        host       <log-server-host>
        port       <port>
        tls        On
        HTTP_User  <username>
        HTTP_Passwd <password>
        Index      fluent-bit
        Suppress_Type_Name On
```

```yaml
# Pod spec — sidecar pattern from Oracle docs
volumes:
- name: logs
  emptyDir: {}
- name: fluentbit-config
  configMap:
    name: fluentbit-config

containers:
- name: app
  image: <region>.ocir.io/<ns>/<repo>:<tag>
  resources:
    requests:
      cpu: "250m"
      memory: "256Mi"
    limits:
      cpu: "250m"
      memory: "256Mi"
  volumeMounts:
  - name: logs
    mountPath: /mnt/log

- name: fluent-bit
  image: fluent/fluent-bit
  resources:
    requests:
      cpu: "100m"
      memory: "50Mi"
    limits:
      cpu: "100m"      # requests == limits — applies to every container including sidecars
      memory: "50Mi"
  volumeMounts:
  - name: logs
    mountPath: /mnt/log
  - name: fluentbit-config
    mountPath: /fluent-bit/etc
```

### Infrastructure events: OCI Audit

OCI Audit captures infrastructure-level events automatically — `ComputeContainerInstance.Create`, `VirtualNodePool.Create`, `LoadBalancer.Update`, IAM policy changes. No configuration required. Queryable via Audit API or OCI Console.

### CrashLoopBackOff forensics

On Virtual Nodes, previous container logs may be overwritten after multiple restarts. Capture them before the next restart overwrites them:

```bash
kubectl -n <namespace> logs <pod-name> --previous > crash-$(date +%s).log
```

---

## Finding 7: Live Subnet Migration — Zero Downtime Validated

During the deployment, the pod subnet needed to be expanded. The expectation was a maintenance window. The result was zero downtime.

### The mechanics

The `pod-configuration.subnetId` on a Virtual Node Pool is a forward-looking configuration. Updating it applies only to new Pods created after the change. Existing Pods retain their VNIC attachments and IPs from the original subnet — not terminated, not migrated, not interrupted.

```bash
oci ce virtual-node-pool update \
  --virtual-node-pool-id <vnp-ocid> \
  --pod-configuration "{\"subnetId\":\"<new-subnet-ocid>\",
                        \"nsgIds\":[\"<pod-nsg-ocid>\"],
                        \"shape\":\"Pod.Standard.E4.Flex\"}"
```

| Pod state | Subnet |
|---|---|
| Already running | Original subnet — unchanged |
| New Pods after update | New subnet — IPs from new CIDR |
| Both groups simultaneously | Both subnets active in the same VNP |

### Why the Load Balancer routes correctly

The Service selector is label-based — it selects Pods by label, not by subnet or IP range. When a new Pod on the new subnet passes its readiness probe, it enters the EndpointSlice, the Cloud Controller Manager reconciles the OCI LB backend set, and traffic begins flowing. Both IP sets — from old and new subnet — are simultaneously valid backends. The LB does not distinguish by subnet.

**Zero downtime. Validated.**

The practical implication: pod subnet expansion is a live operation. Start conservatively in early deployments and expand by updating `pod-configuration.subnetId` when approaching capacity. Ensure NSGs are configured on the new subnet before updating.

---

## Production Baseline: Complete Golden Configuration

### ServiceAccount

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: sa-app
  namespace: <namespace>
imagePullSecrets:
- name: ocir-pullsecret
```

### Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: <app-name>
  namespace: <namespace>
spec:
  replicas: 2
  revisionHistoryLimit: 3
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxUnavailable: 0
      maxSurge: 1
  selector:
    matchLabels:
      app: <app-label>
  template:
    metadata:
      labels:
        app: <app-label>
    spec:
      serviceAccountName: sa-app
      terminationGracePeriodSeconds: 15
      volumes:
      - name: logs
        emptyDir: {}
      - name: fluentbit-config
        configMap:
          name: fluentbit-config
      containers:
      - name: app
        image: <region>.ocir.io/<ns>/<repo>:<tag>
        imagePullPolicy: Always
        ports:
        - containerPort: 80
        resources:
          requests:
            cpu: "250m"
            memory: "256Mi"
          limits:
            cpu: "250m"
            memory: "256Mi"
        readinessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 5
          periodSeconds: 5
          timeoutSeconds: 2
          failureThreshold: 3
        livenessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 15
          periodSeconds: 10
          timeoutSeconds: 2
          failureThreshold: 3
        lifecycle:
          preStop:
            exec:
              command: ["/bin/sh", "-c", "sleep 5"]
        volumeMounts:
        - name: logs
          mountPath: /mnt/log
      - name: fluent-bit
        image: fluent/fluent-bit
        resources:
          requests:
            cpu: "100m"
            memory: "50Mi"
          limits:
            cpu: "100m"
            memory: "50Mi"
        volumeMounts:
        - name: logs
          mountPath: /mnt/log
        - name: fluentbit-config
          mountPath: /fluent-bit/etc
```

### Internal Load Balancer Service

```yaml
apiVersion: v1
kind: Service
metadata:
  name: <app-name>-lb
  namespace: <namespace>
  annotations:
    oci.oraclecloud.com/load-balancer-type: "lb"
    oci.oraclecloud.com/internal-load-balancer: "true"
spec:
  type: LoadBalancer
  externalTrafficPolicy: Local
  selector:
    app: <app-label>
  ports:
  - port: 80
    targetPort: 80
```

### Quick Start

```bash
# 1. Generate kubeconfig
oci ce cluster create-kubeconfig \
  --cluster-id <cluster-ocid> \
  --file $HOME/.kube/config \
  --region <region> \
  --token-version 2.0.0

# 2. Create namespace
kubectl create namespace <namespace>

# 3. OCIR pull secret — must exist before any workload
kubectl -n <namespace> create secret docker-registry ocir-pullsecret \
  --docker-server=<region>.ocir.io \
  --docker-username='<tenancy-namespace>/<identity-domain>/<username>' \
  --docker-password='<auth-token>'

# 4. Apply manifests
kubectl apply -f 00-serviceaccount.yaml
kubectl apply -f 10-deployment.yaml
kubectl apply -f 20-service-lb.yaml

# 5. Verify
kubectl -n <namespace> get pods -o wide
kubectl -n <namespace> get svc
kubectl -n <namespace> rollout status deployment/<app-name>
```

---

## Production Checklist

**IAM**
- [ ] Endorse policies (3 statements) in root compartment — `virtualnode` principal, subnets, vnics, NSG
- [ ] `manage cluster-virtualnode-pools` in cluster compartment (user group)
- [ ] `manage cluster-virtualnode-pools` in network compartment (user group) — SR-confirmed, not in Oracle docs
- [ ] `read virtual-network-family` and `manage vnics` in cluster compartment
- [ ] Cluster principal policies for OCI CNI (instances, private-ips, NSGs)

**Bootstrap**
- [ ] `docker-registry` Secret created before any workload
- [ ] Dedicated ServiceAccount with `imagePullSecrets`
- [ ] `--token-version 2.0.0` in kubeconfig generation (Identity Domain tenancies)

**Workload**
- [ ] `requests == limits` on all containers including sidecars (Guaranteed QoS)
- [ ] Dedicated ServiceAccount in Pod spec with `automountServiceAccountToken: true`
- [ ] Workload Identity (`request.principal.type='workload'`) for Pod-to-OCI auth

**Networking**
- [ ] Pod subnet minimum /26 for production
- [ ] OCIR reachable via Service Gateway
- [ ] NSGs configured on new subnet before any live subnet migration

**Reliability**
- [ ] Pod Readiness Gate OR `externalTrafficPolicy: Local` — choose one, not both
- [ ] `maxUnavailable: 0` in rolling update strategy
- [ ] `preStop` + `terminationGracePeriodSeconds` > sleep value (if Solution A)

**Logging**
- [ ] Fluent Bit sidecar in every Pod spec
- [ ] Shared `emptyDir` volume between app and sidecar
- [ ] Fluent Bit ConfigMap configured for persistent log destination
- [ ] OCI Audit active for infrastructure events (automatic)

**Cluster**
- [ ] Enhanced Cluster — required for Virtual Nodes

---

## Conclusions

OKE Virtual Node Pool delivers genuine serverless Kubernetes: no worker node management, no data plane capacity planning, pod-level billing, elastic scaling across the region. In the right architecture, it represents a meaningful reduction in operational surface area.

The path to production, however, requires navigating behaviors that official documentation either understates or omits entirely.

The cross-compartment IAM multi-scope evaluation is the most consequential gap. In enterprise tenancies with network/application compartment separation — which describes the majority of serious OCI deployments — this stops deployment before a single Pod is scheduled. The fix is one policy statement, but finding it required formal Oracle Support engagement and backend log analysis. The IAM engine logged exactly which permission was evaluated and found absent. That is the kind of precision that turns days of debugging into minutes.

The `virtualnode` IAM principal deserves particular attention. It is the most architecturally misunderstood element in the Virtual Node IAM model. It exists for one purpose: authorizing the cross-tenancy VNIC attachment at Pod scheduling time. It cannot grant application code access to OCI services. Understanding this distinction prevents the most common IAM design error on Virtual Nodes — and points directly to Workload Identity as the correct model for Pod-to-OCI authentication.

The remaining findings — Guaranteed QoS, VNIC GC windows, LB reconciliation, DaemonSet incompatibility, live subnet migration — each have a well-defined root cause and a deterministic solution. Once the underlying mechanics are understood, none of them are surprising. The behavior is predictable. The remediation is clean.

The complete reference implementation is at:
**[github.com/alexmocciaoci/oke-virtual-nodepool-deep-dive](https://github.com/alexmocciaoci/oke-virtual-nodepool-deep-dive)**

---

*Alessandro Moccia — Oracle Solution Architect, OCI Specialist at Accenture/Noovle. Oracle ACE Program candidate.*

*Tags: Oracle Cloud, Kubernetes, OKE, Cloud Architecture, IAM, Cloud Native, DevOps*
