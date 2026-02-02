# Getting Started

## Installation

```bash
uv pip install git+https://github.com/vindicta-platform/Quota-Manager.git
```

## Basic Usage

```python
from quota_manager import QuotaManager

qm = QuotaManager()

# Check available quota
available = qm.available_quota()

# Record usage
qm.record_usage("gemini_api", tokens=1000)

# Get prediction
surplus = qm.predict_surplus(hours=1)
```
