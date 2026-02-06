# Tasks: Quota Forecasting & Smart Throttling

**Feature**: FEAT-008  
**Spec**: [spec.md](./spec.md)  
**Plan**: [plan.md](./plan.md)

---

## T001: Implement RequestClassifier
**Component**: `src/quota_manager/classifier.py`  
**Estimate**: 2 hours  
**Issue**: [#1](https://github.com/vindicta-platform/Quota-Manager/issues/1)

### Subtasks
- [ ] Create `Priority` enum with P0/P1/P2 levels
- [ ] Create `ClassifiedRequest` model
- [ ] Implement `classify()` method with priority resolution order
- [ ] Write unit tests in `tests/test_classifier.py`

### Acceptance
- Classification from header works
- Classification from query param works
- Explicit priority override works
- Default to P0 (human) when unspecified

---

## T002: Implement PriorityScheduler
**Component**: `src/quota_manager/scheduler.py`  
**Estimate**: 3 hours  
**Issue**: [#2](https://github.com/vindicta-platform/Quota-Manager/issues/2)

### Subtasks
- [ ] Create `QueuedRequest` dataclass
- [ ] Implement priority queue with deque per level
- [ ] Implement `enqueue()` and `dequeue()` methods
- [ ] Add timeout handling (5s max)
- [ ] Write unit tests in `tests/test_scheduler.py`

### Acceptance
- P0 requests always dequeued before P1/P2
- Queue depth tracked per priority
- Timeout prevents hanging

---

## T003: Implement UsageStore
**Component**: `src/quota_manager/store.py`  
**Estimate**: 1.5 hours  
**Issue**: [#3](https://github.com/vindicta-platform/Quota-Manager/issues/3)

### Subtasks
- [ ] Create `UsageRecord` dataclass
- [ ] Implement in-memory storage with list
- [ ] Implement hourly aggregation methods
- [ ] Implement `get_usage_pattern()` builder

### Acceptance
- Records stored with timestamp
- Hourly averages calculated correctly
- Pattern includes weekly multipliers

---

## T004: Implement QuotaForecaster
**Component**: `src/quota_manager/forecaster.py`  
**Estimate**: 2.5 hours  
**Issue**: [#4](https://github.com/vindicta-platform/Quota-Manager/issues/4)

### Subtasks
- [ ] Create `UsagePattern` and `QuotaBudget` models
- [ ] Implement `predict_hourly_usage()` 
- [ ] Implement `calculate_surplus()` with 50% reserve
- [ ] Implement `recommend_batch_window()`
- [ ] Implement `allocate_budget()`
- [ ] Write unit tests in `tests/test_forecaster.py`

### Acceptance
- Prediction uses 7-day average by default
- Human reserve always 50%
- Batch window finds lowest traffic hours
- Cold start uses conservative defaults

---

## T005: Implement SmartThrottler
**Component**: `src/quota_manager/throttle.py`  
**Estimate**: 2 hours  
**Issue**: [#5](https://github.com/vindicta-platform/Quota-Manager/issues/5)

### Subtasks
- [ ] Create `ThrottleLevel` enum
- [ ] Create `ThrottleDecision` dataclass
- [ ] Implement threshold-based `decide()` method
- [ ] Implement `should_alert()` for proactive warnings
- [ ] Write unit tests in `tests/test_throttle.py`

### Acceptance
- P0 never throttled
- P2 throttled per documented thresholds
- Alerts fire at configurable thresholds

---

## T006: Integrate QuotaManager Facade
**Component**: `src/quota_manager/manager.py`  
**Estimate**: 3 hours  
**Issue**: [#6](https://github.com/vindicta-platform/Quota-Manager/issues/6)

### Subtasks
- [ ] Wire all components into QuotaManager
- [ ] Implement `check_quota()` facade method
- [ ] Implement `record_usage()` and `get_budget()`
- [ ] Export new classes in `__init__.py`
- [ ] Write integration tests in `tests/test_integration.py`

### Acceptance
- Full request lifecycle works end-to-end
- All BDD scenarios pass
- Import from package root works

---

## T007: Documentation
**Component**: `docs/forecasting.md`  
**Estimate**: 2 hours  
**Issue**: [#7](https://github.com/vindicta-platform/Quota-Manager/issues/7)

### Subtasks
- [ ] Write user-facing documentation
- [ ] Add usage examples
- [ ] Document configuration options
- [ ] Update README with feature description

### Acceptance
- Docs render correctly in MkDocs
- Examples are copy-pastable

---

## Summary

| Task | Component | Hours | Status |
|------|-----------|-------|--------|
| T001 | classifier.py | 2 | ⬜ |
| T002 | scheduler.py | 3 | ⬜ |
| T003 | store.py | 1.5 | ⬜ |
| T004 | forecaster.py | 2.5 | ⬜ |
| T005 | throttle.py | 2 | ⬜ |
| T006 | manager.py | 3 | ⬜ |
| T007 | docs | 2 | ⬜ |
| **Total** | | **16** | |
