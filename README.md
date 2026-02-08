# Quota-Manager

Usage tracking, quota prediction, and budget allocation.

## Overview

Quota-Manager tracks API usage against quotas and predicts hourly surplus for background task scheduling.

## Features

- **Usage Tracking**: Record API calls per endpoint
- **Quota Prediction**: Forecast hourly surplus
- **Budget Allocation**: Reserve quota for human requests

## Installation

```bash
uv pip install git+https://github.com/vindicta-platform/Quota-Manager.git
```

## Platform Documentation

> **📌 Important:** All cross-cutting decisions, feature proposals, and platform-wide architecture documentation live in [**Platform-Docs**](https://github.com/vindicta-platform/Platform-Docs).
>
> Any decision affecting multiple repos **must** be recorded there before implementation.

- 📋 [Feature Proposals](https://github.com/vindicta-platform/Platform-Docs/tree/main/docs/proposals)
- 🏗️ [Architecture Decisions](https://github.com/vindicta-platform/Platform-Docs/tree/main/docs)
- 📖 [Contributing Guide](https://github.com/vindicta-platform/Platform-Docs/blob/main/CONTRIBUTING.md)

## License

MIT License - See [LICENSE](./LICENSE) for details.
