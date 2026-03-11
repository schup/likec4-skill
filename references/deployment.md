# Deployment View

**Goal**: Show how the logical model (systems/containers) maps onto physical infrastructure —
cloud regions, Kubernetes clusters, VMs, environments (dev/staging/prod).
Audience: platform engineers, SREs, DevOps teams.

In LikeC4, deployment is a separate `deployment` block that *references* logical elements
via `instanceOf`. It has its own hierarchy of nodes.

---

## Specification Pattern

```c4
specification {
  // Define your deployment node kinds
  deploymentNode environment     // dev / staging / prod
  deploymentNode cloudRegion     // aws us-east-1, etc.
  deploymentNode availabilityZone
  deploymentNode kubernetesCluster
  deploymentNode kubernetesNamespace
  deploymentNode pod
  deploymentNode vm
  deploymentNode loadBalancer
  deploymentNode cdn

  // Reuse logical element kinds for instances
  element webApp
  element apiService
  element database
  element cache
  element queue
}
```

---

## Model — Logical Layer (abbreviated, for reference)

```c4
model {
  system myPlatform 'My Platform' {
    webApp    spa   'SPA'           { technology 'React'       }
    apiService api  'API Service'   { technology 'Node.js'     }
    database  db    'Primary DB'    { technology 'PostgreSQL'  }
    cache     redis 'Cache'         { technology 'Redis'       }
  }
}
```

---

## Deployment Model Pattern

```c4
deployment {
  // ── Production environment ─────────────────────────────────────────
  environment prod 'Production' {

    cdn cdnEdge 'CDN Edge' {
      instanceOf myPlatform.spa    // SPA served from CDN
    }

    cloudRegion usEast1 'AWS us-east-1' {

      loadBalancer alb 'Application Load Balancer'

      kubernetesCluster eks 'EKS Cluster' {

        kubernetesNamespace appNs 'app namespace' {
          pod apiPod1 'api-pod-1' {
            instanceOf myPlatform.api
          }
          pod apiPod2 'api-pod-2' {
            instanceOf myPlatform.api
          }
          pod redisPod 'redis-pod' {
            instanceOf myPlatform.redis
          }
        }
      }

      vm dbPrimary 'DB Primary (RDS)' {
        instanceOf myPlatform.db
      }
      vm dbReplica 'DB Replica (RDS)' {
        instanceOf myPlatform.db
      }
    }

    // Deployment-specific relationships (override or supplement logical ones)
    alb -> eks.appNs.apiPod1  'routes 50% to'
    alb -> eks.appNs.apiPod2  'routes 50% to'
    eks.appNs.apiPod1 -> dbPrimary  'writes to'
    eks.appNs.apiPod1 -> dbReplica  'reads from'
    eks.appNs.apiPod2 -> dbPrimary  'writes to'
    eks.appNs.apiPod2 -> dbReplica  'reads from'
  }

  // ── Staging environment ────────────────────────────────────────────
  environment staging 'Staging' {
    cloudRegion usEast1Staging 'AWS us-east-1 (staging)' {
      kubernetesCluster eks 'EKS Staging' {
        kubernetesNamespace appNs 'staging namespace' {
          pod apiPod 'api-pod' {
            instanceOf myPlatform.api
          }
        }
      }
      vm dbStaging 'DB Staging' {
        instanceOf myPlatform.db
      }
    }
  }
}
```

---

## Deployment View Pattern

```c4
views {
  // Full production deployment
  deploymentView prod 'Production Deployment' {
    title 'Production Infrastructure — My Platform'
    description 'How My Platform is deployed in the production AWS environment'
    include prod.*             // expand everything in production
  }

  // Focus: just the API tier
  deploymentView apiTier 'API Tier Deployment' {
    title 'API Tier — Kubernetes Layout'
    include prod.usEast1.eks.*
    include prod.usEast1.alb
    include prod.usEast1.dbPrimary
    include prod.usEast1.dbReplica
  }

  // Side-by-side environment comparison
  deploymentView environments 'All Environments' {
    title 'Deployment Environments'
    include prod
    include staging
  }
}
```

---

## Multi-Region Pattern

```c4
deployment {
  environment prod 'Production' {
    cloudRegion primary 'AWS us-east-1 (primary)' {
      kubernetesCluster eks {
        kubernetesNamespace app {
          pod api1 { instanceOf myPlatform.api }
        }
      }
      vm dbPrimary { instanceOf myPlatform.db }
    }

    cloudRegion failover 'AWS eu-west-1 (failover)' {
      kubernetesCluster eks {
        kubernetesNamespace app {
          pod api1 { instanceOf myPlatform.api }
        }
      }
      vm dbReplica { instanceOf myPlatform.db }
    }

    // Cross-region replication
    primary.dbPrimary -> failover.dbReplica 'replicates async'
  }
}
```

---

## Styling Deployment Views

```c4
views {
  deploymentView prod {
    include prod.*

    style prod.usEast1.eks {
      color primary
      notation 'Kubernetes'
      icon tech:kubernetes
    }
    style prod.usEast1.dbPrimary {
      color amber
      shape cylinder
    }
    style prod.cdnEdge {
      color sky
    }
  }
}
```

---

## Agent Checklist — Deployment View

- [ ] `deploymentNode` kinds defined in `specification` block
- [ ] Every `instanceOf` references a valid logical element (system/container)
- [ ] Hierarchy reflects real infra nesting (region → cluster → namespace → pod)
- [ ] Deployment-specific relationships (load balancing, replication) added in `deployment` block
- [ ] Separate environments (`prod`, `staging`, `dev`) as top-level nodes
- [ ] View uses `deploymentView` keyword (not `view`)
- [ ] View has `title` and `description` stating which environment/tier it shows
