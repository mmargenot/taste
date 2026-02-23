---
name: python-testing
description: Any task involving writing, running, or debugging Python tests, or when CI lint/test failures need resolution.
---

## Toolchain

All commands run through **uv**. Never invoke `pytest` or `ruff` directly.

- `uv run pytest` — run tests
- `uv run pytest --cov=<package> --cov-report=term-missing` — with coverage
- `uv run ruff check .` — lint
- `uv run ruff format .` — format

---

## Test Layout

```
tests/
├── conftest.py          # Shared fixtures and helpers
├── unit/                # Isolated: models, schemas, functions
└── integration/         # Multi-component execution paths
```

Unit tests exercise one component in isolation. Integration tests exercise
the interaction between components (e.g., an execution loop that calls
providers, runs tools, and updates state).

---

## Mock-First Testing

Replace external dependencies (API clients, network calls) with mock
implementations that return pre-queued responses. This keeps tests
deterministic, fast, and free of credentials.

Key elements of a good mock:

- A **response queue** (FIFO list) so tests declare the exact sequence of
  responses up front.
- A **call log** that records every invocation for post-run assertions.

An `IndexError` from popping an empty queue means the system under test made
more calls than expected — fix the test setup, not the code.

---

## Pytest Fixtures

- Use **factory fixtures** when tests need flexible object construction with
  sensible defaults (e.g., `make_agent(name=..., tools=...)`).
- Use **instance fixtures** for simple, reusable objects (e.g., a fresh mock
  provider per test).
- Keep fixtures in `conftest.py` so they're shared across the test directory.

---

## Async Tests

Use **pytest-asyncio** with `asyncio_mode = "auto"` in `pyproject.toml`.
Mark each async test with `@pytest.mark.asyncio`. Synchronous unit tests
are plain functions — don't make them async unnecessarily.

For mocking async callables, use `unittest.mock.AsyncMock`.

---

## Conventions

**Naming:** files `test_<component>.py`, classes `Test<Feature>`, functions
`test_<specific_behavior>`.

**Module-level helpers** for shared test objects (e.g., decorated tool
functions). Inline definitions only when behavior is test-specific.

**Provider/client tests** use `monkeypatch` + `AsyncMock` on the underlying
SDK client, with plain-dataclass stand-ins for response shapes. Avoid
importing SDK types into tests when a simple dataclass suffices.

**State and data models** are verified after execution, never mocked.
Define custom model subclasses inline in the test module when needed.

---

## Import Hygiene (Critical)

Ruff `F401` fails CI on unused imports. The lint job runs **without `--fix`**,
so unused imports cause hard failures. When removing or rewriting a test,
always audit the import block. This has been a recurring source of CI failures.

---

## Documenting Limitations in Tests

Tests that assert known framework quirks (e.g., Pydantic v2 dropping subclass
fields during serialization) should carry a docstring explaining the
limitation. These serve as executable documentation — a future upgrade that
changes the behavior will surface through a test failure rather than silently
breaking assumptions elsewhere.

---

## CI

Both jobs run on every PR and must pass:

```
lint:  uv run ruff check .
test:  uv run pytest tests/ -v
```
