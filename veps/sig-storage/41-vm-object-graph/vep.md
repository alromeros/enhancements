# VEP #41: Object Graph API for VM Dependencies

## VEP Status Metadata

### Target releases

- This VEP targets alpha for version: v1.6
- This VEP targets beta for version: v1.11 (proposed — confirm with SIG Storage)
- This VEP targets GA for version:

### Release Signoff Checklist

Items marked with (R) are required *prior to targeting to a milestone / release*.

- [x] (R) Enhancement issue created, which links to VEP dir in [kubevirt/enhancements] : https://github.com/kubevirt/enhancements/issues/41
- [x] (R) Alpha target version is explicitly mentioned and approved
- [ ] (R) Beta target version is explicitly mentioned and approved
- [ ] (R) GA target version is explicitly mentioned and approved

## **Overview**

This is a proposal to include an `Object Graph` API in KubeVirt to represent VM and VMI
dependencies and their relationships. The API is the **authoritative answer to one question:
for a given VM/VMI, what is the exact set of objects it depends on to be defined and to run.**
It is a general-purpose dependency inventory, not tied to any single workflow: backup/restore,
cross-cluster migration, GitOps/config management, disaster recovery, troubleshooting, and
KubeVirt-internal tooling all need the same authoritative set and are all first-class
consumers.

The alpha implementation shipped in v1.6 behind the `ObjectGraph` feature gate. This
revision of the VEP defines the **beta** design. Because the wire format and behavior are
much harder to change after beta, and because consumer feedback during alpha surfaced
concrete gaps, the beta revision **changes the wire format and options transport**. These
are breaking changes and, by design, all of them land while the feature is still alpha.

## **Motivation**

As new features continue to be added to KubeVirt, the graph of objects related to VMs
(DataVolumes, PersistentVolumeClaims, InstanceTypes, Preferences, Secrets, ConfigMaps,
NetworkAttachmentDefinitions, backend-storage PVCs, etc.) continues to expand. Any tool or
user that needs to reason about a VM as a whole — rather than a single object — has to
reconstruct this set by hand today, which is error-prone and drifts as KubeVirt evolves.
Concrete consumers:

- **Backup/restore** — the set that must be captured, and later recreated, for a VM.
- **Cross-cluster migration / DR** — the set that must be replicated to a target cluster,
  distinguishing state that must be copied from resources the target regenerates.
- **GitOps / configuration tooling** — the objects that constitute a VM's desired state and
  should be tracked, exported, or reconciled, distinguishing user-authored config from
  controller-generated artifacts.
