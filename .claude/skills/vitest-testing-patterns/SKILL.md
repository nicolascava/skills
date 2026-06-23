---
name: vitest-testing-patterns
description: Write tests using Vitest. Use when creating unit tests, integration tests, or setting up coverage. Activates for test file creation, mock patterns, coverage, and testing best practices.
allowed-tools: Read,Write,Edit,Bash(npm:*,npx:*,pnpm:*,bun:*)
metadata:
  category: Code Quality & Testing
  tags:
  - testing
  - code
  - automation
  - vitest
  pairs-with:
  - skill: playwright-e2e-tester
    reason: Unit tests (Vitest) and E2E tests (Playwright) form complementary test pyramid layers
  - skill: react-performance-optimizer
    reason: Component test patterns verify that performance optimizations preserve correct behavior
  - skill: typescript-advanced-patterns
    reason: Type-safe test utilities and mock factories leverage advanced TypeScript patterns
---

# Vitest Testing Patterns

This skill helps you write effective tests using Vitest following the project's canonical testing stack.

## Canonical stack

| Layer | Tool |
|-------|------|
| Unit/integration tests | `vitest` + `@vitest/coverage-v8` |
| E2E tests | `@playwright/test` |
| Accessibility audits | `@axe-core/playwright` |
| Bun-runtime packages (exception) | `bun test` + lcov coverage |

See `docs/templates/testing-standards.md` for full setup instructions and the Bun-runtime exception rationale.

## When to Use

**USE this skill for:**
- Writing unit tests for utilities and functions
- Creating integration tests for API handlers and business logic
- Setting up vitest coverage (v8 provider, json-summary reporter)
- Mocking modules and functions with `vi.*`

**Do NOT use for:**
- End-to-end testing: use Playwright (see `docs/templates/testing-standards.md`)
- Packages that depend on `Bun.*` APIs: keep those on `bun test`

## Test infrastructure

**Shared coverage config**: `vitest.shared.ts` (repo root)

**Per-package config**: `vitest.config.ts` merging `sharedCoverage`

**Commands**:

```bash
pnpm test           # run once
pnpm test:coverage  # run with v8 coverage → coverage/coverage-summary.json
```

## File organization

```
src/
├── lib/feature.ts
├── lib/feature.test.ts       # colocated unit test
└── handlers/route.test.ts    # integration test
```

Name files `*.test.ts` (unit/integration) or `*.spec.ts` (E2E).

## Core patterns

### Unit test: pure function

```typescript
import { describe, expect, test } from "vitest";
import { processData } from "./utils.ts";

describe("processData", () => {
  test("transforms input correctly", () => {
    expect(processData({ raw: "data" })).toEqual({ processed: true, data: "DATA" });
  });

  test("throws on null input", () => {
    expect(() => processData(null)).toThrow("Invalid input");
  });
});
```

### Integration test: API route (Node/Next.js)

```typescript
import { describe, expect, test, vi, beforeEach } from "vitest";
import { GET } from "./route.ts";
import { NextRequest } from "next/server";

vi.mock("@/lib/auth", () => ({ getSession: vi.fn() }));

describe("GET /api/feature", () => {
  beforeEach(() => { vi.clearAllMocks(); });

  test("returns 401 when unauthenticated", async () => {
    vi.mocked(getSession).mockResolvedValue(null);
    const res = await GET(new NextRequest("http://localhost/api/feature"));
    expect(res.status).toBe(401);
  });

  test("returns data when authenticated", async () => {
    vi.mocked(getSession).mockResolvedValue({ userId: "user-123" });
    const res = await GET(new NextRequest("http://localhost/api/feature"));
    expect(res.status).toBe(200);
  });
});
```

### React component test (string rendering, no jsdom required)

`apps/web` component tests use `react-dom/server` for string rendering
instead of a jsdom environment, which keeps them fast and dependency-free.

```typescript
import { describe, expect, test } from "vitest";
import { renderToString } from "react-dom/server";
import { FeatureCard } from "./FeatureCard.tsx";

describe("FeatureCard", () => {
  test("renders title", () => {
    const html = renderToString(<FeatureCard title="Hello" />);
    expect(html).toContain("Hello");
  });
});
```

## Mocking patterns

### Module mock

```typescript
vi.mock("@/lib/auth", () => ({
  getSession: vi.fn(),
  requireAuth: vi.fn(),
}));

// Partial mock: keep real exports, override one
vi.mock("date-fns", async () => {
  const actual = await vi.importActual("date-fns");
  return { ...actual, format: vi.fn(() => "2026-01-15") };
});
```

### Function mock

```typescript
const mockFn = vi.fn();

mockFn.mockReturnValue("sync");
mockFn.mockResolvedValue("async");
mockFn.mockRejectedValue(new Error("failed"));
mockFn.mockReturnValueOnce("first call only");
mockFn.mockImplementation(arg => arg.toUpperCase());

expect(mockFn).toHaveBeenCalledWith("expected");
expect(mockFn).toHaveBeenCalledTimes(2);
```

### Timer mock

```typescript
beforeEach(() => { vi.useFakeTimers(); });
afterEach(() => { vi.useRealTimers(); });

test("debounces calls", () => {
  const callback = vi.fn();
  const debounced = debounce(callback, 300);
  debounced(); debounced(); debounced();
  expect(callback).not.toHaveBeenCalled();
  vi.advanceTimersByTime(300);
  expect(callback).toHaveBeenCalledTimes(1);
});
```

## Coverage

Coverage is emitted by `@vitest/coverage-v8` into `coverage/coverage-summary.json`
(json-summary) and `coverage/` (json). The root script `pnpm coverage:readme` merges
vitest and bun-test lcov artifacts into a shields.io badge + table in `README.md`.

```bash
pnpm coverage          # run all coverage (vitest + bun test lcov)
pnpm coverage:readme   # regenerate README.md coverage block
```

## References

- [Vitest documentation](https://vitest.dev)
- [Vitest mocking guide](https://vitest.dev/guide/mocking)
- [Testing standards template](../../docs/templates/testing-standards.md)
