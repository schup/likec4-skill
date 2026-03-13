# Dynamic Views

**Goal**: Show a specific runtime flow, use case, or sequence of interactions
across elements in your architecture. Dynamic views let you walk through
a scenario step-by-step, rather than showing a static structural picture.
Audience: developers, architects, testers explaining a behaviour.

Dynamic views are defined with the `dynamic view` keyword.
Steps are numbered implicitly in the order they are written.

---

## ⚠️ Known Gotchas (read before writing any DSL)

### 1. No inline title on `dynamic view` declaration
**WRONG** — parser error:
```c4
dynamic view myFlow 'My Flow Title' {   // ← quoted title on declaration line is INVALID
```
**CORRECT** — title goes inside as a property:
```c4
dynamic view myFlow {
  title 'My Flow Title'
```

### 2. `hexagon` is not a valid shape
The token list does not include `hexagon`. Use `component` instead for transformer/processor style elements.

| Intent                        | Use this shape  |
|-------------------------------|-----------------|
| transformer, processor, step  | `component`     |
| database, store               | `cylinder`      |
| message queue / bus           | `queue`         |
| storage bucket                | `storage`       |
| person / user                 | `person`        |
| web frontend                  | `browser`       |
| mobile app                    | `mobile`        |
| generic service / system      | `rectangle`     |

### 3. `teal` is not a valid built-in color
Valid built-in color tokens: `primary`, `secondary`, `muted`, `slate`, `red`, `green`, `amber`, `blue`, `indigo`, `sky`.  
`teal` will cause a "Could not resolve reference to CustomColor" error — use `secondary` or `sky` instead.

---

## Basic Pattern

```c4
dynamic view checkoutFlow {
  title 'Order Checkout — Happy Path'
  description 'Sequence of interactions when a customer places an order'

  // Steps are listed in order; LikeC4 numbers them automatically
  customer     -> spa            'submits order'
  spa          -> api            'POST /orders'
  api          -> auth           'validates JWT'
  api          -> pricingService 'calculates total'
  api          -> orderService   'creates order'
  orderService -> db             'persists order'
  orderService -> mq             'publishes OrderCreated event'
  mq           -> jobs           'delivers event'
  jobs         -> emailProvider  'sends confirmation email'
  api          -> spa            'returns order ID'
}
```

---

## Parallel Steps

Use `parallel {}` to group steps that happen concurrently:

```c4
dynamic view orderProcessing {
  title 'Order Processing — With Parallel Async Steps'

  customer -> api 'places order'
  api      -> db  'saves order'

  parallel {
    api -> mq       'publishes OrderCreated'
    api -> auditLog 'writes audit entry'
  }

  mq -> emailWorker     'triggers email'
  mq -> inventoryWorker 'triggers inventory reservation'
}
```

---

## Full Example: User Login Flow

```c4
// Logical model (abbreviated)
model {
  actor    customer 'Customer'
  system   myPlatform 'My Platform' {
    service   spa   'SPA'       { technology 'React' }
    service   api   'API GW'    { technology 'REST'  }
    service   auth  'Auth'      { technology 'OAuth2' }
    database  db    'User DB'   { technology 'PostgreSQL' }
    service   redis 'Session'   { technology 'Redis' }
  }
}

// Dynamic view — note: NO quoted title on the declaration line
views {
  dynamic view loginFlow {
    title 'Customer Login — Success Path'
    description '''
      Describes the sequence of calls when a customer successfully
      authenticates using email/password.
    '''

    customer -> spa   'enters credentials'
    spa      -> api   'POST /auth/login'
    api      -> auth  'validates credentials'
    auth     -> db    'looks up user record'
    db       -> auth  'returns user record'
    auth     -> redis 'creates session token'
    auth     -> api   'returns JWT'
    api      -> spa   'returns JWT + user profile'
    spa      -> customer 'displays dashboard'
  }

  dynamic view loginFail {
    title 'Customer Login — Invalid Credentials'

    customer -> spa  'enters wrong password'
    spa      -> api  'POST /auth/login'
    api      -> auth 'validates credentials'
    auth     -> db   'looks up user record'
    db       -> auth 'returns user record'
    auth     -> api  'returns 401 Unauthorized'
    api      -> spa  'returns error'
    spa      -> customer 'shows error message'
  }
}
```

---

## Styling Dynamic Views

You can apply styles within dynamic views just like in structural views:

```c4
dynamic view checkoutFlow {
  title 'Checkout Flow'

  customer -> spa -> api -> orderService -> db

  style customer {
    color green
    shape person
  }
  style db {
    color amber
    shape cylinder
  }
  style orderService {
    color primary
  }
}
```

---

## Relationship Labels in Dynamic Views

Unlike structural views, labels in dynamic views describe *what happens in this step*, not a general relationship:

```c4
dynamic view tokenRefresh {
  title 'Token Refresh Flow'

  // Labels are specific to this scenario
  spa  -> api   'sends expired token'
  api  -> auth  'requests token validation'
  auth -> api   'confirms expiry, issues new token'
  api  -> redis 'stores new token'
  api  -> spa   'returns new token'
}
```

---

## Agent Checklist — Dynamic View

- [ ] Uses `dynamic view` keyword (not plain `view`)
- [ ] Declaration line is `dynamic view <id> {` — **no quoted title on the same line**
- [ ] `title` is a property *inside* the block
- [ ] Steps are in strict chronological order
- [ ] Each step label describes *what happens in this scenario*, not a generic description
- [ ] `parallel {}` used for genuinely concurrent steps
- [ ] The view has a `title` and optionally a `description`
- [ ] Error / alternate flows modelled as separate dynamic views
- [ ] All referenced elements exist in the `model` block
- [ ] No `hexagon` shape used — use `component` instead
- [ ] No `teal` color used — use `secondary` or `sky` instead
