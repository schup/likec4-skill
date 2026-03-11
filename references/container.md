# Container / Service Map View (C4 Level 2)

**Goal**: Zoom into a single software system and show its major containers —
web apps, APIs, background workers, databases, message queues, etc.
Audience: developers, architects, ops engineers.

A "container" in C4 is anything that runs separately and is deployable independently
(a process, a service, a database, a mobile app).

---

## Specification Pattern

```c4
specification {
  element person
  element externalSystem
  element system

  // Container-level kinds
  element webApp      { style { shape browser } }
  element mobileApp   { style { shape mobile  } }
  element apiService  { style { shape rectangle } }
  element worker      { style { shape rectangle } }
  element database    { style { shape cylinder  } }
  element queue       { style { shape queue     } }
  element cache       { style { shape storage   } }

  relationship async  { style { line dashed } }
  relationship sync   { style { line solid  } }
}
```

---

## Model Pattern

```c4
model {
  person customer 'Customer'

  system myPlatform 'My Platform' {
    // Containers are nested inside the system
    webApp    spa     'Single Page App'  { technology 'React / TypeScript' }
    mobileApp ios     'iOS App'          { technology 'Swift'              }
    apiService api    'API Gateway'      { technology 'Node.js / Express'  }
    apiService auth   'Auth Service'     { technology 'Go'                 }
    worker    jobs    'Job Processor'    { technology 'Python / Celery'    }
    database  db      'Primary Database' { technology 'PostgreSQL'         }
    database  readDb  'Read Replica'     { technology 'PostgreSQL replica' }
    cache     redis   'Cache'            { technology 'Redis'              }
    queue     mq      'Message Queue'    { technology 'RabbitMQ'          }

    // Internal relationships
    spa   -> api    'calls'       sync
    ios   -> api    'calls'       sync
    api   -> auth   'validates token via' sync
    api   -> db     'reads/writes'        sync
    api   -> redis  'caches responses'    async
    api   -> mq     'publishes events'    async
    jobs  -> mq     'consumes from'       async
    jobs  -> db     'writes results'      sync
    db    -> readDb 'replicates to'       async
  }

  // Cross-system relationships (enter from outside)
  externalSystem paymentGateway 'Payment Gateway' { #external }

  customer          -> myPlatform.spa 'uses'
  myPlatform.api    -> paymentGateway 'charges card via'
}
```

---

## View Patterns

### Full container map (show all containers inside the system)

```c4
views {
  view containerMap of myPlatform {
    title 'Container Map — My Platform'
    description 'All runtime containers within My Platform and their interactions'

    include *                        // expands myPlatform's children
    include customer                 // bring in external actor
    include paymentGateway           // bring in external dependency
  }
}
```

### Filtered view (focus on one subsystem)

```c4
views {
  view apiDetails of myPlatform {
    title 'API Layer Detail'
    include myPlatform.api
    include myPlatform.api ->         // everything api calls
    include -> myPlatform.api         // everything that calls api
  }
}
```

### Multiple levels in one view (system + its containers)

```c4
views {
  view overview {
    title 'Platform Overview'
    include myPlatform.*              // expand one level inside myPlatform
    include customer, paymentGateway  // show related external elements
  }
}
```

---

## Styling Conventions

Use color to distinguish container roles:

```c4
views {
  view containerMap of myPlatform {
    include *

    style myPlatform.db, myPlatform.readDb {
      color amber
    }
    style myPlatform.redis {
      color sky
    }
    style myPlatform.mq {
      color indigo
    }
    style *.external {
      color muted
      opacity 50%
    }
  }
}
```

---

## Agent Checklist — Container View

- [ ] Each independently deployable unit is a separate element (don't collapse services into one)
- [ ] `technology` property set on every container
- [ ] Databases, queues, caches use the appropriate `shape`
- [ ] Relationships carry a short label describing the communication style
- [ ] `async` relationship kind used for event-driven / queue-based flows
- [ ] External systems and actors included in the view to show integration points
- [ ] Scoped view (`view of myPlatform`) so short names resolve correctly
