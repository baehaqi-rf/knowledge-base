---
id: "kb-<domain>-<topic>-v1"
title: "<Concise Technical Title>"
version: "1.0.0"
status: "production-standard" # draft | production-standard | deprecated
domain: "<graphics-shaders | frontend-architecture | mobile-expo | ai-agents-workflows>"
stack:
  - "<Technology 1>"
  - "<Technology 2>"
triggers:
  - "<keyword or task intent 1>"
  - "<keyword or task intent 2>"
related: []
sources:
  - title: "<Reference Title>"
    url: "https://..."
last_updated: "YYYY-MM-DD"
---

# <Concise Technical Title>

**Scope:** One-sentence summary of what engineering problem this specification solves and what standard it enforces.

## 1. Decision Scope

### Ideal Use Cases

- **<Use Case 1>:** Concrete scenario and expected benefit.
- **<Use Case 2>:** Concrete scenario and expected benefit.

### Anti-Patterns (Do NOT Use When)

- **<Anti-Pattern 1>:** Explain the trade-off and specify the simpler alternative.
- **<Anti-Pattern 2>:** Explain the failure mode.

## 2. Reference Baseline & Bottleneck Audit

Summarize any reference prototype, video, or naive pattern being audited, and contrast it against production requirements.

| Dimension | Naive / Prototype Pattern | Production Bottleneck | Production Standard |
| :--- | :--- | :--- | :--- |
| `<Dimension>` | `<Baseline>` | `<Failure Mode / Cost>` | `<Recommended Pattern>` |

## 3. Production Architecture & Core Invariants

Include an ASCII architecture diagram, core design pillars, and mathematical or system invariants.

```text
[Input / State] -> [Processing Layer] -> [Output / Fallback]
```

## 4. Reference Implementation Patterns

Provide minimal, type-safe, production-accurate code snippets (memory layout, shaders, APIs, lifecycle hooks).

```ts
// Replace with production-grade reference snippet
```

## 5. Performance Budgets & Verification Checklist

| Metric | Production Budget | Verification Method |
| :--- | :--- | :--- |
| `<Metric>` | `<Hard Threshold>` | `<Profiler / Test>` |

### Runtime QA Verification Checklist

- [ ] `<Edge case or invariant test 1>`
- [ ] `<Fallback or accessibility test 2>`

## 6. Agent Guardrails (MUST / NEVER)

- **MUST:** `<Non-negotiable engineering rule>`
- **NEVER:** `<Prohibited anti-pattern>`

## 7. Glossary & References

- **<Term>:** Concise technical definition.
