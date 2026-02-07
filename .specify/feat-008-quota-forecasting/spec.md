# Specification: Quota Forecasting & Smart Throttling

**Spec ID**: FEAT-008  
**Created**: 2026-02-05  
**Status**: Draft  
**Priority**: High  
**Target**: Quota-Manager v0.2.0

---

## 1. Executive Summary

Implement predictive quota forecasting using historical usage patterns to proactively throttle background tasks before hitting limits, while ensuring human requests always have priority.

### Core Principle
> **Human requests ALWAYS have priority. Never block a human request.**

---

## 2. User Stories

### US-001: Human Request Priority
**As a** platform user  
**I want** my requests to never be blocked by background tasks  
**So that** I always have a responsive experience

**Acceptance Criteria**:
- Human requests are served immediately regardless of quota level
- Human requests draw from a reserved budget (50% of total)
- Background tasks are paused/throttled before human quota is impacted

### US-002: Efficient Background Processing
**As a** system operator  
**I want** background tasks to use surplus quota  
**So that** we maximize API utilization without waste

**Acceptance Criteria**:
- Background tasks consume only unused human reserve
- Batch jobs are scheduled during predicted low-traffic windows
- System reports optimal batch processing windows

### US-003: Proactive Quota Alerts
**As a** platform administrator  
**I want** alerts before quota exhaustion  
**So that** I can take preventive action

**Acceptance Criteria**:
- Alerts fire at configurable thresholds (default: 25%, 10%)
- Alerts include prediction of when quota will be exhausted
- Recommendations to pause batch jobs when appropriate

### US-004: Request Classification
**As a** system integrator  
**I want** requests to be automatically classified by priority  
**So that** the scheduler can make intelligent throttling decisions

**Acceptance Criteria**:
- Three classification tiers: `human_interactive`, `background_batch`, `system_health`
- Classification based on request metadata/headers
- Classification can be overridden via explicit parameter

---

## 3. Acceptance Scenarios (BDD)

```gherkin
Feature: Smart Quota Management

  Background:
    Given the total daily quota is 10,000 requests
    And the human reserve is set to 50%

  Scenario: Human request during low quota
    Given the quota is at 15% remaining (1,500 requests)
    And background tasks are paused
    When a human makes an API request
    Then the request should be served immediately
    And the request should count against human budget
    And background tasks remain paused

  Scenario: Background throttling at medium quota
    Given the quota is at 40% remaining (4,000 requests)
    When a background batch job is queued
    Then it should be processed at reduced rate (50%)
    And human requests remain unaffected
    And the scheduler logs the throttling decision

  Scenario: Background processing during surplus
    Given the quota is at 80% remaining (8,000 requests)
    And predicted human usage for this hour is 200 requests
    When background tasks check for available budget
    Then they should receive a budget of at least 4,800 requests
    And the surplus (unused reserve) is allocated to background

  Scenario: Quota forecast alert
    Given historical usage shows heavy traffic at 2pm (500 requests/hour)
    And it is currently 1pm
    And remaining quota is 600 requests
    When the forecaster runs its hourly check
    Then an alert should recommend pausing batch jobs
    And the alert should show "Predicted usage: 500, Remaining: 600"

  Scenario: Request classification from header
    Given a request arrives with header "X-Request-Priority: background"
    When the classifier processes the request
    Then the request should be classified as "background_batch"
    And queued according to P2 priority

  Scenario: Human request exhausts quota
    Given the quota is at 0% remaining
    And the human reserve has 100 requests remaining
    When a human makes an API request
    Then the request should be served from the reserve
    And the reserve decrements to 99
    And background tasks remain blocked

  Scenario: Batch window recommendation
    Given usage patterns show lowest traffic between 3am-5am
    When an operator requests optimal batch window
    Then the system should recommend "03:00-05:00"
    And show predicted surplus quota for that window
```

---

## 4. Non-Functional Requirements

### 4.1 Performance
- Request classification must complete in < 1ms
- Quota check overhead must be < 5ms per request
- Forecasting calculations run asynchronously, not blocking requests

### 4.2 Reliability
- Quota tracking must survive process restarts (persistent storage)
- Forecaster must handle missing historical data gracefully
- System health requests never throttled (priority P1)

### 4.3 Observability
- All throttling decisions logged with reason
- Metrics exposed: quota_remaining, requests_by_class, throttle_events
- Dashboard-compatible metrics format (Prometheus/OpenMetrics)

---

## 5. Out of Scope (v0.2.0)

- Multi-tenant quota isolation
- Real-time quota negotiation with external APIs
- Machine learning-based prediction (using simple statistical model)
- Distributed quota synchronization across multiple nodes

---

## 6. Open Questions

1. **Historical data bootstrap**: How should the system behave with no historical data? Proposed: Use conservative defaults (assume 100% of reserve needed for humans)

2. **Alert delivery mechanism**: Should alerts be logged, webhooks, or both? Proposed: Start with logging, add webhook support in v0.3.0

