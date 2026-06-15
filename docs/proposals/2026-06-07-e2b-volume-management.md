
# E2B-Compatible Volume Management APIs

## Summary

Add E2B-compatible volume APIs to `sandbox-manager` using existing Kubernetes `PersistentVolume` objects as the backing store.

This proposal introduces:

* `POST /volumes`
* `GET /volumes`
* `GET /volumes/{volumeID}`
* `DELETE /volumes/{volumeID}`
* `volume_mounts` support in `POST /sandboxes`

No new CRDs are introduced. Existing CSI mount functionality is reused.

## Motivation

Recent E2B SDK versions expose first-class volume management APIs. Currently `sandbox-manager` returns `404` for volume endpoints and ignores `volume_mounts` during sandbox creation.

Implementing these APIs improves E2B compatibility while reusing the existing CSI mount infrastructure.

### Goals

* Implement E2B volume management APIs.
* Support persistent volume mounting during sandbox creation.
* Reuse existing CSI mount workflows.
* Enforce namespace isolation.

### Non-Goals/Future Work

* Dynamic volume provisioning.
* PVC-backed storage.
* Volume snapshots and cloning.
* Cross-cluster replication.
* New Kubernetes CRDs.

## Proposal

Registered volumes are represented directly by Kubernetes `PersistentVolume` objects.

A PV is considered registered when it contains:

```yaml
labels:
  agents.kruise.io/volume-owner-namespace: <namespace>
  agents.kruise.io/volume-name: <name>
```

`volumeID` maps directly to `pv.Name`.

### Volume APIs

The following endpoints are added:

* `POST /volumes`
* `GET /volumes`
* `GET /volumes/{volumeID}`
* `DELETE /volumes/{volumeID}`

Volume metadata is stored on PVs and accessed through the existing infrastructure layer.

### Sandbox Creation

`POST /sandboxes` gains an optional `volume_mounts` field.

Each volume reference is resolved to a PV and translated into a `CSIMountConfig` before entering the existing `ProcessCSIMounts` workflow.

No CSI implementation changes are required.

### User Stories

#### Register and manage volumes

As an operator, I can register existing PVs and manage them through E2B-compatible APIs.

#### Mount persistent storage

As a sandbox user, I can attach registered volumes during sandbox creation and access persistent storage inside the sandbox.

## Implementation Details/Notes/Constraints

The implementation follows the existing architecture:

```text
HTTP Handler
    ↓
Sandbox Manager
    ↓
Infrastructure Layer
    ↓
Kubernetes PersistentVolume
```

Namespace ownership is enforced using PV labels.

## Risks and Mitigations

### PVs are cluster-scoped

Ownership is enforced through namespace labels and validation during API operations.

### Mounted volume deletion

Deletion of mounted volumes is rejected unless explicitly forced.

## Alternatives

### SandboxVolume CRD

Rejected.

PersistentVolumes already provide durable storage, informer support, and lifecycle management. Introducing a dedicated CRD would add additional controller and maintenance overhead.

### PVC-backed Volumes

Potential future enhancement but requires broader CSI workflow changes.

## Upgrade Strategy

No migration is required.
The feature is fully additive and does not affect existing sandbox or CSI mount workflows.
