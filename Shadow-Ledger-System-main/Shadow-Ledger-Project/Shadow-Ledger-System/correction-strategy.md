# Drift Correction Strategy

## Overview
The drift correction strategy addresses inconsistencies between primary and shadow ledgers using automated detection and correction mechanisms.

## Drift Detection

### Detection Methods
1. **Real-time Monitoring**: Continuous comparison during event processing
2. **Batch Reconciliation**: Periodic full account reconciliation
3. **Threshold-based Alerts**: Triggers when drift exceeds configured limits

### Detection Triggers
- Balance mismatches exceeding tolerance (default: ±0.01)
- Missing events in shadow ledger
- Timestamp discrepancies beyond acceptable window
- Sequence number gaps

## Correction Algorithms

### 1. Balance Reconciliation
```
IF (primary_balance - shadow_balance) > tolerance THEN
  CREATE correction_event {
    type: BALANCE_CORRECTION,
    amount: difference,
    reason: "Drift detected",
    original_event_ref: event_id
  }
```

### 2. Event Replay
- Identify missing or corrupted events
- Replay events from event store
- Apply events in correct chronological order
- Validate final balance matches primary ledger

### 3. Temporal Corrections
- Handle late-arriving events
- Recalculate affected window balances
- Propagate corrections to subsequent windows

## Correction Workflow

### Automatic Corrections
1. **Minor Drifts** (< 1% of balance): Auto-corrected with audit log
2. **Calculation Errors**: Immediate replay and recalculation
3. **Timestamp Adjustments**: Automatic within 5-minute window

### Manual Approval Required
1. **Major Drifts** (> 1% of balance): Requires ADMIN approval
2. **Data Integrity Issues**: Manual investigation
3. **System Errors**: Administrative review

### Correction Process
```
1. Drift Detection
   ↓
2. Impact Assessment
   ↓
3. Correction Strategy Selection
   ↓
4. Approval Check (if required)
   ↓
5. Correction Execution
   ↓
6. Validation & Audit
```

## Audit Trail
- All corrections logged with:
  - Original values
  - Corrected values
  - Correction reason
  - Approval chain
  - Timestamp of correction

## Configuration
```yaml
drift:
  tolerance: 0.01
  auto-correct-threshold: 1.0
  manual-approval-threshold: 100.0
  correction-window: 24h
  audit-retention: 7y
```

## Recovery Scenarios
- **Service Restart**: Resume from last processed offset
- **Data Corruption**: Full account rebuild from event store
- **Network Partitions**: Conflict resolution using timestamps
