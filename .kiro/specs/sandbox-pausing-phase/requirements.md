# Requirements Document

## Introduction

This document specifies the requirements for implementing a new "Pausing" phase in the Sandbox lifecycle to address the issue where sandboxes transition directly from "Running" to "Paused" phase before the underlying Pod deletion completes. The Pausing phase will serve as a transitional state that accurately reflects when hibernation is in progress, improving user clarity and operational transparency.

## Glossary

- **Sandbox**: A Kubernetes custom resource representing an isolated agent workload environment backed by a Pod
- **Sandbox_Controller**: The controller-runtime based operator that reconciles Sandbox resources
- **Phase**: The current lifecycle state of a Sandbox (Pending, Running, Pausing, Paused, Resuming, Upgrading, etc.)
- **SandboxPaused_Condition**: A metav1.Condition that tracks the pause operation status (True when paused, False when in progress)
- **Pod**: The underlying Kubernetes Pod resource that provides the runtime environment for a Sandbox
- **Hibernation**: The process of pausing a Sandbox by deleting its Pod while preserving its IP/memory/filesystem state
- **Rate_Limiter**: Component that throttles high-priority sandbox creation based on creating sandbox count

## Requirements

### Requirement 1: Pausing Phase Definition

**User Story:** As a platform developer, I want a new "Pausing" phase constant in the API types, so that the phase enum includes the transitional state between Running and Paused.

#### Acceptance Criteria

1. THE Sandbox_API SHALL define a new constant `SandboxPausing` with the value "Pausing" in the `SandboxPhase` type
2. THE Sandbox_API SHALL include documentation comments explaining that Pausing represents a sandbox undergoing hibernation
3. THE Sandbox_API SHALL maintain all existing phase constants without modification

### Requirement 2: State Transition Logic

**User Story:** As a platform user, I want sandboxes to transition through Pausing phase during hibernation, so that I can accurately monitor when pause operations are in progress versus completed.

#### Acceptance Criteria

1. WHEN a Sandbox is in Running phase AND spec.paused becomes true, THEN THE Sandbox_Controller SHALL transition the Phase to Pausing immediately
2. WHEN a Sandbox is in Pausing phase AND the SandboxPaused_Condition status is False, THEN THE Sandbox_Controller SHALL maintain the Pausing phase
3. WHEN a Sandbox is in Pausing phase AND the SandboxPaused_Condition status transitions to True, THEN THE Sandbox_Controller SHALL transition the Phase to Paused
4. WHEN a Sandbox is in Pausing phase, THEN THE Sandbox_Controller SHALL invoke the appropriate pause handler to delete the Pod
5. THE Sandbox_Controller SHALL NOT transition directly from Running to Paused phase when spec.paused becomes true

### Requirement 3: Metrics Integration

**User Story:** As a platform operator, I want metrics to track the Pausing phase, so that I can monitor sandbox hibernation operations through Prometheus.

#### Acceptance Criteria

1. THE Sandbox_Metrics SHALL include `SandboxPausing` in the `allPhases` array for proper metric cleanup
2. WHEN a Sandbox enters Pausing phase, THEN THE Sandbox_Metrics SHALL emit `sandbox_status_phase{phase="Pausing"}` with value 1
3. WHEN a Sandbox exits Pausing phase, THEN THE Sandbox_Metrics SHALL delete the `sandbox_status_phase{phase="Pausing"}` metric series
4. THE Sandbox_Metrics SHALL continue to accurately track `sandbox_pause_duration_seconds` histogram for pause operations
5. THE Sandbox_Metrics SHALL continue to accurately track `sandbox_pause_total` counter for pause operation outcomes

### Requirement 4: Rate Limiting Behavior

**User Story:** As a platform operator, I want the rate limiter to correctly handle Pausing phase sandboxes, so that hibernating sandboxes don't incorrectly block high-priority sandbox creation.

#### Acceptance Criteria

1. WHEN the Rate_Limiter evaluates whether a Sandbox is creating, THEN THE Rate_Limiter SHALL treat Pausing phase as non-creating (similar to Paused phase)
2. WHEN a Sandbox transitions to Pausing phase, THEN THE Rate_Limiter SHALL decrement the creating sandbox count if the sandbox was previously counted as creating
3. THE Rate_Limiter SHALL allow high-priority sandbox creation to proceed without delay when Pausing phase sandboxes exist

### Requirement 5: Backward Compatibility

**User Story:** As a platform maintainer, I want the Pausing phase implementation to be backward compatible, so that existing pause/resume workflows continue to function correctly.

#### Acceptance Criteria

1. WHEN a Sandbox completes the Pausing to Paused transition, THEN THE Sandbox SHALL resume normally when spec.paused is set to false
2. WHEN a Sandbox is in Pausing or Paused phase, THEN THE Sandbox_Controller SHALL handle the SandboxResumed_Condition correctly during resume operations
3. THE Sandbox_Controller SHALL maintain the existing pause and resume condition update logic without breaking changes
4. WHEN a Sandbox in Pausing phase receives spec.paused=false before pause completes, THEN THE Sandbox_Controller SHALL handle the state transition gracefully

### Requirement 6: Reconciliation Loop Handling

**User Story:** As a platform developer, I want the reconciliation loop to properly handle the Pausing phase, so that the controller processes Pausing sandboxes through the correct execution path.

#### Acceptance Criteria

1. THE Sandbox_Controller SHALL include a case handler for `SandboxPausing` phase in the Reconcile function's phase switch statement
2. WHEN processing a Sandbox in Pausing phase, THEN THE Sandbox_Controller SHALL delegate to the pause control handler
3. THE Sandbox_Controller SHALL update the Sandbox status after processing Pausing phase sandboxes
4. WHEN a Sandbox in Pausing phase encounters an error during pause operation, THEN THE Sandbox_Controller SHALL surface the error appropriately in status conditions
