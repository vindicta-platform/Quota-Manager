# Scheduling

## Priority Queue

```
┌─────────────────┐
│ Human Requests  │ ← Always first
├─────────────────┤
│ Background Jobs │ ← Only when surplus
└─────────────────┘
```

## 50% Reserve Rule

At all times, 50% of remaining quota is reserved for human requests.

## Background Task Example

```python
from quota_manager import BackgroundScheduler

scheduler = BackgroundScheduler()

@scheduler.task(cost=10)
async def run_batch_analysis():
    # Only runs when quota available
    pass
```
