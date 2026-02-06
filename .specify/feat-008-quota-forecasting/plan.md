# Architectural Plan: Quota Forecasting & Smart Throttling

**Plan ID**: FEAT-008  
**Spec Reference**: [spec.md](./spec.md)  
**Created**: 2026-02-05  
**Status**: Draft

---

## 1. Overview

This plan defines the architectural implementation for US-001 through US-004, decomposing the Quota Forecasting & Smart Throttling feature into concrete modules, interfaces, and file signatures.

---

## 2. Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────────┐
│                     QuotaManager (Facade)                            │
│                   src/quota_manager/manager.py                       │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────────────┐   │
│  │ Classifier   │───▶│  Scheduler   │───▶│     Throttler        │   │
│  │ classifier.py│    │ scheduler.py │    │    throttle.py       │   │
│  └──────────────┘    └──────────────┘    └──────────────────────┘   │
│         │                   │                      │                 │
│         │                   ▼                      │                 │
│         │           ┌──────────────┐               │                 │
│         └──────────▶│  Forecaster  │◀──────────────┘                 │
│                     │forecaster.py │                                 │
│                     └──────────────┘                                 │
│                            │                                         │
│                            ▼                                         │
│                    ┌──────────────┐                                  │
│                    │ UsageStore   │                                  │
│                    │  (in-memory) │                                  │
│                    └──────────────┘                                  │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 3. Module Definitions

### 3.1 RequestClassifier (`classifier.py`)

```python
"""Request classification by priority tier."""

from enum import Enum
from pydantic import BaseModel


class Priority(str, Enum):
    """Request priority levels."""
    HUMAN_INTERACTIVE = "P0"      # Never throttle
    SYSTEM_HEALTH = "P1"          # Rarely throttle
    BACKGROUND_BATCH = "P2"       # Throttle when needed


class ClassifiedRequest(BaseModel):
    """A request with assigned priority."""
    request_id: str
    priority: Priority
    source: str  # How classification was determined


class RequestClassifier:
    """Classify incoming requests by priority."""

    def classify(
        self, 
        headers: dict[str, str] | None = None,
        query_params: dict[str, str] | None = None,
        explicit_priority: Priority | None = None
    ) -> Priority:
        """
        Classify a request.
        
        Priority resolution order:
        1. explicit_priority parameter (if provided)
        2. X-Request-Priority header
        3. priority query parameter
        4. Default to HUMAN_INTERACTIVE (safe default)
        """
        ...
```

---

### 3.2 PriorityScheduler (`scheduler.py`)

```python
"""Priority queue for request scheduling."""

from collections import deque
from dataclasses import dataclass, field
from datetime import datetime


@dataclass
class QueuedRequest:
    """A request waiting in the queue."""
    request_id: str
    priority: Priority
    queued_at: datetime = field(default_factory=datetime.now)
    timeout_seconds: float = 5.0


class PriorityScheduler:
    """Manage request queues by priority level."""
    
    def __init__(self):
        self._queues: dict[Priority, deque[QueuedRequest]] = {
            Priority.HUMAN_INTERACTIVE: deque(),
            Priority.SYSTEM_HEALTH: deque(),
            Priority.BACKGROUND_BATCH: deque(),
        }
    
    def enqueue(self, request: QueuedRequest) -> None:
        """Add request to appropriate queue."""
        ...
    
    def dequeue(self) -> QueuedRequest | None:
        """Get next request (P0 first, then P1, then P2)."""
        ...
    
    def queue_depth(self, priority: Priority) -> int:
        """Return current queue depth for a priority level."""
        ...
```

---

### 3.3 QuotaForecaster (`forecaster.py`)

```python
"""Predict quota usage based on historical patterns."""

from dataclasses import dataclass
from datetime import datetime


@dataclass
class UsagePattern:
    """Captured usage pattern for forecasting."""
    hourly_avg: dict[int, float]       # Hour (0-23) -> avg requests
    daily_trend: float                  # Growth rate multiplier
    weekly_pattern: list[float]         # Day-of-week multipliers [Mon=1.0...]


@dataclass 
class QuotaBudget:
    """Allocated quota budget."""
    human_reserve: int
    background_budget: int
    surplus: int  # Unused human reserve available for background


class QuotaForecaster:
    """Predict hourly usage and allocate budgets."""
    
    HUMAN_RESERVE_PERCENTAGE = 50  # Always reserve 50% for human
    
    def __init__(self, usage_store: "UsageStore"):
        self._store = usage_store
    
    def predict_hourly_usage(self, hour: int) -> int:
        """Predict human usage for a specific hour (0-23)."""
        ...
    
    def calculate_surplus(self, hour: int, total_remaining: int) -> int:
        """Calculate surplus quota available for background tasks."""
        ...
    
    def recommend_batch_window(self) -> tuple[int, int]:
        """Recommend optimal start/end hours for batch processing."""
        ...
    
    def allocate_budget(
        self, 
        total_quota: int, 
        current_hour: int
    ) -> QuotaBudget:
        """Allocate quota between human and background."""
        ...
```

---

### 3.4 SmartThrottler (`throttle.py`)

