# 0003. One S3 kit object, one registry of model handlers

Date: 2026-11-16 · Status: accepted (plan §5.1–5.2)

## Decision
- Every function creates, uses or checks a `reprokit_kit`.
- A kit cannot exist without a stated reason for not sharing and a non-empty "what is lost"
  list (the validator refuses). This is how the tool avoids becoming an excuse not to share.
- Each analysis type is a handler in a registry; the README coverage table is generated from it.

## Consequences
- Adding a model = adding one handler file + its round-trip tests.
- Documentation and code cannot contradict each other about coverage.
