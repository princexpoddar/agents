# Proposal: E2B-Compatible Volume Management APIs

**Status:** Provisional | **Issue:** #505  
**Related Issues:** #504 (network/egress API parity) · #506 (lifecycle/refresh API parity)


---

## Problem Statement and Motivation

E2B Python SDK ≥ 2.25.0 exposes first-class volume management. Operators running
`sandbox-manager` as an E2B-compatible backend who upgrade their SDK will immediately hit:

- `POST /volumes` → `404` — volume creation fails
- `GET /volumes` → `404` — volume listing fails  
- `POST /sandboxes` silently drops `volume_mounts` — mounts are lost with no error

The internal machinery already exists: `CSIMountConfig`, `SandboxClaim.spec.dynamicVolumesMount`,
and `ProcessCSIMounts` in `claim.go` / `clone.go`. What is missing is a volume
registry, four REST endpoints, and a first-class `volume_mounts` field on sandbox create.

---

## Goals and Non-Goals

**Goals**
- Implement `GET /volumes`, `POST /volumes`, `GET /volumes/{volumeID}`, `DELETE /volumes/{volumeID}`
- Add `volume_mounts` to `POST /sandboxes`
- Enforce volume-name uniqueness and cross-namespace PV isolation
- Block deletion of volumes actively mounted by running sandboxes
- Full backward compatibility — existing CSI metadata extension keys continue to work

**Non-Goals (deferred)**
- Dynamic provisioning via StorageClass / PVC-backed volumes (see Alternatives)
- Volume snapshotting / forking
- `reclaimPolicy: Archive`
- Cross-cluster replication

---

## High-Level Design

Volumes are persisted as a new namespace-scoped CRD `SandboxVolume` (`svol`).
`metadata.name` is the sole `volumeID` (`vol-` + 8 hex chars of UUIDv4) — no
status-field duplication. List/get reads are served from the informer cache;
writes go direct to the API server.

The existing handler → manager → infra → CRD layering is followed unchanged:

```
pkg/servers/e2b/volume.go          ← HTTP handlers
  └── pkg/sandbox-manager/volume.go     ← orchestration + metrics
        └── infra/interface.go               ← interface extension
              └── infra/sandboxcr/volume.go      ← Kubernetes impl
```

`resolveVolumeMounts` lives in `pkg/servers/e2b/` (not `models/`) to avoid an
upward import cycle. It runs before `ClaimSandbox` — no pool sandbox is claimed
if any `volumeID` lookup fails.

**PV reservation** uses a two-step atomic protocol: a name-lock ConfigMap enforces
name uniqueness; an optimistic-lock annotation patch (`agents.kruise.io/volume-reserved-by`)
on the PV enforces single-owner binding. Orphaned annotations from failed rollbacks
are cleared by a periodic controller GC loop (10-min period).

**ReadWriteOnce enforcement** happens in `ResolveVolumeToCsiMount` at sandbox-create
time, not at `POST /volumes` — the create endpoint has no `readOnly` field, so
enforcement there is structurally impossible.

---

## API and Data Model Changes

### New endpoints

| Method | Path | Description |
|---|---|---|
| `POST` | `/volumes` | Register a named volume bound to a pre-provisioned PV |
| `GET` | `/volumes` | List volumes (namespace-scoped, cache-served) |
| `GET` | `/volumes/{volumeID}` | Get volume metadata |
| `DELETE` | `/volumes/{volumeID}` | Delete; `?force=true` bypasses mount-safety guard |

### `POST /volumes` request / response

```json
// Request
{ "name": "my-workspace", "pvName": "user-pv-10gb-001", "sizeGB": 10, "reclaimPolicy": "Retain" }

// Response 201
{ "volumeID": "vol-3f2a1b0c", "name": "my-workspace", "pvName": "user-pv-10gb-001",
  "sizeGB": 10, "phase": "Pending", "reclaimPolicy": "Retain", "createdAt": "..." }
```

`sizeGB` in the request is a hint — rejected `422` if PV capacity is less.
Response always reports actual PV capacity, never the request value.

### `POST /sandboxes` — new field

```json
{
  "templateID": "my-template",
  "volume_mounts": [
    { "volumeID": "vol-3f2a1b0c", "mountPath": "/workspace", "readOnly": false }
  ]
}
```