```python
"""Apply throttling rules based on quota state."""

from dataclasses import dataclass
from enum import Enum


class ThrottleLevel(str, Enum):
    """Throttle intensity levels."""
    NONE = "none"           # 100% throughput
    REDUCED = "reduced"     # 50% throughput  
    PAUSED = "paused"       # 0% throughput (queued)
    BLOCKED = "blocked"     # Rejected entirely


@dataclass
class ThrottleDecision:
    """Decision on how to handle a request."""
    level: ThrottleLevel
    delay_ms: int = 0
    reason: str = ""


class SmartThrottler:
    """Determine throttling based on quota state."""
    
    THRESHOLDS = {
        50: ThrottleLevel.NONE,     # > 50%: no throttling
        25: ThrottleLevel.REDUCED,  # 25-50%: reduced rate
        10: ThrottleLevel.PAUSED,   # 10-25%: paused
        0: ThrottleLevel.BLOCKED,   # < 10%: blocked
    }
    
    def decide(
        self, 
        priority: Priority,
        quota_remaining_pct: float
    ) -> ThrottleDecision:
        """
        Decide throttle action for a request.
        
        Rules:
        - P0 (Human): Never throttled
        - P1 (System): Only throttled if < 10%
        - P2 (Background): Throttled per thresholds
        """
        ...
    
    def should_alert(
        self, 
        quota_remaining_pct: float,
        predicted_exhaustion_hours: float
    ) -> bool:
        """Return True if an alert should be emitted."""
        ...
```

---

### 3.5 UsageStore (Internal)

```python
"""In-memory storage for usage tracking."""

from collections import defaultdict
from dataclasses import dataclass, field
from datetime import datetime


@dataclass
class UsageRecord:
    """Single usage record."""
    timestamp: datetime
    priority: Priority
    endpoint: str | None = None


class UsageStore:
    """Store and query usage history."""
    
    def __init__(self):
        self._records: list[UsageRecord] = []
        self._hourly_counts: dict[int, int] = defaultdict(int)
    
    def record(self, record: UsageRecord) -> None:
        """Record a usage event."""
        ...
    
    def get_hourly_average(self, hour: int, days: int = 7) -> float:
        """Get average usage for an hour over past N days."""
        ...
    
    def get_usage_pattern(self) -> UsagePattern:
        """Build usage pattern from stored records."""
        ...
```

---

## 4. File Structure

```
Quota-Manager/
├── src/
│   └── quota_manager/
│       ├── __init__.py         [MODIFY] Export new classes
│       ├── models.py           [EXISTING] Data models
│       ├── classifier.py       [NEW] RequestClassifier
│       ├── scheduler.py        [NEW] PriorityScheduler
│       ├── forecaster.py       [NEW] QuotaForecaster
│       ├── throttle.py         [NEW] SmartThrottler
│       ├── store.py            [NEW] UsageStore
│       └── manager.py          [MODIFY] Integrate all components
├── tests/
│   ├── test_classifier.py      [NEW]
│   ├── test_scheduler.py       [NEW]
│   ├── test_forecaster.py      [NEW]
│   ├── test_throttle.py        [NEW]
│   └── test_integration.py     [NEW]
└── docs/
    └── forecasting.md          [NEW] User documentation
```

---

## 5. Integration Points

### 5.1 QuotaManager Facade

```python
# src/quota_manager/manager.py

class QuotaManager:
    """Main entry point for quota management."""
    
    def __init__(self, total_quota: int = 10000):
        self._classifier = RequestClassifier()
        self._scheduler = PriorityScheduler()
        self._store = UsageStore()
        self._forecaster = QuotaForecaster(self._store)
        self._throttler = SmartThrottler()
        self._total_quota = total_quota
        self._used = 0
    
    def check_quota(
        self,
        headers: dict[str, str] | None = None,
        priority: Priority | None = None
    ) -> ThrottleDecision:
        """
        Check if a request should be allowed.
        
        Returns ThrottleDecision indicating whether to proceed,
        delay, or reject the request.
        """
        ...
    
    def record_usage(self, priority: Priority) -> None:
        """Record that a request was processed."""
        ...
    
    def get_budget(self) -> QuotaBudget:
        """Get current budget allocation."""
        ...
    
    def get_batch_window(self) -> tuple[int, int]:
        """Get recommended batch processing window."""
        ...
```

---

## 6. Verification Plan

### 6.1 Unit Tests

| Test File | Command | Coverage |
|-----------|---------|----------|
| `test_classifier.py` | `pytest tests/test_classifier.py -v` | Request classification logic |
| `test_scheduler.py` | `pytest tests/test_scheduler.py -v` | Priority queue ordering |
| `test_forecaster.py` | `pytest tests/test_forecaster.py -v` | Usage prediction |
| `test_throttle.py` | `pytest tests/test_throttle.py -v` | Throttling decisions |
| `test_integration.py` | `pytest tests/test_integration.py -v` | End-to-end flow |

### 6.2 All Tests

```bash
# Run all tests
pytest tests/ -v --tb=short

# With coverage
pytest tests/ --cov=src/quota_manager --cov-report=term-missing
```

### 6.3 Manual Verification

1. **Import test**: `python -c "from quota_manager import QuotaManager; print('OK')"`
2. **Type check**: `mypy src/quota_manager/`
3. **Lint check**: `ruff check src/quota_manager/`

---

## 7. Implementation Order

| Phase | Components | Est. Hours |
|-------|------------|------------|
| 1 | `classifier.py` + tests | 2h |
| 2 | `scheduler.py` + tests | 3h |
| 3 | `store.py` + `forecaster.py` + tests | 4h |
| 4 | `throttle.py` + tests | 2h |
| 5 | `manager.py` integration + tests | 3h |
| 6 | Documentation + cleanup | 2h |
| **Total** | | **16h** |

---

## 8. Risk Assessment

| Risk | Likelihood | Impact | Mitigation |
|------|------------|--------|------------|
| Historical data cold start | High | Medium | Conservative defaults in forecaster |
| Performance overhead | Low | High | Profile early, cache predictions |
| Thread safety issues | Medium | High | Document threading model, add locks if needed |

---

## 9. Approval Checklist

- [ ] Spec reviewed and approved
- [ ] Architecture approved
- [ ] Test strategy approved
- [ ] Estimates accepted
