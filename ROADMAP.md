# Quota-Manager Roadmap

> **Vision**: API quota tracking and management
> **Status**: Merged into Agent-Auditor-SDK
> **Last Updated**: 2026-02-03

---

## Current Status: Deprecated

Quota-Manager functionality has been **merged into Agent-Auditor-SDK** as the canonical quota management solution.

---

## Migration Path

All quota management functionality should use:

```python
# Instead of:
from quota_manager import QuotaManager

# Use:
from agent_auditor_sdk import ArbiterScheduler, QuotaPredictor
```

---

## See Also

- [Agent-Auditor-SDK ROADMAP](../Agent-Auditor-SDK/ROADMAP.md)

---

*This package is deprecated. Use Agent-Auditor-SDK.*
