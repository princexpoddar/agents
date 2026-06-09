# Implementation Plan: Sandbox Pausing Phase

## Overview

This implementation plan provides a structured approach to adding the "Pausing" phase to the Sandbox lifecycle. The work is divided into discrete, incremental steps that build upon each other, with testing integrated at each stage to validate correctness early.

The implementation follows the state machine pattern used throughout the Sandbox controller, adding the new Pausing phase as a transitional state between Running and Paused. Each task is scoped to be independently reviewable and includes specific requirements references for traceability.

## Tasks

- [x] 1. Add Pausing phase constant to API types
  - Modify `api/v1alpha1/sandbox_types.go` to add `SandboxPausing` constant
  - Add documentation comment explaining the Pausing phase represents hibernation in progress
  - Run `make generate manifests` to update generated code and CRDs
  - _Requirements: 1.1, 1.2_

- [x] 2. Update controller state machine for Pausing phase
  - [x] 2.1 Modify calculateStatus function for Running→Pausing transition
    - In `pkg/controller/sandbox/sandbox_controller.go`, update the `SandboxRunning` case
    - Change phase assignment from `SandboxPaused` to `SandboxPausing` when `box.Spec.Paused` is true
    - Initialize SandboxPaused condition to False with reason SetPause
    - _Requirements: 2.1, 2.5_

  - [x] 2.2 Add Pausing phase case to calculateStatus
    - Add new `case agentsv1alpha1.SandboxPausing:` in calculateStatus function
    - Implement logic to check SandboxPaused condition status
    - Transition to Paused only when condition.Status is True
    - Handle user cancellation (spec.paused=false during Pausing)
    - _Requirements: 2.2, 2.3, 5.4_

  - [ ]* 2.3 Write property test for Running→Pausing transition
    - **Property 1: Running to Pausing transition**
    - **Validates: Requirements 2.1, 2.5**
    - Create `pkg/controller/sandbox/pause_properties_test.go`
    - Test that Running phase with spec.paused=true transitions to Pausing (not Paused)
    - Run 100 iterations with varied sandbox configurations

  - [ ]* 2.4 Write property test for Pausing phase stability
    - **Property 2: Pausing phase stability during operation**
    - **Validates: Requirements 2.2**
    - Test that Pausing phase with condition=False remains Pausing

  - [ ]* 2.5 Write property test for Pausing→Paused completion
    - **Property 3: Pausing to Paused completion transition**
    - **Validates: Requirements 2.3**
    - Test that Pausing phase with condition=True transitions to Paused

- [x] 3. Add Pausing phase handler to reconcile loop
  - [x] 3.1 Add SandboxPausing case to Reconcile switch statement
    - In `pkg/controller/sandbox/sandbox_controller.go`, add case for `agentsv1alpha1.SandboxPausing`
    - Delegate to `r.EnsureSandboxPaused(ctx, args)` (same handler as Paused phase)
    - Ensure error handling is consistent with other phases
    - _Requirements: 2.4, 6.1, 6.2_

  - [ ]* 3.2 Write unit test for Pausing phase reconciliation
    - Test that Pausing phase invokes the correct pause handler
    - Test error propagation from pause handler
    - _Requirements: 6.2, 6.4_

- [x] 4. Checkpoint - Verify core state machine tests pass
  - Ensure all tests pass for the controller state machine
  - Verify that calculateStatus and Reconcile correctly handle Pausing phase
  - Ask the user if questions arise

- [x] 5. Update metrics for Pausing phase
  - [x] 5.1 Add SandboxPausing to allPhases array
    - In `pkg/controller/sandbox/metrics.go`, add `agentsv1alpha1.SandboxPausing` to allPhases slice
    - This ensures metric cleanup for Pausing phase
    - _Requirements: 3.1_

  - [ ]* 5.2 Write property test for Pausing phase metrics
    - **Property 4: Pausing phase metrics**
    - **Validates: Requirements 3.2, 3.3**
    - Test that sandbox_status_phase{phase="Pausing"} is emitted correctly
    - Test that metric series is deleted when exiting Pausing

  - [ ]* 5.3 Write property test for pause metrics continuation
    - **Property 5: Pause metrics continuation**
    - **Validates: Requirements 3.4, 3.5**
    - Test that sandbox_pause_duration_seconds histogram continues to work
    - Test that sandbox_pause_total counter continues to work

