---
name: likec4
description: >
  Generate, scaffold, and refine LikeC4 architecture diagrams using the LikeC4 DSL (.c4 / .likec4 files).
  Use this skill whenever the user wants to: create architecture diagrams, model software systems,
  generate C4-style views (context, container, component, deployment), produce LikeC4 DSL code,
  describe a system and turn it into a diagram, or ask about element kinds, views, relationships,
  deployment nodes, trust/network boundaries, or connection detail (protocols, ports, hostnames,
  service accounts, per-view detail levels) in LikeC4. Trigger even if the user just says
  "make a diagram of my system", "model this architecture", "generate a C4 diagram",
  "show me the containers in my app", "add trust boundaries", "show network zones",
  "add connection detail", "show protocol and port on the diagram", or "different views for different audiences".
---

# LikeC4 Skill

LikeC4 is a DSL for describing and visualising software architecture. It is inspired by the C4 model
and Structurizr DSL, but lets you define your own notation, element kinds, and any number of nesting levels.

Source files use `.c4` or `.likec4` extensions. All files in a project are merged into one model.

---

## Agent Workflow

When a user asks you to diagram a system, follow this sequence:

1. **Understand scope** — Ask (or infer from context): what system, what audience, what level of detail?
2. **Pick the right view type(s)** — Use the table below to match intent → view type.
3. **Read the reference file** for the chosen view type before writing DSL.
4. **Scaffold the model** — `specification` → `model` → `views` (in that order in the file).
5. **Iterate** — Offer to drill down, add deployment, or generate additional views.

### View Type Selection Guide

| User intent                                               | View type           | Reference file                    |
|-----------------------------------------------------------|---------------------|-----------------------------------|
| "Show the big picture", "who uses what"                   | System Context      | `references/context.md`           |
| "Show containers / services / APIs"                       | Container Map       | `references/container.md`         |
| "Show internals of one service / component breakdown"     | Component View      | `references/component.md`         |
| "Show infra, Kubernetes, cloud, environments"             | Deployment View     | `references/deployment.md`        |
| "Sequence / flow / walk through a use case"               | Dynamic View        | `references/dynamic.md`           |
| "Add trust boundaries", "show network zones / segments",  | Trust Boundaries    | `references/trust-boundary.md`    |
| "Add connection detail", "show protocol/port/auth",       | + Connection Detail | `references/trust-boundary.md`    |
| "Different views for different audiences"                 |                     |                                   |

> Always read the relevant reference file(s) before generating DSL. Multiple reference files may apply —
> for example, a context diagram that also shows network zones needs both `context.md` AND `trust-boundary.md`.

---

## DSL Skeleton

Every LikeC4 project has three top-level blocks. Scaffold them in this order:

```c4
// 1. Define your notation
specification {
  element actor
  element system
  element container
  element component
  element database
  relationship async
  tag deprecated
  tag external
}

// 2. Define the model (elements + relationships)
model {
  actor customer 'Customer' 'End user of the platform'

  system mySystem 'My System' {
    container api    'API Gateway'  { technology 'REST/HTTP' }
    container webapp 'Web App'      { technology 'React'     }
    database  db     'Primary DB'   { technology 'PostgreSQL' }

    api -> db     'reads/writes'
    webapp -> api 'calls'
  }

  customer -> mySystem.webapp 'uses'
}

// 3. Define views
views {
  view index {
    title 'System Landscape'
    include *
  }
}
```

---

## Core DSL Rules (memorise these)

- **Names**: alphanumeric + hyphens/underscores, cannot start with digit, no dots.
- **References**: use dot-notation — `mySystem.api`. Inside a scoped view (`view of X`) you can use short names.
- **Tags must come first** inside an element block, before any other properties.
- **Properties order inside element**: `#tags`, `title`, `description`, `technology`, `links`, `metadata`, then child elements.
- **Relationships**: `source -> target 'label'` or `source -[kind]-> target 'label'`
- **All files merge** into one model — split large models into `spec.c4`, `model.c4`, `views.c4`.
- **`index` view** is the default landing view; always define one.

---

## Styling Quickref

```c4
specification {
  element database {
    style {
      shape cylinder
      color amber
    }
  }
  element actor {
    style {
      shape person
      color green
    }
  }
}

// Per-element override in the model:
model {
  database legacyDB 'Legacy DB' {
    style { color red }
  }
}

// Per-view style overrides:
views {
  view myView {
    include *
    style customer {
      color green
      shape person
    }
    style *.external {
      color gray
      opacity 40%
    }
  }
}
```

**Available shapes**: `rectangle` (default), `cylinder`, `queue`, `ellipse`, `person`, `hexagon`, `storage`, `browser`, `mobile`, `component`

**Built-in colors**: `primary`, `secondary`, `muted`, `slate`, `red`, `green`, `amber`, `blue`, `indigo`, `sky`, `teal`

**Built-in icon sets**: `aws:`, `azure:`, `gcp:`, `tech:` — e.g. `icon aws:simple-storage-service`

---

## File Layout Convention

For any non-trivial project, split files like this:

```
architecture/
├── spec.c4          # specification block — element kinds, relationship kinds, tags, colors
├── model.c4         # model block — elements and relationships
├── views.c4         # views block — all view definitions
└── deployment.c4    # deployment block — infra nodes and deployment views (if needed)
```

---

## Validating the Diagram

After generating or editing `.c4` / `.likec4` files, always validate the model before presenting it to the user:

```bash
likec4 validate [path]
```

- `[path]` is the directory containing your `.c4` files (defaults to current directory if omitted).
- A clean run prints no errors and exits with code 0.
- **Fix all reported errors before proceeding.** Common issues: missing element references, duplicate IDs, syntax errors, undefined relationship kinds.
- Run validate after every non-trivial edit, not just at the end.

Example:
```bash
likec4 validate ./architecture
```

---

## Previewing the Diagram

To let the user interactively preview the rendered diagram in a browser:

```bash
likec4 start [path]          # aliases: serve, dev
```

- Starts a local dev server with hot-reload — changes to `.c4` files are reflected instantly.
- `[path]` is the directory containing your `.c4` files (defaults to current directory).
- Opens at `http://localhost:5173` by default (port may vary — check the terminal output).
- Tell the user to open that URL in their browser to navigate and inspect all views.

For a production-quality static preview (no hot-reload):

```bash
likec4 build [path]          # build the static site
likec4 preview [path]        # serve the built site locally
```

Use `likec4 start` during iterative development; use `build` + `preview` to verify the final output before sharing.

---

## Reference Files

Read these before generating DSL for the corresponding diagram type:

- `references/context.md`        — System Context view (C4 Level 1)
- `references/container.md`      — Container / Service Map view (C4 Level 2)
- `references/component.md`      — Component internals view (C4 Level 3)
- `references/deployment.md`     — Deployment / infrastructure view
- `references/dynamic.md`        — Dynamic / sequence / flow views
- `references/trust-boundary.md` — Trust / network boundary zones AND connection detail
                                   (protocols, ports, hostnames, service accounts, per-view levels)
