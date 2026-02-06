# Requirements Checklist: Quota Forecasting & Smart Throttling

**Feature**: FEAT-008  
**Spec**: [spec.md](./spec.md)

This checklist serves as "English unit tests" — human-readable verification criteria.

---

## User Story Verification

### US-001: Human Request Priority
- [ ] Human requests are NEVER blocked, even when quota is exhausted
- [ ] Human requests draw from dedicated 50% reserve
- [ ] Background tasks pause before human quota is impacted
- [ ] Response time for human requests < 5ms check overhead

### US-002: Efficient Background Processing
- [ ] Background tasks only use surplus after human prediction
- [ ] System can recommend optimal batch processing window
- [ ] Background utilization maximizes unused quota
- [ ] Background throttling is gradual (100% → 50% → 0%)

### US-003: Proactive Quota Alerts
- [ ] Alert fires when quota drops below 25%
- [ ] Alert fires when quota drops below 10%
- [ ] Alert includes prediction of exhaustion time
- [ ] Alert recommends pausing batch jobs when appropriate

### US-004: Request Classification
- [ ] Requests classified into 3 tiers: P0, P1, P2
- [ ] Classification via `X-Request-Priority` header works
- [ ] Classification via `priority` query param works
- [ ] Explicit parameter overrides all other methods
- [ ] Unclassified requests default to P0 (human)

---

## Scenario Verification

### Scenario: Human request during low quota
- [ ] With 15% quota remaining, human request served immediately
- [ ] Request does not wait in queue
- [ ] Background tasks remain paused throughout

### Scenario: Background throttling at medium quota
- [ ] With 40% quota, background processed at 50% rate
- [ ] Human requests unaffected during throttling
- [ ] Throttling decision logged with reason

### Scenario: Background processing during surplus
- [ ] With 80% quota and low predicted human usage, background gets large budget
- [ ] Surplus calculation: `human_reserve - predicted_human`
- [ ] Budget never exceeds actual remaining quota

### Scenario: Quota forecast alert
- [ ] 1 hour before predicted exhaustion, alert fires
- [ ] Alert shows predicted vs remaining quota
- [ ] Recommendation to pause batch jobs included

### Scenario: Request classification from header
- [ ] Header `X-Request-Priority: background` → P2
- [ ] Header `X-Request-Priority: human` → P0
- [ ] Header `X-Request-Priority: system` → P1

### Scenario: Human request exhausts quota
- [ ] With 0% regular quota, human served from reserve
- [ ] Reserve decrements correctly
- [ ] Background remains blocked

### Scenario: Batch window recommendation
- [ ] System recommends lowest-traffic hours
- [ ] Prediction based on 7-day historical average
- [ ] Window includes surplus estimate

---

## Edge Case Verification

- [ ] Zero historical data → Conservative defaults (assume 100% human)
- [ ] Quota reset mid-day → Counters restart, anomaly logged
- [ ] All requests human → No throttling needed
- [ ] All requests background → Heavy throttling below 50%
- [ ] Negative surplus calculation → Clamped to 0
- [ ] Rapid quota exhaustion → Emergency mode activates

---

## Non-Functional Verification

### Performance
- [ ] Request classification completes in < 1ms
- [ ] Quota check overhead < 5ms per request
- [ ] Forecasting runs asynchronously

### Reliability
- [ ] Process restart → Recovery with conservative defaults
- [ ] Missing historical data → Graceful degradation
- [ ] Component failure → System continues with fallbacks

### Observability
- [ ] All throttling decisions logged
- [ ] Metrics: quota_remaining, requests_by_class, throttle_events
- [ ] Metrics in Prometheus-compatible format

---

## Approval

- [ ] All User Story checkboxes verified
- [ ] All Scenario checkboxes verified
- [ ] All Edge Case checkboxes verified
- [ ] All Non-Functional checkboxes verified

**Verified by**: ________________  
**Date**: ________________