- [x] 6. Update rate limiter for Pausing phase
  - [x] 6.1 Add Pausing phase to isCreatingSandbox function
    - In `pkg/controller/sandbox/core/rateLimiter.go`, add `agentsv1alpha1.SandboxPausing` to non-creating phases check
    - This ensures Pausing sandboxes don't block high-priority creation
    - _Requirements: 4.1, 4.2_

  - [ ]* 6.2 Write property test for rate limiter behavior
    - **Property 6: Rate limiter non-creating treatment**
    - **Validates: Requirements 4.1, 4.2, 4.3**
    - Test that isCreatingSandbox returns false for Pausing phase
    - Test that high-priority creation is not blocked by Pausing sandboxes

- [ ] 7. Add comprehensive unit tests
  - [ ]* 7.1 Write unit tests for state transitions
    - In `pkg/controller/sandbox/sandbox_controller_test.go`, add table-driven tests
    - Test Running→Pausing→Paused happy path
    - Test user cancellation during Pausing (spec.paused toggled to false)
    - Test edge case: missing SandboxPaused condition during Pausing
    - _Requirements: 2.1, 2.2, 2.3, 5.4_

  - [ ]* 7.2 Write unit tests for metrics
    - In `pkg/controller/sandbox/metrics_test.go`, add tests for Pausing phase
    - Test metric emission when entering Pausing
    - Test metric cleanup when exiting Pausing
    - Test that existing pause metrics continue to work
    - _Requirements: 3.2, 3.3, 3.4, 3.5_

  - [ ]* 7.3 Write unit tests for rate limiter
    - In `pkg/controller/sandbox/core/rateLimiter_test.go`, add tests for Pausing phase
    - Test isCreatingSandbox with Pausing phase sandboxes
    - Test rate limiter track updates
    - _Requirements: 4.1, 4.2, 4.3_

- [ ] 8. Add backward compatibility tests
  - [ ]* 8.1 Write property test for pause-resume round trip
    - **Property 7: Pause-resume round trip**
    - **Validates: Requirements 5.1, 5.2**
    - Test full cycle: Running→pause→Pausing→Paused→unpause→Resuming→Running
    - Verify SandboxResumed condition handled correctly

  - [ ]* 8.2 Write property test for error surfacing
    - **Property 8: Error surfacing**
    - **Validates: Requirements 6.4**
    - Test that errors during pause are surfaced in conditions
    - Test that error messages are descriptive

- [x] 9. Checkpoint - Verify all unit and property tests pass
  - Run focused test suite for modified packages
  - Verify all property tests pass with 100 iterations
  - Ensure no regressions in existing pause/resume functionality
  - Ask the user if questions arise

- [ ] 10. Add integration tests
  - [ ]* 10.1 Write E2E test for full pause cycle
    - In `test/e2e/sandbox/pause_test.go`, add integration test
    - Create sandbox, wait for Running
    - Set spec.paused=true, verify Pausing phase appears
    - Wait for Paused phase, verify Pod deleted
    - Verify SandboxPaused condition reflects each stage
    - _Requirements: 2.1, 2.2, 2.3_

  - [ ]* 10.2 Write E2E test for pause cancellation
    - Test setting spec.paused=false during Pausing phase
    - Verify graceful transition back to Running or proper resume
    - _Requirements: 5.4_

  - [ ]* 10.3 Write E2E test for metrics verification
    - Create sandbox, pause it through Pausing phase
    - Query metrics endpoint
    - Verify Pausing phase metrics emitted correctly
    - _Requirements: 3.2, 3.3, 3.4, 3.5_

- [ ] 11. Final checkpoint - Complete validation
  - Ensure all tests pass (unit, property, integration)
  - Verify metrics are emitted correctly in test environment
  - Verify backward compatibility with existing pause/resume workflows
  - Review logs for any unexpected warnings or errors
  - Ask the user if questions arise

## Notes

- Tasks marked with `*` are optional and can be skipped for faster MVP
- Each task references specific requirements for traceability
- Property tests validate universal correctness properties with 100+ iterations
- Unit tests validate specific examples and edge cases
- Integration tests verify end-to-end behavior in a real cluster
- The implementation maintains full backward compatibility with existing pause/resume logic
- After modifying API types, always run `make generate manifests` to update generated code
