# nginx-helm-test — SSA vs CSA Deep Dive

This repo is a minimal nginx Helm chart deployed via ArgoCD on GKE Autopilot,
used to observe and understand Server-Side Apply (SSA) field ownership,
controller mutations, and how SSA differs from Client-Side Apply (CSA).

**Actual ArgoCD config used in this repo:**
```yaml
syncPolicy:
  automated:
    prune: true
    selfHeal: true
  syncOptions:
    - ServerSideApply=true
    - CreateNamespace=true
# No ignoreDifferences — not needed with SSA
```

---

## Table of Contents

1. [What is CSA and SSA](#1-what-is-csa-and-ssa)
2. [How they differ — the core mechanic](#2-how-they-differ--the-core-mechanic)
3. [Does CSA cause sync loops with controller-added fields?](#3-does-csa-cause-sync-loops-with-controller-added-fields)
4. [How SSA handles controller-added fields](#4-how-ssa-handles-controller-added-fields)
5. [Why ignoreDifferences is NOT needed with SSA](#5-why-ignoredifferences-is-not-needed-with-ssa)
6. [Real cluster output — field ownership proof](#6-real-cluster-output--field-ownership-proof)
7. [How to read managedFields](#7-how-to-read-managedfields)
8. [GKE Autopilot — what it injects and who owns it](#8-gke-autopilot--what-it-injects-and-who-owns-it)
9. [The real reasons to prefer SSA over CSA](#9-the-real-reasons-to-prefer-ssa-over-csa)
10. [How to verify SSA is active on your cluster](#10-how-to-verify-ssa-is-active-on-your-cluster)

---

## 1. What is CSA and SSA

### Client-Side Apply (CSA)

The original `kubectl apply` mechanism. The **client** (kubectl or ArgoCD)
computes what changed and sends the full object to the API server.

It stores a snapshot of the last applied config as an annotation on the object:

```
kubectl.kubernetes.io/last-applied-configuration: {"apiVersion":"v1","kind":"Service",...}
```

On every apply it runs a **3-way merge**:

```
last-applied  ──┐
                ├──► client computes diff ──► sends full object to API server
new manifest  ──┘
live object   ──┘ (used to detect external changes)
```

The merge logic:
- Field in new manifest → set it
- Field in last-applied but NOT in new manifest → delete it
- Field NOT in last-applied AND NOT in new manifest → leave it alone

### Server-Side Apply (SSA)

Introduced in Kubernetes 1.18, GA in 1.22. The **API server** handles all merge
logic. The client only sends the fields it declares — a partial object.

The API server maintains a `managedFields` map on every object, recording which
manager owns which field. On every apply the API server asks:
"does the applying manager conflict with any existing owner?"

```
ArgoCD sends partial object ──► API server checks managedFields
                                         │
                           ┌─────────────┴──────────────┐
                           │                            │
                     No conflict                   Conflict (409)
                    Apply succeeds               Rejected until resolved
                                                 with --force-conflicts
```

---

## 2. How they differ — the core mechanic

| | CSA | SSA |
|---|---|---|
| Merge happens at | Client | API Server |
| Field ownership tracked | ❌ No | ✅ Yes (`managedFields`) |
| Conflict detection | ❌ Silent overwrite | ✅ 409 error |
| Last-applied storage | Annotation on object | `managedFields` in metadata |
| Partial object support | ❌ Sends full object | ✅ Sends only declared fields |
| Multi-manager safety | ❌ Last writer wins silently | ✅ Explicit ownership per field |
| Operation type in managedFields | `Update` | `Apply` |
| ArgoCD diff scope | Full live object vs desired | Only owned fields vs desired |

---

## 3. Does CSA cause sync loops with controller-added fields?

**Not automatically — and this is commonly misunderstood.**

CSA only causes a sync loop when **all three conditions are true**:

```
Condition 1: Field is NOT in your current manifest
Condition 2: Field WAS in a previous last-applied-configuration
Condition 3: A controller keeps re-adding it after CSA removes it
```

### Why controller-added fields are safe with CSA in practice

CSA's 3-way merge logic for a field like `clusterIP`:

```
Is clusterIP in last-applied?  → No (you never put it there)
Is clusterIP in new manifest?  → No
Result: LEAVE IT ALONE ✅
```

Because `clusterIP` was never in `last-applied-configuration`, CSA never
tries to delete it. It is simply invisible to the merge logic.

This is why other apps in this cluster work fine with CSA — fields like
`clusterIP`, `ipFamilies`, `neg annotations` were never in last-applied so
CSA ignores them completely on every sync.

### When CSA DOES cause a loop

The loop only triggers if you first declared the field in your manifest,
committed it, then removed it:

```
Step 1: You add clusterIP: 10.0.0.1 to manifest → last-applied stores it
Step 2: You remove it from manifest
Step 3: CSA sees: "was in last-applied, not in new manifest → DELETE"
Step 4: API server rejects (immutable) OR controller re-adds it
Step 5: Live object still has it → ArgoCD detects drift → go to Step 3
        = sync loop
```

The loop is caused by the **history in last-applied**, not by the field
existing in the live object.

### GKE Autopilot and CSA

On GKE Autopilot, Autopilot's admission webhook mutates Pods at scheduling
time — injecting `ephemeral-storage`, `securityContext`, `tolerations`, etc.
These are injected at the **Pod level**, not the Deployment level. Since ArgoCD
manages Deployments (not Pods directly), these injections never appear in the
Deployment's `last-applied-configuration` and therefore CSA never tries to
remove them.

---

## 4. How SSA handles controller-added fields

With SSA, ArgoCD sends only the fields from your manifest tagged with
`operation: Apply` and field manager `argocd-controller`. The API server
records ArgoCD as owner of exactly those fields — nothing more.

```
ArgoCD owns (from your manifest):     Controllers own (via Update):
──────────────────────────────────    ────────────────────────────────
spec.type                             spec.clusterIP
spec.selector                         spec.clusterIPs
spec.ports[port=80].port              spec.ipFamilies
spec.ports[port=80].targetPort        spec.ipFamilyPolicy
metadata.annotations.tracking-id     spec.sessionAffinity
                                      metadata.annotations.neg
```

When ArgoCD re-syncs, it only re-asserts its own fields. `clusterIP`, `neg`
annotation etc. are not in ArgoCD's ownership set so the API server never
touches them — they cannot be affected by ArgoCD syncs at all.

**The diff engine scope also changes with SSA.** Without `ignoreDifferences`,
ArgoCD with SSA scopes its diff to the fields it owns. Controller-added fields
outside ArgoCD's ownership are not part of the comparison, so they never
appear as OutOfSync.

This is why the repo runs with no `ignoreDifferences` and stays Synced even
though the live objects have many fields the manifest never declared.

---

## 5. Why ignoreDifferences is NOT needed with SSA

`ignoreDifferences` is needed in two situations:

| Situation | Fix |
|---|---|
| Using CSA and controller adds fields → shows as OutOfSync | Add `ignoreDifferences` to tell the diff engine to skip those fields |
| Using SSA but a webhook mutates a field ArgoCD itself owns (e.g. Autopilot changes your declared `resources.cpu`) | Add `ignoreDifferences` for that specific field |

In this repo neither situation applies:
- SSA is active → diff is scoped to ArgoCD-owned fields only
- Autopilot injections happen at Pod level → Deployment spec fields ArgoCD
  owns (cpu, memory) are not mutated by Autopilot on the Deployment object

So `ignoreDifferences` adds no value here and is intentionally omitted.

You would add it if, for example, Autopilot started mutating the Deployment's
`resources` directly — then ArgoCD would own `resources.cpu` but Autopilot
would change it, creating a real ownership conflict that SSA would surface as
a 409 or continuous drift.

---

## 6. Real cluster output — field ownership proof

This is actual `managedFields` from this repo on GKE Autopilot.

> Note: GKE strips `managedFields` from normal `kubectl get` responses.
> Use `kubectl get --raw` to see them (shown in section 10).

### Service — one manager only

```json
[
  {
    "manager": "argocd-controller",
    "operation": "Apply",
    "time": "2026-06-04T06:32:02Z",
    "fieldsV1": {
      "f:metadata": {
        "f:annotations": {
          "f:argocd.argoproj.io/tracking-id": {}
        }
      },
      "f:spec": {
        "f:ports": {
          "k:{\"port\":80,\"protocol\":\"TCP\"}": {
            ".": {},
            "f:port": {},
            "f:targetPort": {}
          }
        },
        "f:selector": {},
        "f:type": {}
      }
    }
  }
]
```

Key observations:
- `operation: Apply` confirms SSA is active — CSA would show `Update`
- ArgoCD owns exactly: tracking-id annotation, port, targetPort, selector, type
- `clusterIP`, `ipFamilies`, `sessionAffinity`, `neg annotation` are absent
  from managedFields entirely — no manager claims them via Apply, they were
  set by k8s defaulting and the NEG controller via Update and are invisible
  to ArgoCD's diff

### Deployment — two managers, zero overlap

```
Manager 1: argocd-controller  (operation: Apply)
───────────────────────────────────────────────────
spec.replicas
spec.selector
spec.template.metadata.labels.app
spec.template.spec.containers[nginx].image
spec.template.spec.containers[nginx].imagePullPolicy
spec.template.spec.containers[nginx].ports[80]
spec.template.spec.containers[nginx].resources.limits.cpu
spec.template.spec.containers[nginx].resources.limits.memory
spec.template.spec.containers[nginx].resources.requests.cpu
spec.template.spec.containers[nginx].resources.requests.memory
metadata.annotations.argocd.argoproj.io/tracking-id

Manager 2: kube-controller-manager  (operation: Update, subresource: status)
───────────────────────────────────────────────────────────────────────────────
metadata.annotations.deployment.kubernetes.io/revision
status.availableReplicas
status.conditions[Available]
status.conditions[Progressing]
status.observedGeneration
status.readyReplicas
status.replicas
status.updatedReplicas
```

ArgoCD owns spec. kube-controller-manager owns status. Zero overlap — no
conflicts possible. Notice `ephemeral-storage` is NOT in ArgoCD's ownership
on the Deployment — Autopilot injects it only at the Pod level.

### Pod — three managers at different layers

```
Manager 1: kube-controller-manager  (operation: Update)
────────────────────────────────────────────────────────
metadata.generateName
metadata.labels (app, pod-template-hash)
metadata.ownerReferences (link to ReplicaSet)
spec.containers[nginx].image
spec.containers[nginx].resources.limits.ephemeral-storage    ← Autopilot injected
spec.containers[nginx].resources.requests.ephemeral-storage  ← Autopilot injected
spec.containers[nginx].securityContext.capabilities.drop     ← Autopilot injected
spec.securityContext.seccompProfile                          ← Autopilot injected
spec.tolerations                                             ← Autopilot injected
spec.dnsPolicy
spec.enableServiceLinks
spec.restartPolicy
spec.schedulerName
spec.terminationGracePeriodSeconds

Manager 2: kubelet  (operation: Update, subresource: status)
─────────────────────────────────────────────────────────────
status.conditions (Ready, Initialized, ContainersReady, PodScheduled...)
status.containerStatuses
status.hostIP / hostIPs
status.podIP / podIPs
status.phase
status.startTime

Manager 3: argocd-controller → does NOT appear
────────────────────────────────────────────────
ArgoCD manages the Deployment, not Pods directly.
Pods are created by the ReplicaSet controller.
ArgoCD owns zero Pod fields — Autopilot injections
are completely outside ArgoCD's scope.
```

---

## 7. How to read managedFields

The `fieldsV1` format uses path prefixes:

| Prefix | Meaning | Example |
|---|---|---|
| `f:` | Field name | `f:replicas` = the replicas field |
| `k:` | Map key — identifies a list item by its key fields | `k:{"port":80,"protocol":"TCP"}` |
| `v:` | Exact value | `v:true` |
| `.` | The object/list entry itself | `".": {}` = I own this entry |
| `{}` | Empty object = ownership claim | `"f:type": {}` = I own type |

### Reading a list entry

```json
"f:ports": {
  "k:{\"port\":80,\"protocol\":\"TCP\"}": {
    ".": {},          ← owns the list entry identified by port=80+protocol=TCP
    "f:port": {},     ← owns the port field within it
    "f:targetPort": {}← owns the targetPort field within it
  }
}
```

SSA identifies list items by their **key fields**, not by index position.
This is why SSA handles lists correctly when controllers reorder or insert
items — the key always identifies the right entry regardless of position.
CSA uses index-based merging which breaks when list order changes.

---

## 8. GKE Autopilot — what it injects and who owns it

GKE Autopilot runs an admission webhook that mutates Pods at scheduling time.
These mutations happen after ArgoCD applies the Deployment, when the
ReplicaSet controller creates the Pod — a layer below what ArgoCD manages.

| Field injected | On resource | Owner in managedFields |
|---|---|---|
| `resources.limits.ephemeral-storage: 1Gi` | Pod | kube-controller-manager |
| `resources.requests.ephemeral-storage: 1Gi` | Pod | kube-controller-manager |
| `securityContext.capabilities.drop: [NET_RAW]` | Pod | kube-controller-manager |
| `securityContext.seccompProfile.type: RuntimeDefault` | Pod | kube-controller-manager |
| `tolerations[kubernetes.io/arch=amd64]` | Pod | kube-controller-manager |
| `dnsPolicy: ClusterFirst` | Pod | kube-controller-manager |
| `terminationGracePeriodSeconds: 30` | Pod | kube-controller-manager |
| `cloud.google.com/neg annotation` | Service | neg-controller |
| `deployment.kubernetes.io/revision` | Deployment | kube-controller-manager |

With both CSA and SSA, these fields are safe because:
- **CSA**: they were never in `last-applied` so the 3-way merge ignores them
- **SSA**: they are not in ArgoCD's `managedFields` ownership so the diff
  never includes them

---

## 9. The real reasons to prefer SSA over CSA

CSA works fine in simple cases. The cases where SSA is clearly better:

### HPA / KEDA / VPA conflicts

```
With CSA:
  You set replicas: 3 in manifest → last-applied stores it
  HPA scales to 7
  ArgoCD syncs → CSA sees "was 3 in last-applied, still 3 in manifest" → sets to 3
  HPA scales to 7 again → loop forever ❌

With SSA:
  You set replicas: 3 → argocd-controller owns replicas
  HPA scales to 7 → hpa-controller tries to own replicas → 409 Conflict
  You see the conflict, remove replicas from your manifest
  HPA now owns replicas, ArgoCD never touches it ✅
```

SSA surfaces the conflict explicitly. CSA hides it and causes a loop.

### Multi-manager safety

When multiple tools (ArgoCD, Helm, kubectl, a custom operator) all touch the
same object, CSA silently overwrites. SSA gives each tool explicit ownership
and raises a 409 if two tools try to own the same field — you know immediately
something is wrong.

### Explicit audit trail

`managedFields` answers: "who set this field and when?" — useful for debugging
why a value is what it is in production. CSA's `last-applied` annotation only
tells you what ArgoCD last applied, not who else touched the object.

### Partial object efficiency

ArgoCD only sends fields it owns — not the full object. For large CRDs with
many controller-managed fields this reduces API server load and payload size.

---

## 10. How to verify SSA is active on your cluster

### Check 1 — Confirm no last-applied-configuration annotation

```bash
kubectl get svc nginx-test -n nginx-test \
  -o jsonpath='{.metadata.annotations}' | jq .

# SSA active:  annotation absent ✅
# CSA active:  annotation contains full JSON blob ❌
```

### Check 2 — View managedFields via raw API

GKE strips managedFields from normal `kubectl get` responses. Use raw API:

```bash
# Service
kubectl get --raw \
  "/api/v1/namespaces/nginx-test/services/nginx-test" \
  | jq '[.metadata.managedFields[] | {manager: .manager, operation: .operation}]'

# Deployment
kubectl get --raw \
  "/apis/apps/v1/namespaces/nginx-test/deployments/nginx-test" \
  | jq '[.metadata.managedFields[] | {manager: .manager, operation: .operation}]'

# Pod
POD=$(kubectl get pod -n nginx-test -o jsonpath='{.items[0].metadata.name}')
kubectl get --raw \
  "/api/v1/namespaces/nginx-test/pods/$POD" \
  | jq '[.metadata.managedFields[] | {manager: .manager, operation: .operation}]'
```

Expected — SSA confirmed:
```json
[{ "manager": "argocd-controller", "operation": "Apply" }]
```

If argocd-controller shows `"operation": "Update"` → CSA is still active.

### Check 3 — What ArgoCD owns vs what it doesn't

```bash
# Full ownership breakdown — Service
kubectl get --raw "/api/v1/namespaces/nginx-test/services/nginx-test" \
  | jq '.metadata.managedFields[] | {manager: .manager, owns: (.fieldsV1 | keys)}'

# Full ownership breakdown — Deployment
kubectl get --raw "/apis/apps/v1/namespaces/nginx-test/deployments/nginx-test" \
  | jq '.metadata.managedFields[] | {manager: .manager, owns: (.fieldsV1 | keys)}'

# Only ArgoCD's owned fields
kubectl get --raw "/api/v1/namespaces/nginx-test/services/nginx-test" \
  | jq '.metadata.managedFields[] | select(.manager == "argocd-controller") | .fieldsV1'
```

### Check 4 — Confirm in ArgoCD operation record

```bash
kubectl get application nginx -n argocd -o json \
  | jq '.status.operationState.operation.sync.syncOptions'

# Should contain "ServerSideApply=true"
```

---

## Summary

```
CSA                                    SSA (this repo)
──────────────────────────────         ──────────────────────────────
Merge at client                        Merge at API server
No ownership tracking                  managedFields tracks every field
last-applied annotation present        No last-applied annotation
operation: Update                      operation: Apply
Diff = desired vs full live object     Diff = desired vs owned fields only
Controller fields safe IF never        Controller fields always safe —
  in last-applied (fragile)              ownership boundary is explicit
Multi-manager: silent overwrite        Multi-manager: 409 conflict
HPA conflict: silent loop              HPA conflict: explicit 409
ignoreDifferences often needed         ignoreDifferences rarely needed
```

The key insight: SSA makes what ArgoCD **manages** and what ArgoCD **declares**
the same set. Controller-added fields exist outside that set by definition.
ArgoCD's diff is scoped to its own ownership, so fields it never claimed
are never flagged as drift — no ignoreDifferences, no sync loop, no conflict.
