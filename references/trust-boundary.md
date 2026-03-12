# Trust Boundaries & Connection Detail in LikeC4

Two concerns addressed here:
1. **Trust boundaries** — grouping elements into named security/network zones
2. **Connection detail** — surfacing hostnames, ports, protocols, service accounts on demand

Read this file whenever the user asks for trust boundaries, network zones, security perimeters,
connection detail, per-view detail levels, or "show/hide technical detail".

---

## Part 1 — Trust Boundaries

LikeC4 has no built-in `boundary` primitive. Model trust zones using a **custom element kind**
that acts as a grouping container with near-transparent styling.

### Specification

```c4
specification {
  element boundary {
    style {
      color secondary
      opacity 5%      // near-transparent fill → reads as a labelled outline zone
      border dotted   // dotted border reinforces the "zone" feel
    }
  }
  tag trustBoundary   // optional — lets you select all boundaries at once
}
```

> **`border` values**: `dashed` (default), `dotted`, `solid`, `none`
> **`opacity`**: applies to the container fill, not to nested child elements.
> Setting `opacity 5%` is the closest LikeC4 gets to a "border-only" shape.

### Model — nest elements inside their zone

```c4
model {
  boundary userNetwork 'User Network' {
    #trustBoundary
    description 'Corporate LAN — end users access the system from here'
    person developer 'Developer' 'Reviews dashboard and raises waiver requests'
  }

  boundary internet 'Internet / Cloud' {
    #trustBoundary
    description 'Public internet and cloud-hosted SaaS'
    externalSystem entraID 'Azure Entra ID' {
      #external
      description 'Cloud identity provider for SSO'
    }
  }

  boundary internalNetwork 'Internal Network' {
    #trustBoundary
    description 'Private network hosting internal services'
    externalSystem ingressController 'Ingress Controller' {
      description 'Terminates external traffic at the network edge'
    }
    system myApp 'My Application' {
      description 'Core service'
    }
  }

  // Cross-boundary relationships MUST use dot-notation
  userNetwork.developer             -> internalNetwork.ingressController 'accesses over HTTPS'
  internalNetwork.ingressController -> internalNetwork.myApp             'proxies requests to'
  userNetwork.developer             -> internet.entraID                  'authenticates via SSO'
  internalNetwork.myApp             -> internet.entraID                  'validates tokens with'
}
```

### View — expand children inside zones

`include *` only renders the boundary boxes, **not** their children. Always expand explicitly:

```c4
views {
  view index {
    title 'System Context with Trust Boundaries'

    include userNetwork, internet, internalNetwork   // the zones
    include userNetwork.*                            // children inside zones
    include internet.*
    include internalNetwork.*

    style userNetwork     { color green;  opacity 5%; border dotted }
    style internet        { color indigo; opacity 5%; border dotted }
    style internalNetwork { color sky;    opacity 5%; border dotted }

    style userNetwork.developer    { color green;   shape person }
    style internet.entraID         { color indigo;  opacity 80% }
    style internalNetwork.myApp    { color primary; opacity 90% }
  }
}
```

### Suggested zone colours

| Zone                        | `color`   |
|-----------------------------|-----------|
| User / client network       | `green`   |
| Internet / cloud / SaaS     | `indigo`  |
| Internal / private network  | `sky`     |
| DMZ / edge                  | `amber`   |
| Restricted / high-trust     | `red`     |

---

## Part 2 — Connection Detail (host, port, protocol, service accounts)

### What LikeC4 actually renders on the canvas

This is the most important thing to get right. Only these fields appear visually on the diagram:

| DSL field              | Where it appears                       | Notes                              |
|------------------------|----------------------------------------|------------------------------------|
| relationship `title`   | Edge label (always)                    | Keep short — human-readable label  |
| `technology` on rel    | Second line on the edge label (always) | Use for protocol:port, e.g. `'HTTPS:443'` |
| `metadata { ... }`     | **Details panel on click** (always)    | Full detail — host, auth, authz, notes |
| element `technology`   | Subtitle on the node card              | Use for runtime/stack label        |
| element `description`  | Body of the node card                  | Human-readable summary             |

> ⚠️ `notation` inside a `style` block defines the **diagram legend entry** for an element kind.
> It does NOT render text on individual nodes. Do not use it to surface per-element detail.

### The three-field pattern for connections

Put connection detail in all three places, each serving a different audience:

