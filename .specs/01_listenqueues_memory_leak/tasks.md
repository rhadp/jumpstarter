# Implementation Plan: listenQueues Memory Leak Fix

<!-- AGENT INSTRUCTIONS
- Implement exactly ONE top-level task group per session
- Task group 1 writes failing tests from test_spec.md — all subsequent groups
  implement code to make those tests pass
- Follow the git-flow: feature branch from develop -> implement -> test -> merge to develop -> push
- Update checkbox states as you go: [-] in progress, [x] complete
-->

## Overview

This is a small, targeted fix. Task group 1 writes failing tests that verify cleanup behavior. Task group 2 adds the single-line fix (`defer s.listenQueues.Delete(leaseName)`) and verifies all tests pass.

## Test Commands

- Spec tests: `cd controller && go test ./internal/service/ -run "TestListenQueues" -v`
- Unit tests: `cd controller && go test ./internal/service/ -v`
- All tests: `cd controller && go test ./... -v`
- Linter: `cd controller && golangci-lint run ./...`

## Tasks

- [x] 1. Write failing spec tests
  - [x] 1.1 Create test helpers for listenQueues testing
    - Add mock gRPC stream implementation for `Listen()` testing (or extend existing test infrastructure in `controller_service_test.go`)
    - Mock must support: controllable context, injectable `Send()` error, recording of sent messages
    - _Test Spec: TS-01-1 through TS-01-4_

  - [x] 1.2 Write unit tests for cleanup on disconnect
    - `TestListenQueuesCleanupOnContextCancel`: Verify entry removed after context cancellation
    - `TestListenQueuesCleanupOnSendError`: Verify entry removed after `Send()` failure
    - Tests must directly inspect `listenQueues` (white-box, same package)
    - _Test Spec: TS-01-1, TS-01-2_

  - [x] 1.3 Write edge case and concurrency tests
    - `TestListenQueuesConcurrentDialAndCleanup`: Concurrent `LoadOrStore`/`Delete` with no panic
    - `TestListenQueuesReListenAfterCleanup`: New channel created after cleanup
    - `TestListenQueuesSendToRemovedChannel`: No panic sending to orphaned channel reference
    - _Test Spec: TS-01-E1, TS-01-E2, TS-01-E3_

  - [x] 1.4 Write property-style tests
    - `TestListenQueuesCleanupGuarantee`: Parameterized over exit modes, verify cleanup for each
    - `TestListenQueuesConcurrentSafety`: Stress test with random concurrent operations
    - _Test Spec: TS-01-P1, TS-01-P2_

  - [x] 1.5 Write functional preservation tests
    - `TestListenQueuesDialDelivery`: Verify message delivery from queue to stream
    - `TestListenQueuesChannelCapacity`: Verify channel capacity is 8
    - _Test Spec: TS-01-3, TS-01-4, TS-01-P3_

  - [x] 1.V Verify task group 1
    - [x] All spec tests exist and are syntactically valid: `cd controller && go vet ./internal/service/`
    - [x] Cleanup tests FAIL (red) — no `defer Delete` yet
    - [x] Functional preservation tests PASS (existing behavior unchanged)
    - [x] No linter warnings introduced

- [x] 2. Fix listenQueues memory leak
  - [x] 2.1 Add defer cleanup in Listen()
    - Add `defer s.listenQueues.Delete(leaseName)` after the `LoadOrStore` call at line 442
    - This is the only production code change
    - _Requirements: 1.1, 1.2, 1.3_

  - [x] 2.V Verify task group 2
    - [x] All cleanup tests now pass: `cd controller && go test ./internal/service/ -run "TestListenQueues" -v`
    - [x] All existing tests still pass: `cd controller && go test ./internal/service/ -v`
    - [x] No linter warnings introduced
    - [x] Requirements 01-REQ-1.1, 01-REQ-1.2, 01-REQ-1.3 acceptance criteria met

- [x] 3. Checkpoint - Fix Complete
  - Ensure all tests pass across the controller module
  - Verify no regressions in integration tests: `cd controller && go test ./internal/service/ -run "Integration" -v`

## Traceability

| Requirement | Test Spec Entry | Implemented By Task | Verified By Test |
|-------------|-----------------|---------------------|------------------|
| 01-REQ-1.1 | TS-01-1 | 2.1 | TestListenQueuesCleanupOnContextCancel |
| 01-REQ-1.2 | TS-01-2 | 2.1 | TestListenQueuesCleanupOnSendError |
| 01-REQ-1.3 | TS-01-P1 | 2.1 | TestListenQueuesCleanupGuarantee |
| 01-REQ-1.E1 | TS-01-E1 | 2.1 | TestListenQueuesConcurrentDialAndCleanup |
| 01-REQ-1.E2 | TS-01-E2 | 2.1 | TestListenQueuesReListenAfterCleanup |
| 01-REQ-1.E3 | TS-01-E3 | 2.1 | TestListenQueuesSendToRemovedChannel |
| 01-REQ-2.1 | TS-01-3 | (existing) | TestListenQueuesDialDelivery |
| 01-REQ-2.2 | TS-01-4 | (existing) | TestListenQueuesChannelCapacity |
| Property 1 | TS-01-P1 | 2.1 | TestListenQueuesCleanupGuarantee |
| Property 2 | TS-01-P2 | 2.1 | TestListenQueuesConcurrentSafety |
| Property 3 | TS-01-P3 | (existing) | TestListenQueuesDialDelivery |
| Property 4 | TS-01-P4 | 2.1 | TestListenQueuesReListenAfterCleanup |

## Notes

- The production change is a single line: `defer s.listenQueues.Delete(leaseName)`.
- Tests require white-box access to `listenQueues` (same package `service`), which is already the pattern used in `controller_service_test.go`.
- Mock gRPC streams need to satisfy `pb.ControllerService_ListenServer` interface. Check existing test helpers in `suite_test.go` or `controller_service_integration_test.go` for patterns.
- The `Listen()` function also performs authentication and lease validation before reaching `LoadOrStore` — tests for cleanup behavior need to either mock these or use the integration test setup.