Optional field (nil = ignored). Resolves `volumeID → pvName` via `SandboxVolume`
CR lookup, then feeds the existing `ProcessCSIMounts` pipeline unchanged.

### `SandboxVolume` CRD

```
spec:   name, pvName, reclaimPolicy (Retain | Delete)
status: phase, actualSizeGB, accessModes, mountedBy [], mountCount, conditions
```

Phases: `Pending | Ready | InUse | Deleting | Failed`

`reclaimPolicy: Delete` causes the controller to delete the backing PV when
the `SandboxVolume` CR is deleted. `Retain` leaves the PV intact.

---

## State Transitions and Compatibility

### Volume lifecycle

```
POST /volumes
     │
  Pending ──[controller verifies PV]──► Ready ──[sandbox mounts]──► InUse
     │                                    │                             │
     └──[PV missing / mismatch]──► Failed └──[all sandboxes deleted]──► Ready
                                                                         │
                                                            DELETE /volumes/{id}
                                                                         │
                                                                     Deleting ──[finalizer]──► [Absent]
```

The controller reconciles `mountedBy` on Sandbox watch events and via a 5-minute
periodic loop to handle missed events.

### Compatibility

| Existing behavior | After this change |
|---|---|
| CSI mounts via `e2b.agents.kruise.io/csi-*` metadata keys | Unchanged — additive; both paths work in the same request |
| `SandboxClaim.spec.dynamicVolumesMount` | Unchanged |
| `clone.go` CSI annotation resolution | Unchanged |
| `agent-runtime` CSI mount providers | Unchanged — output feeds same `ProcessCSIMounts` |

---

## Migration Strategy

None required. `SandboxVolume` is a net-new CRD. No existing object changes schema.
`volume_mounts` on `POST /sandboxes` is optional — old clients are unaffected.

---

## Failure Scenarios and Edge Cases

| Scenario | Behaviour |
|---|---|
| PV not found | `404` |
| PV not in `Available` phase | `422` |
| PV namespace label mismatch | `403` |
| PV capacity < requested `sizeGB` | `422` |
| PV already reserved (concurrent race) | `409` |
| Name already taken in namespace | `409` |
| Volume count at namespace limit | `429` |
| `DELETE` while mounted, no `?force` | `409` with sandbox list |
| `DELETE ?force=true` while mounted | `202` with warning + affected sandbox IDs |
| `DELETE` called twice (CR already gone) | `404` (idempotent) |
| `DELETE` called twice (finalizer still running) | `202` — "already deleting" |
| `volume_mounts` with volumeID from another namespace | `400` |
| ReadWriteOnce PV already mounted read-write | `409` from `ResolveVolumeToCsiMount` |
| Rollback fails mid `POST /volumes` | PV-orphan GC clears annotation within 10 min |
| Controller crash mid-finalizer (`reclaimPolicy: Delete`) | Finalizer re-runs on restart; PV delete is idempotent |

---



## Rollout Plan

1. Introduce the `SandboxVolume` CRD and controller.
2. Add the `/volumes` API surface.
3. Add `volume_mounts` support to sandbox creation.
4. Validate compatibility with existing CSI-based mount workflows.
5. Roll out behind normal deployment updates to `sandbox-manager` and the controller.

**Rollback:** remove the route registrations and controller deployment, then
delete the CRD after confirming no `SandboxVolume` objects remain.

---

## Alternatives Considered

### PVC-backed volumes _(preferred long-term)_

PVCs are namespace-scoped natively, which eliminates the name-lock ConfigMap and
PV annotation reservation protocol entirely — isolation is enforced by Kubernetes
RBAC rather than a custom label convention.

**Why deferred:** the existing `CSIMountConfig` pipeline references `pvName` directly.
Switching requires changing `CSIMountConfig`, the `agent-runtime` sidecar, and
`SandboxClaim` — a cross-cutting change beyond this feature's scope. When dynamic
provisioning is added, this path will need revisiting. The current PV-based approach prioritizes compatibility with the existing CSI mount pipeline while keeping the scope of this feature manageable. PVC-backed storage remains a strong candidate for future work, particularly if dynamic provisioning is introduced.

### In-memory volume registry

Not resilient to manager restarts. Rejected.

### Volume metadata in ConfigMaps only

No typed validation, no informer-backed listing, inconsistent with project patterns. Rejected.
