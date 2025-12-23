# Event Ordering Rules

## Overview
The Shadow Ledger System maintains strict ordering guarantees for financial transaction events to ensure data consistency across all services.

## Ordering Mechanisms

### Kafka Topic Partitioning
- Events are partitioned by `accountId` to ensure all transactions for an account are processed in order
- Each partition maintains sequential ordering within the account scope
- Partition key: `accountId` (ensures account-level ordering)

### Event Sequence Numbers
- Each event contains a monotonically increasing sequence number
- Format: `{accountId}-{timestamp}-{sequenceId}`
- Duplicate detection using sequence numbers prevents replay attacks

### Time-based Ordering
- Events include both business timestamp and system timestamp
- Business timestamp: When the transaction occurred
- System timestamp: When the event was received
- Processing uses business timestamp for ordering, system timestamp for drift detection

## Ordering Guarantees

### Account-Level Ordering
```
Account A: Event1 → Event2 → Event3
Account B: Event4 → Event5 → Event6
```
- Events within the same account are processed sequentially
- Cross-account events can be processed in parallel

### Service Processing Order
1. **Event Service**: Validates and enriches incoming events
2. **Shadow Ledger Service**: Consumes events in partition order
3. **Drift Correction Service**: Processes events after ledger updates

### Window Function Ordering
- Shadow ledger uses time-based windows (configurable, default: 1 minute)
- Events within windows are sorted by business timestamp
- Late-arriving events trigger drift detection if outside tolerance window

## Failure Handling
- Failed events are retried with exponential backoff
- Dead letter queue for events exceeding retry limits
- Out-of-order events are buffered for configurable time window (default: 30 seconds)

## Configuration
```yaml
event:
  ordering:
    partition-key: accountId
    buffer-window: 30s
    max-retries: 3
    dead-letter-enabled: true
```
