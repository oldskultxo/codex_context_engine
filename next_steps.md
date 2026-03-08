# Next Steps — codex_context Roadmap

This document describes the **next planned iterations** for the `codex_context` system.
The goal of these iterations is to evolve the project beyond basic context storage
into a **full context orchestration system for AI coding agents**.

The current version (Iteration Four) already includes:

- persistent project memory
- deterministic context packets
- relevance scoring
- memory compaction
- telemetry
- global metrics and diagnostics

The following iterations expand the system with more intelligent context management.

---

# Iteration 5 — Context Planner

## Goal

Introduce a **context planning layer** that determines what context will be needed
*before* Codex begins executing a task.

Currently the system retrieves context reactively.  
Iteration Five adds a planning step that predicts what context will be useful.

## Concept

```
User Task
    |
    v
Context Planner
    |
    v
Context Retrieval
    |
    v
Deterministic Context Packet
    |
    v
Codex Execution
```

## Capabilities

The context planner may:

- identify relevant repository areas
- determine which memory categories to query
- predict which files are likely to be involved
- pre-build the context packet before reasoning starts

## Benefits

- reduced unnecessary context
- faster reasoning
- improved performance on large repositories

---

# Iteration 6 — Task-Specific Memory

## Goal

Introduce memory specialized by **type of task**.

Different tasks require different knowledge patterns.

Example categories:

```
.codex_task_memory/

bug_fixing/
refactoring/
tests/
performance/
architecture/
```

## Example Memory Entry

```
task_type: bug_fixing
common_locations:
- logging module
- validation layer

common_mistakes:
- missing null checks
- inconsistent error propagation
```

## Benefits

- more relevant context retrieval
- better task specialization
- improved debugging and refactoring workflows

---

# Iteration 7 — Failure Memory

## Goal

Add a system for **recording previously encountered failures and their solutions**.

Many debugging tasks repeat similar mistakes.
Failure memory helps avoid repeating them.

## Example Record

```
error_type: build_failure
root_cause: incorrect import path
solution: update module reference

files_involved:
- build_config
- module_loader
```

## Benefits

- faster debugging
- avoids repeated mistakes
- creates a repository-specific troubleshooting knowledge base

---

# Iteration 8 — Context Cost Optimizer

## Goal

Optimize context **before it is injected into the model**.

The optimizer evaluates the estimated token cost of the context packet
and reduces it if necessary.

## Example

```
estimated_context_size = 8000 tokens
budget = 3000 tokens
```

Optimizer actions:

- drop low-score memory entries
- compress summaries
- remove redundant knowledge
- prioritize high-value records

## Benefits

- lower token usage
- faster responses
- more predictable model behavior

---

# Iteration 9 — Memory Graph

## Goal

Replace simple memory lists with a **graph structure of knowledge**.

This allows relationships between memory entries to be captured.

## Example Graph

```
Architecture Decision
     |
     +--- related module
     |
     +--- known bug pattern
     |
     +--- previous refactor
```

## Benefits

- richer knowledge retrieval
- contextual reasoning across components
- improved architecture awareness

---

# Long-Term Vision

The ultimate goal is for `codex_context` to evolve from a simple
external memory layer into a **context operating system for AI agents**.

Future capabilities may include:

- adaptive context retrieval
- reinforcement learning from context usage
- automated context compression
- cross-project knowledge graphs
- explainable context decisions

---

# Summary

Planned iterations:

| Iteration | Feature |
|-----------|--------|
| 5 | Context Planner |
| 6 | Task-Specific Memory |
| 7 | Failure Memory |
| 8 | Context Cost Optimizer |
| 9 | Memory Graph |

These steps progressively transform the system from **context storage**
into **intelligent context orchestration**.
