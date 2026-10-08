# Matt Pocock Skills — Curated Engineering Workflow

A curated set of Matt Pocock's engineering skills for moving from architecture/design through specification, tickets, implementation, and bug diagnosis.

## Core Skills

| Skill | Purpose | Dependencies |
|---|---|---|
| `grill-with-docs` | Interview you rigorously about a design or problem while capturing durable domain context and architectural decisions. | `grilling`, `domain-modeling` |
| `improve-codebase-architecture` | Finds opportunities to improve the codebase's architecture and helps you work through the chosen improvement. | `codebase-design`, `domain-modeling` |
| `to-spec` | Turns a decided design into a durable implementation specification. | `setup-matt-pocock-skills` |
| `to-tickets` | Breaks a spec into small, independently implementable tickets with explicit dependencies. | `setup-matt-pocock-skills` |
| `implement-spec` | Implements an entire spec and its ticket graph, coordinating the implementation of multiple tickets. | `implement` |
| `diagnosing-bugs` | Systematically reproduces, isolates, and tests bugs before implementing a verified fix. | Standalone |

## Dependency Graph

```text
grill-with-docs
├── grilling
└── domain-modeling

improve-codebase-architecture
├── codebase-design
└── domain-modeling

to-spec
└── setup-matt-pocock-skills

to-tickets
└── setup-matt-pocock-skills

implement-spec
└── implement
    ├── tdd
    └── code-review

diagnosing-bugs
└── standalone
```

## Recommended Workflow

### Large feature / change

```text
grill-with-docs
       ↓
    to-spec
       ↓
  to-tickets
       ↓
implement-spec
       ↓
   [tickets]
       ↓
   implement
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

## Installation Set

For the full workflow, install these skills:

### Primary skills

- `grill-with-docs`
- `improve-codebase-architecture`
- `to-spec`
- `to-tickets`
- `implement-spec`
- `diagnosing-bugs`

### Required dependencies

- `grilling`
- `domain-modeling`
- `codebase-design`
- `setup-matt-pocock-skills`
- `implement`
- `tdd`
- `code-review`

## Notes

- `implement-spec` is the higher-level implementation workflow; `implement` is its underlying implementation skill.
- `diagnosing-bugs` is standalone and can be used whenever a bug needs systematic investigation.
- `to-spec` and `to-tickets` are most useful when work needs to persist across sessions or be split into multiple implementation units.
- `grill-with-docs` is useful before formalizing a significant design so that domain terminology and architectural decisions are captured first.