```c4
model {
  myApp -> nexusIQ 'pulls vulnerability data from' {  // (1) short human label on edge
    technology 'HTTPS:443'                             // (2) protocol:port — renders on edge
    metadata {                                         // (3) full detail — on click
      authn     'Service account — svc-app@example.com'
      authz     'API token (read-only)'
      host      'nexusiq.internal.example.com'
      direction 'outbound from myApp'
    }
  }
}
```

> ⚠️ **Valid metadata keys**: Any identifier is valid as a metadata key (e.g., `authn`, `authz`, `host`, `flow`, `direction`, `port`, `protocol`, `serviceAccount`).  
> **`notes` is NOT valid inside `metadata {}`** — it is a top-level element/relationship property, not a metadata attribute. Use a descriptive custom key like `description` or `comment` instead, or place `notes` at the top level of the element/relationship body.

### Metadata on elements (host, port, service account)

Store per-element connection properties in `metadata`:

```c4
model {
  externalSystem nexusIQ 'NexusIQ' {
    description 'Sonatype NexusIQ — vulnerability scan data source'
    metadata {
      host           'nexusiq.internal.example.com'
      port           '443'
      protocol       'HTTPS'
      serviceAccount 'svc-app-nexus@example.com'
      apiToken       'read-only NexusIQ API token'
    }
  }
}
```

Metadata is **always visible on click** in every view — no special view configuration needed.
Metadata properties are displayed **alphabetically** in the details panel regardless of definition order.

### Two-view pattern: high-level vs security review

Use `extends` to build a security-review view on top of the base view without duplication:

```c4
views {
  // Base view — clean, audience-friendly
  view index {
    title 'System Context'
    include userNetwork, internet, internalNetwork
    include userNetwork.*, internet.*, internalNetwork.*

    style userNetwork     { color green;  opacity 5%; border dotted }
    style internet        { color indigo; opacity 5%; border dotted }
    style internalNetwork { color sky;    opacity 5%; border dotted }
    // ... element styles ...
  }

  // Security review — inherits everything from index, adds visual emphasis
  view securityReview extends index {
    title 'Security Review'
    description 'Click any element or relationship for full detail: host, port, protocol, service account'

    // LikeC4 does NOT support "style A -> B {}" syntax in views.
    // To visually emphasise specific relationships, tag them in the model and style by tag.
    // Example — tag cross-boundary relationships in model:
    //   userNetwork.developer -> internalNetwork.ingressController 'accesses over HTTPS' {
    //     #crossBoundary
    //     ...
    //   }
    // Then style by tag in views:
    style element.tag = #crossBoundary {
      color red
      line solid
    }
  }
}
```

The two views differ in **visual emphasis**, not information access — metadata is always one click away.

> ⚠️ **Relationship line/color styling cannot be overridden in view `style` blocks.**  
> `line` and `color` on relationships are only valid inside `specification { relationship myKind { ... } }`.  
> To make a security-review view visually distinct from a base view, use **element style overrides**:  
> raise boundary zone opacity, change borders to `solid`, and dim element nodes — this draws the eye  
> to the relationship lines, which carry their colour from the relationship kind in specification.

---

## Agent Checklist — Trust Boundaries + Connection Detail

- [ ] `boundary` element kind declared in `specification` with `opacity 5%` and `border dotted`
- [ ] Each zone is a `boundary` in `model` with a `description`
- [ ] All elements that belong to a zone are **nested inside** it in `model`
- [ ] Cross-boundary relationships use **dot-notation** (`zone.element -> otherZone.element`)
- [ ] View uses `include boundary, boundary.*` — not just `include *` — to expand children inside zones
- [ ] Relationship `title` is a short human label
- [ ] `technology` on each relationship carries `'PROTOCOL:PORT'` — this renders on the edge
- [ ] `metadata` on elements holds host, port, protocol, service account, API token scope
- [ ] `metadata` on relationships holds authn, authz, flow, direction detail
- [ ] Security view uses `extends baseView` — never duplicates includes or base styles
- [ ] Security view re-styles cross-boundary edges with `line solid` and a distinct `color` — style by **tag** (e.g. `style element.tag = #crossBoundary { ... }`), NOT with `style A -> B {}` syntax (invalid DSL)
- [ ] Do NOT use `notation` in `style` blocks to surface per-element detail — it only sets legend text
