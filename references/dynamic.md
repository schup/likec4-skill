# Dynamic Views

**Goal**: Show a specific runtime flow, use case, or sequence of interactions
across elements in your architecture. Dynamic views let you walk through
a scenario step-by-step, rather than showing a static structural picture.
Audience: developers, architects, testers explaining a behaviour.

Dynamic views are defined with the `dynamic view` keyword.
Steps are numbered implicitly in the order they are written.

---

## Basic Pattern

```c4
dynamic view checkoutFlow 'Checkout Flow' {
  title 'Order Checkout — Happy Path'
  description 'Sequence of interactions when a customer places an order'

  // Steps are listed in order; LikeC4 numbers them automatically
  customer    -> spa         'submits order'
  spa         -> api         'POST /orders'
  api         -> auth        'validates JWT'
  api         -> pricingService 'calculates total'
  api         -> orderService   'creates order'
  orderService -> db         'persists order'
  orderService -> mq         'publishes OrderCreated event'
  mq          -> jobs        'delivers event'
  jobs        -> emailProvider  'sends confirmation email'
  api         -> spa         'returns order ID'
}
```

---

## Parallel Steps

Use `parallel {}` to group steps that happen concurrently:

```c4
dynamic view orderProcessing 'Order Processing' {
  title 'Order Processing — With Parallel Async Steps'

  customer -> api 'places order'
  api      -> db  'saves order'

  parallel {
    api -> mq         'publishes OrderCreated'
    api -> auditLog   'writes audit entry'
  }

  mq  -> emailWorker 'triggers email'
  mq  -> inventoryWorker 'triggers inventory reservation'
}
```

---

## Full Example: User Login Flow

```c4
// Logical model (abbreviated)
model {
  person   customer 'Customer'
  system   myPlatform 'My Platform' {
    webApp    spa   'SPA'
    apiService api  'API Gateway'
    apiService auth 'Auth Service'
    database  db    'User DB'
    cache     redis 'Session Cache'
  }
}

// Dynamic view
views {
  dynamic view loginFlow 'Login Flow' {
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

  dynamic view loginFail 'Login Failure Flow' {
    title 'Customer Login — Invalid Credentials'

    customer -> spa   'enters wrong password'
    spa      -> api   'POST /auth/login'
    api      -> auth  'validates credentials'
    auth     -> db    'looks up user record'
    db       -> auth  'returns user record'
    auth     -> api   'returns 401 Unauthorized'
    api      -> spa   'returns error'
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
  spa  -> api   'sends expired token'        // scenario-specific label
  api  -> auth  'requests token validation'
  auth -> api   'confirms expiry, issues new token'
  api  -> redis 'stores new token'
  api  -> spa   'returns new token'
}
```

---

## Agent Checklist — Dynamic View

- [ ] Uses `dynamic view` keyword (not plain `view`)
- [ ] Steps are in strict chronological order
- [ ] Each step label describes *what happens in this scenario*, not a generic description
- [ ] `parallel {}` used for genuinely concurrent steps
- [ ] The view has a `title` (what is this flow?) and `description` (what scenario / preconditions)
- [ ] Error / alternate flows modelled as separate dynamic views
- [ ] All referenced elements exist in the `model` block
