# Backend Development Guidelines

> Core architecture, patterns, and conventions for the opencli backend.

---

## Guidelines Index

| Guide | Description | Status |
|-------|-------------|--------|
| [Directory Structure](./directory-structure.md) | Flat core modules, pipeline/ sub-module, clis/ adapters, test co-location | Done |
| [Architecture](./architecture.md) | Module dependency graph, data flow, pipeline executor, browser abstraction layers, two CLI modes | Done |
| [Type Safety](./type-safety.md) | IPage, CliCommand, Strategy enum, StepHandler, ESM imports, intentional `any` patterns | Done |
| [Error Handling](./error-handling.md) | Top-level error boundary, graceful failures, timeout utilities, browser connection errors, debug mode | Done |
| [Testing](./testing.md) | Vitest setup, IPage mock factory, __test__ exports, per-module test patterns | Done |
| [Quality Guidelines](./quality-guidelines.md) | tsc as linter, stdout/stderr separation, chalk conventions, ESM rules, forbidden patterns | Done |

---

## Pre-Development Checklist

Before writing any backend code, read these files in order:

1. **[Directory Structure](./directory-structure.md)** -- where to put new files
2. **[Architecture](./architecture.md)** -- how modules connect and data flows
3. **[Type Safety](./type-safety.md)** -- key interfaces and import patterns
4. **[Error Handling](./error-handling.md)** -- how to handle failures
5. **[Quality Guidelines](./quality-guidelines.md)** -- style rules and forbidden patterns

If writing tests: also read **[Testing](./testing.md)**.

---

## Quick Reference

| What | Convention |
|------|-----------|
| Module layout | Flat files under `src/`, only `pipeline/` and `clis/` are sub-dirs |
| Import extensions | Always `.js` (ESM with Node16 resolution) |
| Browser abstraction | Use `IPage` interface, never concrete `Page` class |
| Error output | `process.stderr.write()` or `console.error()` |
| Data output | `console.log()` via `renderOutput()` |
| Timeouts | Use constants from `src/runtime.ts`, never hardcode |
| Tests | Co-locate `*.test.ts` next to source, use `createMockPage()` |
| Type checking | `npm run typecheck` (`tsc --noEmit`) |

---

**Language**: All documentation should be written in **English**.
