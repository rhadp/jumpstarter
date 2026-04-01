# Requirements Document

## Introduction

This specification addresses a memory leak in the Jumpstarter controller service where `listenQueues` entries are never removed after listener disconnect, causing unbounded growth of the `sync.Map` over time.

## Glossary

- **listenQueues**: A `sync.Map` field on `ControllerService` that maps lease names (strings) to buffered channels of `*pb.ListenResponse`. Used to pass dial notifications from `Dial()` to `Listen()`.
- **Listen()**: A server-streaming gRPC method called by exporters to receive dial notifications for a specific lease.
- **Dial()**: A unary gRPC method called by clients to initiate a connection through a lease. Sends a `ListenResponse` (containing router endpoint and token) into the listen queue for the target lease.
- **Lease**: A Kubernetes custom resource representing a reserved connection between a client and an exporter.
- **Exporter**: A component that exposes hardware devices and listens for client connections via leases.

## Requirements

### Requirement 1: Cleanup of listenQueues on Listener Disconnect

**User Story:** As an operator running the Jumpstarter controller, I want listen queue entries to be cleaned up when listeners disconnect, so that the controller does not leak memory over time.

#### Acceptance Criteria

1. [01-REQ-1.1] WHEN a `Listen()` call returns due to context cancellation, THE controller service SHALL remove the corresponding entry from `listenQueues`.
2. [01-REQ-1.2] WHEN a `Listen()` call returns due to a `stream.Send` failure, THE controller service SHALL remove the corresponding entry from `listenQueues`.
3. [01-REQ-1.3] WHEN a `Listen()` call returns for any reason after a successful `LoadOrStore`, THE controller service SHALL remove the entry from `listenQueues` before the function completes.

#### Edge Cases

1. [01-REQ-1.E1] IF `Dial()` calls `LoadOrStore` concurrently with `Listen()` cleanup, THEN THE controller service SHALL not panic or deadlock.
2. [01-REQ-1.E2] IF a new `Listen()` call arrives after a previous `Listen()` for the same lease has cleaned up, THEN THE controller service SHALL create a new channel and function correctly.
3. [01-REQ-1.E3] IF `Dial()` sends a message to a channel that is subsequently removed by `Listen()` cleanup, THEN THE controller service SHALL not panic.

### Requirement 2: No Regression in Dial-Listen Communication

**User Story:** As a developer, I want the memory leak fix to preserve the existing dial-listen notification flow, so that client connections still work correctly.

#### Acceptance Criteria

1. [01-REQ-2.1] WHEN `Dial()` sends a `ListenResponse` to a listen queue and `Listen()` is active for that lease, THE controller service SHALL deliver the response to the listener's gRPC stream.
2. [01-REQ-2.2] THE controller service SHALL continue to use a buffered channel with capacity 8 for listen queues.

#### Edge Cases

1. [01-REQ-2.E1] IF `Listen()` is not active when `Dial()` sends a message, THEN THE controller service SHALL allow `Dial()` to block on the send until its own context is cancelled.
