# Curated Engineering Workflow

A curated set of Matt Pocock's engineering skills for moving from architecture/design through specification, tickets, implementation, handoffs, and bug diagnosis.

## Core Skills

| Skill | Purpose |
|---|---|
| `grill-with-docs` | Interview you rigorously about a design or problem while capturing durable domain context and architectural decisions. |
| `improve-codebase-architecture` | Finds opportunities to improve the codebase's architecture and helps you work through the chosen improvement. |
| `to-spec` | Turns a decided design into a durable implementation specification. |
| `to-tickets` | Breaks a spec into small, independently implementable tickets with explicit dependencies. |
| `implement-spec` | Implements an entire spec and its ticket graph, coordinating the implementation of multiple tickets. |
| `diagnosing-bugs` | Systematically reproduces, isolates, and tests bugs before implementing a verified fix. |
| `wait-what` | Challenges assumptions and clarifies confusing or underspecified requirements before work proceeds. |
| `handoff` | Creates a concise, durable handoff so another agent or session can continue work with the necessary context. |

## Dependency Graph

```text
grill-with-docs
├── grilling
└── domain-modeling

improve-codebase-architecture
├── codebase-design
└── domain-modeling

to-spec
└── standalone

to-tickets
└── standalone

implement-spec
└── implement
    ├── tdd
    └── code-review

diagnosing-bugs
└── standalone

wait-what
└── standalone

handoff
└── writing-for-agent

writing-for-agent
└── standalone
```

## Recommended Workflows

### Large feature / change

```text
grill-with-docs
       ↓
    to-spec
       ↓
  to-tickets
       ↓
implement-spec
```

### Architecture improvement

```text
improve-codebase-architecture
       ↓
   choose opportunity
       ↓
grill-with-docs / to-spec
       ↓
  to-tickets
       ↓
implement-spec
```

### Bug

```text
diagnosing-bugs
       ↓
  identify root cause
       ↓
  implement fix
```

### Agent handoff

```text
writing-for-agent
       ↓
     handoff
       ↓
another agent / session
```

### Unclear requirements / assumptions

```text
wait-what
       ↓
clarify assumptions
```

## Installation Set

For the full workflow, install these skills:

### Primary skills

- `grill-with-docs`
- `improve-codebase-architecture`
- `to-spec`
- `to-tickets`
- `implement-spec`
- `diagnosing-bugs`
- `wait-what`
- `handoff`

### Required dependencies

- `grilling`
- `domain-modeling`
- `codebase-design`
- `setup-matt-pocock-skills`
- `implement`
- `tdd`
- `code-review`
- `writing-for-agent`

## Notes

- Instruct `to-spec` and `to-tickets` to create local .md files instead of reaching out to external issue trackers.
- `grill-with-docs` is useful before formalizing a significant design so that domain terminology and architectural decisions are captured first.
- `wait-what` is useful as an early clarification step when requirements, assumptions, or proposed approaches are unclear.
- `handoff` builds on that agent-oriented writing to transfer work between sessions or agents.
