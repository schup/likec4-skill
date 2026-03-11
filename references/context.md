# System Context View (C4 Level 1)

**Goal**: Show the system being built and how it relates to users and other external systems.
Audience: non-technical stakeholders, product owners.

---

## Specification Pattern

```c4
specification {
  element person          // human actors (shape: person)
  element externalSystem  // third-party systems outside your control
  element system          // your own software systems
}
```

---

## Model Pattern

```c4
model {
  // External actors
  person customer       'Customer'        'End-user of the platform'
  person admin          'Administrator'   'Internal operations staff'

  // External systems (not owned by you)
  externalSystem paymentGateway 'Payment Gateway' {
    #external
    description 'Stripe payment processing'
  }
  externalSystem emailProvider 'Email Provider' {
    #external
    description 'SendGrid transactional email'
  }

  // Your system — keep opaque at this level
  system myPlatform 'My Platform' 'Core e-commerce platform'

  // Relationships — use plain language labels
  customer       -> myPlatform      'browses and purchases'
  admin          -> myPlatform      'manages catalogue and orders'
  myPlatform     -> paymentGateway  'processes payments via'
  myPlatform     -> emailProvider   'sends notifications via'
}
```

---

## View Pattern

```c4
views {
  view index {
    title 'System Context — My Platform'
    description 'Who uses My Platform and what external systems does it depend on?'

    include *                  // show everything at top level
    // or be explicit:
    // include customer, admin, myPlatform, paymentGateway, emailProvider
  }
}
```

---

## Styling Conventions

```c4
specification {
  element person {
    style { shape person; color green }
  }
  element externalSystem {
    style { color gray; opacity 60% }
  }
  element system {
    style { color primary }
  }
  tag external {
    color gray
  }
}
```

---

## Agent Checklist — Context View

- [ ] One `system` element represents the system-under-focus (don't expand internals here)
- [ ] All human actors use `shape person`
- [ ] External systems tagged `#external` and styled differently (muted/gray)
- [ ] Relationships use plain-English labels describing *what* flows, not *how*
- [ ] `index` view includes all top-level elements
- [ ] View has a `title` and `description` stating the question it answers
