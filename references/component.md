# Component View (C4 Level 3)

**Goal**: Zoom into a single container and show its major internal components —
classes, modules, controllers, services, repositories, handlers, etc.
Audience: developers working on or integrating with that container.

---

## Specification Pattern

```c4
specification {
  // Reuse higher-level kinds where useful, add component-level kinds:
  element controller    { style { shape component } }
  element service       { style { shape component } }
  element repository    { style { shape component } }
  element handler       { style { shape component } }
  element middleware    { style { shape component } }
  element facade        { style { shape component } }
  element eventConsumer { style { shape component } }
  element eventProducer { style { shape component } }

  // Relationships
  relationship calls
  relationship delegates
  relationship subscribes  { style { line dashed } }
  relationship publishes   { style { line dashed } }
}
```

---

## Model Pattern: REST API Service

```c4
model {
  // Parent system/container context (abbreviated)
  system myPlatform 'My Platform' {
    apiService api 'API Service' {
      // ── Inbound layer ──────────────────────────────────────
      middleware  authMiddleware   'Auth Middleware'   { technology 'JWT validation' }
      middleware  rateLimiter      'Rate Limiter'      { technology 'Redis leaky bucket' }

      // ── Controllers ────────────────────────────────────────
      controller  orderController  'Order Controller'  { technology 'Express router' }
      controller  userController   'User Controller'   { technology 'Express router' }

      // ── Domain services ────────────────────────────────────
      service     orderService     'Order Service'     { technology 'TypeScript class' }
      service     userService      'User Service'      { technology 'TypeScript class' }
      service     pricingService   'Pricing Service'   { technology 'TypeScript class' }
      service     notifyService    'Notification Svc'  { technology 'TypeScript class' }

      // ── Data access ────────────────────────────────────────
      repository  orderRepo        'Order Repository'  { technology 'TypeORM' }
      repository  userRepo         'User Repository'   { technology 'TypeORM' }

      // ── Event producers ────────────────────────────────────
      eventProducer orderEvents    'Order Events'      { technology 'RabbitMQ publisher' }

      // ── Internal wiring ────────────────────────────────────
      authMiddleware   -> orderController  'passes request to' calls
      authMiddleware   -> userController   'passes request to' calls
      orderController  -> orderService     'delegates to'      delegates
      userController   -> userService      'delegates to'      delegates
      orderService     -> pricingService   'calculates price via' calls
      orderService     -> orderRepo        'persists via'      calls
      orderService     -> notifyService    'triggers notify'   calls
      orderService     -> orderEvents      'publishes event'   publishes
      userService      -> userRepo         'reads/writes via'  calls
    }

    // External containers referenced from inside api
    database db    'Primary Database' { technology 'PostgreSQL' }
    queue    mq    'Message Queue'    { technology 'RabbitMQ'   }

    api.orderRepo -> db 'queries'
    api.userRepo  -> db 'queries'
    api.orderEvents -> mq 'publishes to'
  }
}
```

---

## View Patterns

### Full component view of one container

```c4
views {
  view apiComponents of myPlatform.api {
    title 'API Service — Component View'
    description 'Internal structure of the API Service container'

    include *                                // expand all components inside api
    include myPlatform.db                    // show external DB it talks to
    include myPlatform.mq                    // show external queue
  }
}
```

### Focus on one flow (e.g., order placement)

```c4
views {
  view orderFlow of myPlatform.api {
    title 'Order Placement Flow'

    include orderController
    include orderController ->               // everything orderController calls
    include -> orderController               // everything that calls orderController
    include orderService ->                  // downstream from orderService
  }
}
```

### Layered view with grouping

```c4
views {
  view apiLayers of myPlatform.api {
    title 'API Service — Layers'
    include *

    // Group by layer using style
    style authMiddleware, rateLimiter {
      color sky
      notation 'Middleware'
    }
    style orderController, userController {
      color primary
      notation 'Controllers'
    }
    style orderService, userService, pricingService, notifyService {
      color green
      notation 'Domain Services'
    }
    style orderRepo, userRepo {
      color amber
      notation 'Repositories'
    }
  }
}
```

---

## Common Architectural Patterns to Model

### Hexagonal / Ports & Adapters

```c4
model {
  apiService api 'API' {
    // Driving ports (inbound)
    controller httpAdapter   'HTTP Adapter'
    // Domain core
    service    domainService 'Domain Service'
    // Driven ports (outbound)
    repository dbAdapter     'DB Adapter'
    facade     emailAdapter  'Email Adapter'

    httpAdapter   -> domainService 'drives'
    domainService -> dbAdapter     'persists via'
    domainService -> emailAdapter  'notifies via'
  }
}
```

### CQRS

```c4
model {
  apiService api 'API' {
    handler commandHandler 'Command Handler' { #write-side }
    handler queryHandler   'Query Handler'   { #read-side  }
    service commandBus     'Command Bus'
    service queryBus       'Query Bus'
    repository writeStore  'Write Store'
    repository readModel   'Read Model'

    commandBus -> commandHandler 'routes command to'
    queryBus   -> queryHandler   'routes query to'
    commandHandler -> writeStore 'writes to'
    queryHandler   -> readModel  'reads from'
  }
}
```

---

## Agent Checklist — Component View

- [ ] Scope the view to the specific container (`view of myPlatform.api`)
- [ ] Every component has a `technology` label indicating its implementation
- [ ] Architectural layers are visible (inbound → domain → outbound)
- [ ] Relationship labels describe *what* crosses the boundary, not just "calls"
- [ ] External datastores and queues included to show integration points
- [ ] Use `#tags` to mark components with cross-cutting concerns (e.g. `#deprecated`, `#team2`)