- **Users / cluster admins** — impact analysis and troubleshooting ("what does this VM
  actually depend on; what breaks if I delete this ConfigMap?").
- **KubeVirt developers** — a single, maintained place where a VM's dependency relationships
  live, updated alongside feature work.

Backup/restore vendors are the most demanding consumer and built against the alpha API first;
the gaps they surfaced generalize to every consumer above and must be addressed before the
surface is frozen at beta:

- **Fan-out cost.** One `GET` per VM makes namespace-wide or multi-VM operations (backup,
  migration, inventory) issue N calls, adding latency and apiserver load.
- **Untyped outcomes.** The feature-gate-off response is a `400` whose only signal is a
  human-readable message, forcing consumers to substring-match. `NotFound` vs. transient
  `Unavailable` vs. `FeatureGateDisabled` cannot be distinguished cleanly.
- **Coarse filtering.** Options travel in a `GET` request body; labels are only the coarse
  `kubevirt.io/dependency-type`, with no way to tell durable state from controller-generated
  runtime artifacts; invalid selectors fail *open*; filtering happens after the full graph is
  built.
- **`Kind` vs. plural resource.** The API exposes `Kind` (`PersistentVolumeClaim`) while
  dynamic clients and tooling such as Velero key off plurals (`persistentvolumeclaims`).
- **Tree ergonomics.** The hierarchical tree duplicates shared referents and nests
  inconsistently, pushing dedup/flatten work onto every consumer.

## **Goals**

- Provide a stable, complete, predictable dependency set for VM and VMI, usable as the source
  of truth by any consumer (backup, migration, GitOps, DR, troubleshooting).
- Make node identity unambiguous and dynamic-client-ready (group/version/resource/kind).
- Classify nodes (by type and by lifecycle) so **any** consumer can derive the subset it
  needs — server-side, without walking the whole graph.
- Support namespace / multi-VM fan-out in a single request with partial-success semantics.
- Allow computing a graph from a supplied manifest, so consumers can enumerate dependencies
  for a VM that does not (yet) exist in the cluster.
- Return typed, machine-readable error reasons.
- Keep the code path reusable as a library (pure `spec+status → graph`, with isolated live
  lookups) rather than locked behind the REST handler.

## **Non-Goals**

- Reimplementation of existing VM/VMI specs.
- Building a generic Kubernetes-wide graph system.
- Prescribing any single consumer's workflow (backup, migration, GitOps): the API returns the
  classified dependency set; consumers decide how to use it.
- Guaranteeing referent existence by default (existence resolution is opt-in).

## **User Stories**

1. As a KubeVirt user or admin, I want to retrieve the full, authoritative set of a VM/VMI's
   dependencies, so I can understand and troubleshoot what it needs to run.
2. As a backup partner, I want to identify a VM's related objects so I can comprehensively
   back up and restore everything a VM needs.
3. As a consumer operating on many VMs (namespace backup, bulk migration, inventory), I want
   graphs for all selected VMs in one call, with per-VM errors, so fan-out is cheap and
   resilient.
4. As any consumer, I want to filter the set by classification — e.g. durable state vs.
   controller-generated runtime artifacts, or by resource type — **without post-processing the
   whole graph**.
5. As a platform operator, I want to replicate a VM across clusters (migration/DR),
   identifying which dependencies must be copied and which the target regenerates.
6. As a GitOps/config tool, I want the objects that make up a VM's desired state, separating
   user-authored config from controller-generated artifacts, so I can track/export/reconcile
   the right set.
7. As a KubeVirt developer, I want a single place to keep dependency logic updated when I
   introduce code that changes a VM's relationship to its dependent objects.
8. As a consumer, I want to compute a VM's dependency graph from a supplied manifest — a
   backed-up VM at restore time, or a not-yet-created VM for pre-apply/migration planning —
   even though the object does not exist in the cluster.

## **Repos**

- [KubeVirt](https://github.com/kubevirt/kubevirt)
- [KubeVirt Velero Plugin](https://github.com/kubevirt/kubevirt-velero-plugin)

## **Design**

### Summary of changes from alpha

| Area | Alpha (v1.6) | Beta |
|---|---|---|
| Graph shape | Hierarchical tree (`children`), duplicated referents | Flat, **deduped** node list + `parentRef` |
| Node identity | `TypedObjectReference` (`apiGroup`, `Kind`, name, ns) | `group`/`version`/`resource` (plural)/`kind`/`ns`/`name`/`uid` |
| Labels | `kubevirt.io/dependency-type` (one axis) | `objectgraph.kubevirt.io/type` + `objectgraph.kubevirt.io/lifecycle` |
| Options transport | Body on `GET` | Query params on `GET`; body on batch `POST` |
| Fan-out | 1 GET per object | Batch endpoint, partial success |
| Errors | `400` with message only | Typed reasons (`FeatureGateDisabled`/`NotFound`/`Unavailable`/`Invalid`) |
| Bad selector | Fails open (includes all) | Fails closed (`Invalid`) |
| Node completeness | missing sysprep, ephemeral | adds sysprep, ephemeral; memoryDump labeled `runtime`; containerDisk as external ref |

### Graph representation

We return a **flat, deduplicated list** of dependency nodes with a deterministic order,
reversing the alpha hierarchical decision (see *Alternatives Considered*). The single
meaningful relationship — "this object is materialized/owned by that one" (e.g. a PVC by
its DataVolume) — is preserved via an optional `parentRef` rather than nesting. This gives
every consumer exactly one node per real object and removes the flatten/dedup burden.

### API Schema

```go
// ObjectGraphObjectID uniquely identifies a referenced object and is dynamic-client-ready.
type ObjectGraphObjectID struct {
    Group     string `json:"group"`               // "" for core
    Version   string `json:"version,omitempty"`   // best-effort
    Resource  string `json:"resource"`            // PLURAL, e.g. "persistentvolumeclaims"
    Kind      string `json:"kind"`                // e.g. "PersistentVolumeClaim"
    Namespace string `json:"namespace,omitempty"` // empty => cluster-scoped
    Name      string `json:"name"`
    UID       string `json:"uid,omitempty"`       // set only when resolved from a live object
}

// ObjectGraphNode represents an individual dependency.
//
// +k8s:deepcopy-gen:interfaces=k8s.io/apimachinery/pkg/runtime.Object
type ObjectGraphNode struct {
    ObjectGraphObjectID `json:",inline"`
    // Labels carries the classification axes. See "Labels".
    Labels map[string]string `json:"labels,omitempty"`
    // Optional indicates the referent may legitimately not exist. Pointer so consumers can
    // distinguish "explicitly required" (false) from "unset" for reporting.
    Optional *bool `json:"optional,omitempty"`
    // ParentRef points at the node this one is materialized/owned by, if any.
    ParentRef *ObjectGraphObjectID `json:"parentRef,omitempty"`
}

// ObjectGraph is the graph for a single VM/VMI.
//
// +k8s:deepcopy-gen:interfaces=k8s.io/apimachinery/pkg/runtime.Object
type ObjectGraph struct {
    metav1.TypeMeta `json:",inline"`
    Root  ObjectGraphObjectID `json:"root"`  // which node is the VM/VMI itself
    Nodes []ObjectGraphNode   `json:"nodes"` // deduped by identity, deterministically ordered
}

// ObjectGraphOptions holds filtering/behavior options (query params on GET; body on batch).
type ObjectGraphOptions struct {
    // IncludeOptional includes nodes whose referent may legitimately be absent. Default true.
    IncludeOptional *bool `json:"includeOptional,omitempty"`
    // LabelSelector filters nodes by their labels. Invalid selectors fail closed (Invalid).
    LabelSelector *metav1.LabelSelector `json:"labelSelector,omitempty"`
    // ResolveExistence, when true, sets ObjectGraphObjectID.UID / presence by checking the
    // cluster. Opt-in because it adds live reads and latency. Default false.
    ResolveExistence *bool `json:"resolveExistence,omitempty"`
}

// --- Batch types ---

// ObjectGraphList is returned by the batch endpoint.
//
// +k8s:deepcopy-gen:interfaces=k8s.io/apimachinery/pkg/runtime.Object
type ObjectGraphList struct {
    metav1.TypeMeta `json:",inline"`
    Items []ObjectGraphResult `json:"items"`
}

// ObjectGraphResult is a per-object outcome (partial success).
type ObjectGraphResult struct {
    Ref   ObjectGraphObjectID `json:"ref"`
    Graph *ObjectGraph        `json:"graph,omitempty"`
    Error *ObjectGraphError   `json:"error,omitempty"`
}

// ObjectGraphError is a machine-readable per-item failure.
type ObjectGraphError struct {
    Reason  string `json:"reason"` // FeatureGateDisabled|NotFound|Unavailable|Invalid
    Message string `json:"message"`
}

// ObjectGraphBatchRequest is the body of the batch endpoint. Callers may supply an explicit
// list of names, a VM/VMI label selector, and/or inline manifests (union). Names/Selector
// resolve objects that exist in the cluster; Objects are computed from the supplied manifest
// and need NOT exist (restore / DR).
type ObjectGraphBatchRequest struct {
    Kind          string                `json:"kind"`                    // VirtualMachine | VirtualMachineInstance
    Names         []string              `json:"names,omitempty"`         // must exist in cluster
    Selector      *metav1.LabelSelector `json:"selector,omitempty"`      // must exist in cluster
    // Objects carries inline VM/VMI manifests (decoded per Kind). These need NOT exist in the
    // cluster; the graph is computed purely from the supplied spec+status. Enables restore/DR
    // when the VM has already been deleted.
    Objects       []runtime.RawExtension `json:"objects,omitempty"`
    Options       *ObjectGraphOptions   `json:"options,omitempty"`
}
```

### Node identity and the Kind/plural contract

Every node carries both `kind` (`PersistentVolumeClaim`) and `resource` (the plural
`persistentvolumeclaims`), plus `group` and best-effort `version`, so consumers can build a
`GroupVersionResource` and use the dynamic client with no guessing. `uid` is populated only
when the node was resolved from a live object (always for launcher pod / backend-storage
PVC; for spec-derived nodes only when `ResolveExistence` is set), letting restore detect a
recreated referent. The plural/Kind contract is documented and stable for beta.

### Labels

Two orthogonal, documented, stable-enum axes are set on every node:

| Label key | Values | Meaning |
|---|---|---|
| `objectgraph.kubevirt.io/type` | `storage`, `network`, `compute`, `config` | categorization by domain |
| `objectgraph.kubevirt.io/lifecycle` | `persistent`, `runtime` | durable state vs. controller-generated |

The `lifecycle` axis is deliberately consumer-neutral — it describes the *object*, and each
consumer maps it to its own workflow:

- **`persistent`** — durable state/config the user or an operation must own: it is not
  regenerated by KubeVirt (DataVolume, PVC, backend-storage PVC, ConfigMap, Secret,
  ServiceAccount, ControllerRevision, NAD, instancetype/preference, sysprep source, the
  VM/VMI object itself).
- **`runtime`** — reconstructed by KubeVirt controllers from the persistent set (launcher
  **pod**, the child **VMI** node in a VM graph, memoryDump).

The two axes plus `optional` let each consumer derive the subset it needs, server-side:

| Consumer intent | Filter |
|---|---|
| Full dependency inventory | all nodes |
| Objects to replicate / recreate (backup capture, migration copy, GitOps desired state) | `lifecycle=persistent` |
| Controller-generated artifacts to skip (do not copy/recreate) | `lifecycle=runtime` |
| Only storage / only network dependencies | `type=storage` / `type=network` |
| Hard requirements (must exist) | `optional != true` |

Labels are an **open map**: consumers MUST ignore unknown keys/values, so additional axes
can be introduced post-beta without breaking anyone.

### Optional semantics

`Optional` means only *"the referent may legitimately not exist"* (e.g. a ConfigMap volume
marked optional in the spec, or a cluster-scoped instancetype the VM points at). It does
**not** mean "not required for the VM to run" — that distinction is expressed by
`lifecycle`. NADs are therefore **not** marked optional (a VM needs its NAD), and volume-level
`Optional` on ConfigMap/Secret volumes is honored.

### API Endpoints

Single object (options as query parameters — no body on GET):

```
GET  /apis/subresources.kubevirt.io/v1/namespaces/{ns}/virtualmachines/{name}/objectgraph
        ?includeOptional=true&labelSelector=...&resolveExistence=false
GET  /apis/subresources.kubevirt.io/v1/namespaces/{ns}/virtualmachineinstances/{name}/objectgraph
        ?includeOptional=true&labelSelector=...&resolveExistence=false
```

Batch (namespace / multi-VM fan-out, `POST` with body, partial success):

```
POST /apis/subresources.kubevirt.io/v1/namespaces/{ns}/objectgraphs
     body: ObjectGraphBatchRequest  ->  ObjectGraphList
```

### Filtering behavior

- **Fail closed:** an invalid `labelSelector` returns `Invalid`, never "include everything".
- **Work-reducing:** when the effective filter excludes `lifecycle=runtime` (or
  `type=compute`), the server skips the launcher-pod lookup entirely, so filtering reduces
  server work, not just response size.

### Error model

Stable, machine-readable reasons on `metav1.Status` (single) and in each `ObjectGraphResult.Error` (batch):

| Situation | HTTP | Reason |
|---|---|---|
| Feature gate off | 400 | `FeatureGateDisabled` |
| VM/VMI not found | 404 | `NotFound` |
| Transient live-read failure | 503 | `Unavailable` |
| Bad selector / options | 400 | `Invalid` |

A VM that needs backend storage but whose backend PVC has not yet been created returns a
**partial graph** (the backend-PVC node omitted or marked `optional`), not an error — a
read must not fail on provisioning-in-progress state.

### Computing a graph from a supplied object (need not exist)

Several consumers need a graph for an object that is not present live in the cluster:

- **Restore** — the VM was deleted after backup and must be re-derived at restore time.
- **GitOps / pre-apply** — enumerate what a VM *will* need before it is created.
- **Migration / DR planning** — enumerate dependencies from a manifest to stage on a target
  cluster.

Two mechanisms cover this without weakening the live subresource:

1. **Graph as a point-in-time artifact (recommended where available).** The graph is a
   snapshot of dependencies at the moment it was taken. A consumer that already has one (e.g.
   backup captured it and stored it) SHOULD reuse it directly, with no cluster read of the
   (possibly absent) VM. This is the most accurate path: it reflects the exact live-resolved
   dependencies (uids, the backend-storage PVC a live lookup found) as they were, and is not
   subject to graph-logic drift between KubeVirt versions.

2. **Compute-from-manifest.** The batch endpoint accepts inline VM/VMI manifests via
   `ObjectGraphBatchRequest.Objects`. The server computes the graph as a pure function of the
   supplied `spec + status`, so the object need not exist in the cluster. The per-object
   subresource is unchanged and still requires the object to exist — this capability lives
   only on the collection `POST`, so nothing is dropped.

   Because the object is absent, live-only nodes cannot be resolved: the launcher **pod** is
   `runtime` (regenerated by controllers) and is omitted; the **VM** backend-storage PVC is
   currently a live lookup and will be missing unless present in the supplied status. To close
   that last gap we will **surface the backend-storage PVC name in the VM status** (addressing
   the existing TODO in `pkg/storage/utils/volumes.go`), making the VM graph fully derivable
   from `spec + status`. Combined with `resolveExistence`, consumers can also validate which
   referents currently exist (e.g. drift detection for GitOps, pre-restore checks).

The pure graph library MUST be defensive against partially-populated or arbitrary supplied
specs (no panics), since `Objects` is client-provided.

### Included Resources

#### KubeVirt-native / Kubernetes resources

- **Instance type / preference `ControllerRevision`** (`status.instancetypeRef` /
  `status.preferenceRef`, falling back to the spec `revisionName`).
- **Instancetype / Preference** (namespaced and cluster-scoped) — `optional`.
- **VirtualMachineInstance** (VM graphs) — `runtime`.
  - **virt-launcher Pod** (matched by label + ownerRef/annotation) — `runtime`.
- **Volumes:**
  - **DataVolume** (+ its materialized PVC via `parentRef`)
  - **PersistentVolumeClaim**
  - **ConfigMap** (honoring volume-level `optional`)
  - **Secret** (honoring volume-level `optional`)
  - **ServiceAccount**
  - **cloudInitNoCloud / cloudInitConfigDrive** user/network-data Secrets
  - **Sysprep** (Secret or ConfigMap) — *new at beta*
  - **Ephemeral** backing PVC — *new at beta*
  - **MemoryDump** PVC — labeled `runtime`
  - **ContainerDisk** — represented as an external reference (image, not a k8s object)
- **AccessCredentials:** SSH and user-password Secrets.
- **Backend-storage PVC** (persistent TPM/EFI/NVRAM) — identified from VMI status, or by the
  persistent-state PVC label for VMs.

A `default` branch logs any unrecognized volume source or access-credential source so future
additions cannot silently drop out of the contract.

#### External Resources

KubeVirt does not own all resources involved in VM operations (e.g. Multus
NetworkAttachmentDefinitions, IPAMClaims). These are included via a **generic mechanism**
using the **dynamic client** on unstructured objects, so KubeVirt does not import downstream
CRDs. To qualify for inclusion an external resource must be:

1. **Relevant** to a KubeVirt-supported VM/VMI operation.
2. **Referenced** in the VM/VMI spec (e.g. NADs in `networks[*]`) **or** handled by a
   KubeVirt-owned project (e.g. IPAMClaims from `kubevirt/ipam-extensions`).
3. **Discoverable** — reliably associable with a VM/VMI (spec/status reference or a
   KubeVirt-defined label).
4. **Dependency-free** — accessed via the dynamic client as unstructured, no external imports.
5. **Limited scope** — only metadata is needed; inclusion logic stays isolated to the graph
   package.

`containerDisk` images are represented under this external-reference model (no group/resource,
distinct kind) so consumers can see the image reference the VM depends on.

### Example Output (flat)

```json
{
  "root": {
    "group": "kubevirt.io", "version": "v1", "resource": "virtualmachines",
    "kind": "VirtualMachine", "namespace": "default", "name": "vm-cirros"
  },
  "nodes": [
    {
      "group": "kubevirt.io", "version": "v1", "resource": "virtualmachines",
      "kind": "VirtualMachine", "namespace": "default", "name": "vm-cirros",
      "labels": { "objectgraph.kubevirt.io/type": "compute",
                  "objectgraph.kubevirt.io/lifecycle": "persistent" }
    },
    {
      "group": "cdi.kubevirt.io", "version": "v1beta1", "resource": "datavolumes",
      "kind": "DataVolume", "namespace": "default", "name": "cirros-dv",
      "labels": { "objectgraph.kubevirt.io/type": "storage",
                  "objectgraph.kubevirt.io/lifecycle": "persistent" }
    },
    {
      "group": "", "version": "v1", "resource": "persistentvolumeclaims",
      "kind": "PersistentVolumeClaim", "namespace": "default", "name": "cirros-dv",
      "labels": { "objectgraph.kubevirt.io/type": "storage",
                  "objectgraph.kubevirt.io/lifecycle": "persistent" },
      "parentRef": {
        "group": "cdi.kubevirt.io", "version": "v1beta1", "resource": "datavolumes",
        "kind": "DataVolume", "namespace": "default", "name": "cirros-dv"
      }
    },
    {
      "group": "", "version": "v1", "resource": "pods",
      "kind": "Pod", "namespace": "default", "name": "virt-launcher-vm-cirros-frn9h",
      "uid": "3f2b...",
      "labels": { "objectgraph.kubevirt.io/type": "compute",
                  "objectgraph.kubevirt.io/lifecycle": "runtime" }
    }
  ]
}
```

## Alternatives Considered

### Hierarchical tree (the alpha design)

The alpha implementation returned a tree (`children`). We reverse this for beta because, in
practice for every consumer:

- Shared referents were duplicated and nesting was inconsistent (a DataVolume's PVC was
  nested while a plain PVC volume was top-level; the launcher pod was nested under the VMI in
  a VM graph but top-level in a VMI graph), forcing every consumer to flatten and dedup.
- Consumers iterate over the dependency **set**; where ordering matters (e.g. recreate order
  at restore), `lifecycle` labels and `parentRef` serve it better than tree depth.
- A flat, deduped list is simpler to filter server-side and cheaper to reason about as the
  dependency set grows.

`parentRef` preserves the one relationship that matters (materialization/ownership) without
reintroducing a tree.

### Naming

`ObjectGraph` is retained for continuity with the alpha subresource despite the flat output.

## **Scalability**

Each single-object graph is scoped to one VM/VMI and generated on demand by virt-api. The
batch endpoint amortizes fan-out for any namespace/multi-VM operation (backup, migration,
inventory) into one request, cutting apiserver load and latency. Filters that exclude runtime
nodes let the server skip live pod lookups, reducing work under filtered queries.

## **Update/Rollback Compatibility**

- The beta wire-format and options-transport changes are **breaking** and land while the
  feature is still alpha (gated), so no released stable surface changes.
- Non-intrusive addition via subresources; safe to enable/disable per version.
- The one spec/status change is additive: surfacing the backend-storage PVC name in VM
  status (a new status field). No existing field changes; safe to roll back (the field is
  simply re-derived on the next reconcile).
- No changes to existing VM/VMI specs.

### API Stability Guarantees (committed at beta)

Once beta ships, the following are contractual and will not change without a new
major-version migration:

- **Node identity fields** (`group`, `version`, `resource`, `kind`, `namespace`, `name`,
  `uid`) and the plural-`resource`/`kind` contract.
- **Label keys** `objectgraph.kubevirt.io/type` and `objectgraph.kubevirt.io/lifecycle`, and
  their documented value enums. New values or new label keys may be *added*; consumers MUST
  ignore unknown keys/values (this is what keeps additions non-breaking).
- **`optional` semantics** ("referent may legitimately not exist"), distinct from
  `lifecycle`.
- **Error reason strings** (`FeatureGateDisabled`, `NotFound`, `Unavailable`, `Invalid`).
- **The consumer-derivable subsets** (full inventory, `lifecycle`-based replicate/skip splits,
  `type`-based storage/network splits, `optional`-based hard-requirement set) and the filters
  that produce them.

Node *coverage* (which dependency types appear) may expand across versions; a
required-vs-optional matrix per CNV version is published so consumers can reason about
differences.

## **Security Considerations**

- **Authorization:** the per-object subresources are guarded by the existing
  `virtualmachines/objectgraph` and `virtualmachineinstances/objectgraph` RBAC. The batch
  collection endpoint requires an analogous verb on the `objectgraphs` collection resource;
  callers must be authorized in the target namespace.
- **Compute-from-manifest:** the batch endpoint accepts client-supplied VM/VMI manifests and
  runs dependency logic over them (precedent: the `expand-spec` subresource, which likewise
  processes a supplied VM). This is read-only and persists nothing, but the graph library
  MUST be hardened against partial/malformed input (no panics, bounded work). It does not
  read arbitrary cluster objects on the caller's behalf beyond the same live lookups the
  normal path performs, and only when the referenced object exists.
- **Information exposure:** the graph returns only object *references* (identity + labels),
  never object contents (no Secret/ConfigMap data), so it does not widen data exposure beyond
  what listing those objects already would.

## **Graduation Criteria**

### Alpha → Beta

- Wire format, labels, `optional` semantics, transport, batch endpoint, typed errors, and
  node completeness implemented behind the gate (Implementation Phases 1–5).
- The VM graph is fully derivable from `spec + status` (backend-storage PVC name surfaced in
  status), so compute-from-manifest is complete for both VM and VMI.
- Test coverage closes the alpha gaps: VM/VMI parity, dedup, deterministic ordering,
  cross-namespace refs, non-existent referents, persistent EFI *and* TPM backend PVC,
  feature-gate-disabled, and the partial-graph path.
- User-facing documentation published (contract page + required-vs-optional matrix).
- At least one partner (e.g. Velero plugin / Cohesity) validates the beta API against a real
  backup+restore flow, including restore of a deleted VM via compute-from-manifest.

### Beta → GA

- API stable across at least one release with no breaking changes required by feedback.
- External-resource inclusion mechanism finalized (NADs; explicit decision on IPAMClaims).
- Velero plugin (and/or other partners) shipping on the API.
- Scale validation of the batch endpoint for namespace-sized fan-out.

## **Functional Testing Approach**

- Unit tests on a pure, reusable graph library (`spec+status → graph`), including: every
  volume/credential/network node type, dedup, deterministic ordering, `optional`/`lifecycle`
  labeling, and the backend-PVC-not-yet-created partial-graph path.
- E2E tests covering single and batch endpoints, query-param options, partial success,
  fail-closed selectors, work-reducing filters, typed error reasons, VM/VMI parity, and
  feature-gate-disabled behavior.

## **Implementation Phases**

1. **Library extraction (non-breaking).** Move graph logic into a reusable `pkg/objectgraph`
   package with a pure `Build(spec+status)` core and isolated live enrichers (launcher pod,
   VM backend-storage PVC). Lock current behavior with golden tests first.
2. **Wire format + labels + completeness + fixes (breaking).** Flat deduped node list,
   identity with plural resource + uid, two-axis labels, corrected `optional` semantics, new
   node types (sysprep, ephemeral, memoryDump, containerDisk), `default` log branches, and
   the backend-PVC partial-graph fix. Surface the backend-storage PVC name in VM status so the
   VM graph is fully derivable from `spec + status` (needed for compute-from-manifest).
3. **Transport (breaking).** Query-param options for single-object; batch `POST` endpoint
   with partial success and all selection modes (names, selector, and inline `Objects`
   manifests for restore/deleted-VM); fail-closed selectors; work-reducing filters; opt-in
   existence resolution.
4. **Typed errors.** Stable reasons on single and batch responses.
5. **virtctl + client-go.** Update flags/output, add batch support, fix the `--selector`
   example, regenerate clients/mocks.
6. **Docs.** Contract page: label enums, derived sets, plural/Kind contract, error reasons,
   required-vs-optional-per-version matrix, VM/VMI differences, tree→flat migration note.
7. **Beta flip.** Verify graduation criteria and flip the `ObjectGraph` gate to beta.

## **Feature Lifecycle Phases**

- **Alpha (v1.6+):**
  - Initial hierarchical implementation with basic filtering.
  - Generic inclusion mechanism for external resources (NADs) explored.
- **Beta (proposed v1.11):**
  - Flat deduped wire format, two-axis labels, batch API, typed errors (this revision).
  - Finalized external-resource inclusion (NADs; decision on IPAMClaims).
  - Adapt external repos (e.g. Velero plugin) to the beta API.
  - Documented required-vs-optional matrix and stability guarantees.
- **GA:**
  - Improvements based on feedback; potential watch semantics and informer-backed reads.