3. **Throttle rate granularity**: Should background throttling be percentage-based or absolute rate? Proposed: Percentage-based (50%, 25%, 0%)

---

## 7. References

- [Proposal: feat-008-quota-forecasting](../docs/proposals/feat-008-quota-forecasting.md)
- [ROADMAP.md](../ROADMAP.md)
- Platform constitution: "Human requests ALWAYS have priority"

---

## CLARIFY CYCLE 1: Ambiguity Search

**Objective**: Identify vague terms and unclear specifications in the draft.

### Ambiguities Identified

| # | Ambiguous Term | Location | Resolution |
|---|----------------|----------|------------|
| 1 | "reduced rate" | Scenario: Background throttling | **Clarified**: 50% of normal rate, meaning if normal is 100 req/sec, throttled is 50 req/sec |
| 2 | "surplus quota" | US-002 | **Clarified**: Unused portion of human reserve after predicted human usage is calculated |
| 3 | "low quota" | Scenario: Human request during low quota | **Clarified**: < 25% of daily quota remaining |
| 4 | "immediately" | US-001 | **Clarified**: Within 5ms of quota check, no queuing |
| 5 | "proactive alerts" | US-003 | **Clarified**: Alerts fire when forecaster predicts exhaustion within 2 hours |
| 6 | "request metadata/headers" | US-004 | **Clarified**: Specifically `X-Request-Priority` header or `priority` query parameter |
| 7 | "batch jobs" | Multiple | **Clarified**: Any request classified as `background_batch`, not specifically cron jobs |

### Clarification Status
✅ All identified ambiguities have been resolved above.

---

## CLARIFY CYCLE 2: Component Impact Analysis

**Objective**: Trace which modules need modification or creation.

### New Components Required

| Component | File Path | Responsibility |
|-----------|-----------|----------------|
| **RequestClassifier** | `src/quota_manager/classifier.py` | Classify requests by priority tier (P0/P1/P2) |
| **PriorityScheduler** | `src/quota_manager/scheduler.py` | Queue and dispatch requests by priority |
| **QuotaForecaster** | `src/quota_manager/forecaster.py` | Predict hourly usage from historical patterns |
| **SmartThrottler** | `src/quota_manager/throttle.py` | Apply throttling rules based on quota state |

### Existing Components to Modify

| Component | File Path | Changes Required |
|-----------|-----------|------------------|
| **QuotaManager** | `src/quota_manager/manager.py` | Integrate classifier, scheduler, forecaster, throttler |
| **pyproject.toml** | `pyproject.toml` | Add any new dependencies (asyncio, caching) |

### New Test Files

| Test File | Coverage |
|-----------|----------|
| `tests/test_classifier.py` | Request classification logic |
| `tests/test_scheduler.py` | Priority queue behavior |
| `tests/test_forecaster.py` | Usage prediction accuracy |
| `tests/test_throttle.py` | Throttling decision logic |
| `tests/test_integration.py` | End-to-end quota management |

### Dependency Analysis
- No external dependencies beyond existing `pydantic`
- Uses Python stdlib `collections.deque` for priority queue
- Uses `datetime` for time-based predictions
- Storage: Initially in-memory dict, future iteration adds SQLite

---

## CLARIFY CYCLE 3: Edge Case & Failure Analysis

**Objective**: Identify how the feature could fail and define handling.

### Edge Cases

| # | Edge Case | Expected Behavior |
|---|-----------|-------------------|
| 1 | **Zero historical data** | Use conservative defaults: assume 100% human usage |
| 2 | **Quota reset mid-day** | Detect reset, restart counters, log anomaly |
| 3 | **All requests are human** | Background budget = 0, no throttling needed |
| 4 | **All requests are background** | Throttle heavily once < 50% quota |
| 5 | **Negative surplus calculation** | Clamp to 0, emit warning alert |
| 6 | **Clock skew in hourly prediction** | Use server time, not request time |
| 7 | **Rapid quota exhaustion** | Emergency mode: block all background, alert immediately |

### Failure Modes

| Failure | Impact | Mitigation |
|---------|--------|------------|
| **Forecaster crashes** | No predictions available | Fallback to 50/50 split, log error |
| **Classifier misclassifies human as background** | Human request throttled | Priority override header, allow manual correction |
| **Storage lost (process restart)** | Historical data gone | Persist to file every N minutes |
| **Infinite loop in scheduler** | System hang | Timeout on queue operations (5s max) |
| **Race condition on quota decrement** | Overspend quota | Use atomic operations or locks |

### Recovery Strategies
1. **Graceful degradation**: If any component fails, system continues with conservative defaults
2. **Circuit breaker**: After 3 forecaster failures, disable forecasting for 10 minutes
3. **Audit trail**: All throttling decisions logged for post-mortem analysis

### Clarification Status
✅ All edge cases and failure modes documented with mitigations.
