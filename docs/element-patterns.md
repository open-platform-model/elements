# Element Composition Patterns

> **Common patterns and best practices for composing OPM elements into components**

This guide demonstrates effective element composition patterns through practical examples. Learn how to combine elements to build robust, maintainable components.

**For element reference**: See [Element Catalog](element-catalog.md)
**For architecture details**: See [Element System Architecture](https://github.com/open-platform-model/core/docs/architecture/element-system.md)

---

## Table of Contents

- [Basic Patterns](#basic-patterns)
- [Workload Patterns](#workload-patterns)
- [Data Patterns](#data-patterns)
- [Networking Patterns](#networking-patterns)
- [Multi-Element Patterns](#multi-element-patterns)
- [Anti-Patterns](#anti-patterns)

---

## Basic Patterns

### Pattern: Simple Stateless Web Service

**Use Case**: Basic web application with horizontal scaling and health checks

```cue
import (
    elements "github.com/open-platform-model/core/elements/core"
    elements "github.com/open-platform-model/core/elements/core"
)

components: {
    web: {
        // Use StatelessWorkload composite for standard pattern
        elements.#StatelessWorkload
        elements.#Expose

        stateless: {
            container: {
                image: "nginx:latest"
                ports: http: {
                    name: "http"
                    targetPort: 80
                }
            }
            replicas: {count: 3}
            healthCheck: {
                liveness: {
                    httpGet: {
                        path: "/health"
                        port: 80
                    }
                }
            }
        }

        expose: {
            type: "LoadBalancer"
            ports: http: {
                exposedPort: 80
                targetPort: 80
            }
        }
    }
}
```

**Benefits**:

- ✅ Clear intent with StatelessWorkload
- ✅ Built-in scaling with Replicas
- ✅ Health monitoring
- ✅ External access via Expose

---

## Workload Patterns

### Pattern: Stateless API Service with Sidecar

**Use Case**: API service with logging/metrics sidecar

```cue
import (
    elements "github.com/open-platform-model/core/elements/core"
    elements "github.com/open-platform-model/core/elements/core"
)

components: {
    api: {
        elements.#StatelessWorkload
        elements.#Expose

        stateless: {
            container: {
                image: "api-service:v1.2.0"
                ports: {
                    api: {targetPort: 8080}
                    metrics: {targetPort: 9090}
                }
                env: {
                    LOG_LEVEL: {value: "info"}
                }
            }

            // Sidecar for log aggregation
            sidecarContainers: [{
                name: "fluentd"
                image: "fluentd:latest"
                volumeMounts: logs: {mountPath: "/var/log"}
            }]

            replicas: {count: 5}

            healthCheck: {
                liveness: {
                    httpGet: {path: "/health", port: 8080}
                }
                readiness: {
                    httpGet: {path: "/ready", port: 8080}
                }
            }

            updateStrategy: {
                type: "RollingUpdate"
                rollingUpdate: {
                    maxSurge: 1
                    maxUnavailable: 0
                }
            }
        }

        expose: {
            type: "ClusterIP"
            ports: api: {
                exposedPort: 8080
                targetPort: 8080
            }
        }
    }
}
```

**Benefits**:

- ✅ Separation of concerns with sidecar
- ✅ Zero-downtime deployments
- ✅ Comprehensive health checks
- ✅ Internal service exposure

### Pattern: Stateful Database

**Use Case**: Persistent database with stable identity

```cue
import (
    elements "github.com/open-platform-model/core/elements/core"
    elements "github.com/open-platform-model/core/elements/core"
)

components: {
    database: {
        elements.#StatefulWorkload
        elements.#Secret

        stateful: {
            container: {
                image: "postgres:15"
                ports: db: {targetPort: 5432}
                env: {
                    POSTGRES_DB: {value: "myapp"}
                    POSTGRES_USER: {valueFrom: {
                        secretKeyRef: {
                            name: "db-credentials"
                            key: "username"
                        }
                    }}
                    POSTGRES_PASSWORD: {valueFrom: {
                        secretKeyRef: {
                            name: "db-credentials"
                            key: "password"
                        }
                    }}
                }
            }

            volume: {
                persistentClaim: {
                    size: "100Gi"
                    storageClass: "fast-ssd"
                    accessMode: "ReadWriteOnce"
                }
            }

            replicas: {count: 3}

            initContainers: [{
                name: "init-permissions"
                image: "busybox:latest"
                command: ["sh", "-c", "chmod 700 /var/lib/postgresql/data"]
            }]

            updateStrategy: {
                type: "RollingUpdate"
            }
        }

        secrets: {
            "db-credentials": {
                type: "Opaque"
                data: {
                    username: "cG9zdGdyZXM="  // base64: postgres
                    password: "c2VjcmV0"      // base64: secret
                }
            }
        }
    }
}
```

**Benefits**:

- ✅ Persistent storage with stable identity
- ✅ Secret management for credentials
- ✅ Init containers for setup
- ✅ Controlled rolling updates

### Pattern: Background Worker / Task

**Use Case**: Run-to-completion job processing

```cue
import (
    elements "github.com/open-platform-model/core/elements/core"
    elements "github.com/open-platform-model/core/elements/core"
)

components: {
    dataProcessor: {
        elements.#TaskWorkload
        elements.#ConfigMap

        task: {
            container: {
                image: "data-processor:v2.1.0"
                command: ["python", "process.py"]
                env: {
                    CONFIG_PATH: {value: "/etc/config/processor.yaml"}
                }
            }

            restartPolicy: {
                policy: "OnFailure"
            }

            initContainers: [{
                name: "download-data"
                image: "curl:latest"
                command: ["curl", "-o", "/data/input.csv", "https://api.example.com/data"]
            }]
        }

        configMaps: {
            "processor-config": {
                data: {
                    "processor.yaml": """
                        batch_size: 1000
                        timeout: 3600
                        """
                }
            }
        }
    }
}
```

**Benefits**:

- ✅ Run-to-completion semantics
- ✅ Restart on failure only
- ✅ Init containers for data prep
- ✅ External configuration

### Pattern: Scheduled Maintenance Task

**Use Case**: Recurring cleanup or backup job

```cue
import (
    elements "github.com/open-platform-model/core/elements/core"
)

components: {
    backup: {
        elements.#ScheduledTaskWorkload

        scheduledTask: {
            schedule: "0 2 * * *"  // 2 AM daily

            container: {
                image: "backup-tool:latest"
                command: ["./backup.sh"]
                env: {
                    BACKUP_DEST: {value: "s3://backups/database"}
                }
            }

            restartPolicy: {
                policy: "OnFailure"
            }
        }
    }
}
```

**Benefits**:

- ✅ Automated scheduling
- ✅ Cron-like syntax
- ✅ Isolated execution

### Pattern: Node-Level Monitoring Agent

**Use Case**: System monitoring running on every node

```cue
import (
    elements "github.com/open-platform-model/core/elements/core"
)

components: {
    nodeExporter: {
        elements.#DaemonWorkload

        daemon: {
            container: {
                image: "prom/node-exporter:latest"
                ports: metrics: {targetPort: 9100}
            }

            updateStrategy: {
                type: "RollingUpdate"
                rollingUpdate: {
                    maxUnavailable: 1
                }
            }
        }
    }
}
```

**Benefits**:

- ✅ Runs on every node
- ✅ Controlled rolling updates
- ✅ Node-level metrics collection

---

## Data Patterns

### Pattern: Application Configuration

**Use Case**: External configuration for application

```cue
import (
    elements "github.com/open-platform-model/core/elements/core"
    elements "github.com/open-platform-model/core/elements/core"
)

components: {
    app: {
        elements.#StatelessWorkload
        elements.#ConfigMap

        stateless: {
            container: {
                image: "myapp:latest"
                env: {
                    CONFIG_FILE: {value: "/etc/config/app.yaml"}
                }
            }
        }

        configMaps: {
            "app-config": {
                data: {
                    "app.yaml": """
                        server:
                          port: 8080
                          timeout: 30s
                        features:
                          feature_a: true
                          feature_b: false
                        """
                }
            }
        }
    }
}
```

### Pattern: Shared Storage Between Components

**Use Case**: Multiple components accessing shared volume

```cue
import (
    elements "github.com/open-platform-model/core/elements/core"
    elements "github.com/open-platform-model/core/elements/core"
)

components: {
    writer: {
        elements.#StatelessWorkload
        elements.#Volume

        stateless: {
            container: {
                image: "data-writer:latest"
            }
        }

        volumes: {
            shared: {
                persistentClaim: {
                    size: "50Gi"
                    accessMode: "ReadWriteMany"  // Shared access
                    storageClass: "nfs"
                }
            }
        }
    }

    reader: {
        elements.#StatelessWorkload
        // References the same volume by name

        stateless: {
            container: {
                image: "data-reader:latest"
            }
        }

        volumes: {
            shared: {
                persistentClaim: {
                    claimName: "writer-shared"  // Reference existing claim
                }
            }
        }
    }
}
```

**Benefits**:

- ✅ Shared data access
- ✅ ReadWriteMany for multi-pod access
- ✅ Volume reuse across components

---

## Networking Patterns

### Pattern: Internal Service Communication

**Use Case**: Backend services communicating internally

```cue
import (
    elements "github.com/open-platform-model/core/elements/core"
    elements "github.com/open-platform-model/core/elements/core"
)

components: {
    frontend: {
        elements.#StatelessWorkload
        elements.#Expose

        stateless: {
            container: {
                image: "frontend:latest"
                ports: http: {targetPort: 3000}
                env: {
                    BACKEND_URL: {value: "http://backend:8080"}
                }
            }
        }

        expose: {
            type: "LoadBalancer"
            ports: http: {exposedPort: 80, targetPort: 3000}
        }
    }

    backend: {
        elements.#StatelessWorkload
        elements.#Expose

        stateless: {
            container: {
                image: "backend:latest"
                ports: api: {targetPort: 8080}
            }
        }

        expose: {
            type: "ClusterIP"  // Internal only
            ports: api: {exposedPort: 8080, targetPort: 8080}
        }
    }
}
```

**Benefits**:

- ✅ Frontend externally accessible
- ✅ Backend internal only
- ✅ Service discovery via DNS

### Pattern: Network Isolation with Scope

**Use Case**: Isolate database from external access

```cue
import (
    elements "github.com/open-platform-model/core/elements/core"
)

scopes: {
    "database-network": {
        elements.#NetworkScope

        networkScope: {
            ingress: {
                // Only allow traffic from app components
                allowFrom: ["app", "backend"]
            }
            egress: {
                // No external egress
                allowTo: []
            }
        }

        appliesTo: ["database"]
    }
}
```

**Benefits**:

- ✅ Network segmentation
- ✅ Zero-trust networking
- ✅ Scope-level policy enforcement

---

## Multi-Element Patterns

### Pattern: Complete Web Application Stack

**Use Case**: Full 3-tier application

```cue
import (
    elements "github.com/open-platform-model/core/elements/core"
    elements "github.com/open-platform-model/core/elements/core"
    elements "github.com/open-platform-model/core/elements/core"
)

components: {
    // Frontend
    web: {
        elements.#StatelessWorkload
        elements.#Expose
        elements.#ConfigMap

        stateless: {
            container: {
                image: "webapp-frontend:v1.0.0"
                ports: http: {targetPort: 80}
            }
            replicas: {count: 3}
            healthCheck: {
                liveness: {httpGet: {path: "/", port: 80}}
            }
        }

        configMaps: {
            config: {
                data: {
                    "app.conf": "API_URL=http://api:8080"
                }
            }
        }

        expose: {
            type: "LoadBalancer"
            ports: http: {exposedPort: 80, targetPort: 80}
        }
    }

    // Backend API
    api: {
        elements.#StatelessWorkload
        elements.#Expose
        elements.#Secret

        stateless: {
            container: {
                image: "webapp-api:v1.0.0"
                ports: api: {targetPort: 8080}
                env: {
                    DB_HOST: {value: "database"}
                    DB_USER: {valueFrom: {secretKeyRef: {name: "db-creds", key: "user"}}}
                    DB_PASS: {valueFrom: {secretKeyRef: {name: "db-creds", key: "password"}}}
                }
            }
            replicas: {count: 5}
            healthCheck: {
                liveness: {httpGet: {path: "/health", port: 8080}}
                readiness: {httpGet: {path: "/ready", port: 8080}}
            }
        }

        secrets: {
            "db-creds": {
                type: "Opaque"
                data: {
                    user: "YXBpdXNlcg=="
                    password: "c2VjcmV0cGFzcw=="
                }
            }
        }

        expose: {
            type: "ClusterIP"
            ports: api: {exposedPort: 8080, targetPort: 8080}
        }
    }

    // Database
    database: {
        elements.#StatefulWorkload

        stateful: {
            container: {
                image: "postgres:15"
                ports: db: {targetPort: 5432}
            }
            volume: {
                persistentClaim: {
                    size: "100Gi"
                    storageClass: "fast-ssd"
                }
            }
            replicas: {count: 1}
        }
    }
}

scopes: {
    "db-network-isolation": {
        elements.#NetworkScope
        networkScope: {
            ingress: {allowFrom: ["api"]}
        }
        appliesTo: ["database"]
    }
}
```

**Benefits**:

- ✅ Complete application stack
- ✅ Proper network isolation
- ✅ Secret management
- ✅ Service discovery
- ✅ External and internal services

---

## Anti-Patterns

### ❌ Anti-Pattern: Mixing Workload Types

**Problem**: Cannot have multiple workloadTypes in one component

```cue
// INVALID - Don't do this!
invalid: {
    elements.#StatelessWorkload  // workloadType: "stateless"
    elements.#StatefulWorkload   // workloadType: "stateful" - CONFLICT!
}
```

**Solution**: Use separate components

```cue
// CORRECT
frontend: {
    elements.#StatelessWorkload
    // stateless configuration
}

database: {
    elements.#StatefulWorkload
    // stateful configuration
}
```

### ❌ Anti-Pattern: Modifier Without Compatible Workload

**Problem**: Modifier requires specific primitive/composite

```cue
// INVALID - Don't do this!
invalid: {
    elements.#Volume              // Just a resource
    elements.#SidecarContainers   // ERROR: Needs workload
}
```

**Solution**: Include compatible workload

```cue
// CORRECT
valid: {
    elements.#StatelessWorkload   // Provides workload
    elements.#Volume              // Resource
    // Sidecar now works because StatelessWorkload includes it
}
```

### ❌ Anti-Pattern: Replicas on Daemon

**Problem**: Daemons run one pod per node, can't set replica count

```cue
// INVALID - Don't do this!
invalid: {
    elements.#DaemonWorkload

    daemon: {
        container: {image: "node-exporter:latest"}
    }

    replicas: {count: 3}  // ERROR: Daemons don't use replicas
}
```

**Solution**: Remove Replicas modifier

```cue
// CORRECT
valid: {
    elements.#DaemonWorkload

    daemon: {
        container: {image: "node-exporter:latest"}
    }
    // No replicas - runs on all nodes automatically
}
```

### ❌ Anti-Pattern: Mixing Composite with Primitive

**Problem**: Composites already include the primitive

```cue
// INVALID - Don't do this!
invalid: {
    elements.#Container          // Primitive
    elements.#StatelessWorkload  // Composite (includes Container)
    // Redundant and causes workloadType conflict
}
```

**Solution**: Use composite OR primitive, not both

```cue
// CORRECT - Use composite
recommended: {
    elements.#StatelessWorkload
    stateless: {
        container: {image: "app:latest"}
    }
}

// OR

// CORRECT - Use primitive with modifiers
advanced: {
    elements.#Container
    elements.#Replicas
    elements.#HealthCheck

    container: {image: "app:latest"}
    replicas: {count: 3}
    healthCheck: {liveness: {httpGet: {path: "/health", port: 8080}}}
}
```

### ❌ Anti-Pattern: Hardcoded Secrets

**Problem**: Secrets in plain text in module definition

```cue
// AVOID - Don't do this in production!
database: {
    elements.#StatefulWorkload

    stateful: {
        container: {
            env: {
                DB_PASSWORD: {value: "supersecret123"}  // Plain text!
            }
        }
    }
}
```

**Solution**: Use Secret element

```cue
// CORRECT
database: {
    elements.#StatefulWorkload
    elements.#Secret

    stateful: {
        container: {
            env: {
                DB_PASSWORD: {
                    valueFrom: {
                        secretKeyRef: {
                            name: "db-secret"
                            key: "password"
                        }
                    }
                }
            }
        }
    }

    secrets: {
        "db-secret": {
            type: "Opaque"
            data: {
                password: "base64encodedvalue"
            }
        }
    }
}
```

---

## Best Practices

### 1. Start with Composites

Use composite elements (StatelessWorkload, StatefulWorkload) for standard patterns. Only use primitives directly when you need custom composition.

### 2. Use Descriptive Component Names

```cue
// Good
components: {
    userAuthService: {...}
    paymentProcessor: {...}
    postgresDatabase: {...}
}

// Avoid
components: {
    service1: {...}
    app: {...}
    db: {...}
}
```

### 3. Externalize Configuration

Use ConfigMaps and Secrets rather than hardcoding values:

```cue
// Good
container: {
    env: {
        API_KEY: {valueFrom: {secretKeyRef: {...}}}
        CONFIG: {valueFrom: {configMapKeyRef: {...}}}
    }
}

// Avoid
container: {
    env: {
        API_KEY: {value: "hardcoded-key"}
    }
}
```

### 4. Always Include Health Checks

For production workloads, always add health checks:

```cue
stateless: {
    container: {...}
    healthCheck: {
        liveness: {httpGet: {path: "/health", port: 8080}}
        readiness: {httpGet: {path: "/ready", port: 8080}}
    }
}
```

### 5. Use Appropriate Update Strategies

Configure rolling updates for zero-downtime deployments:

```cue
stateless: {
    updateStrategy: {
        type: "RollingUpdate"
        rollingUpdate: {
            maxSurge: 1
            maxUnavailable: 0  // Zero downtime
        }
    }
}
```

### 6. Leverage Scopes for Cross-Cutting Concerns

Use scopes for policies that apply to multiple components:

```cue
scopes: {
    "production-security": {
        security.#PodSecurity
        podSecurity: {
            runAsNonRoot: true
            readOnlyRootFilesystem: true
        }
        appliesTo: "*"  // All components
    }
}
```

---

## Related Documentation

- **[Element Catalog](element-catalog.md)** - Complete element reference
- **[Creating Elements](creating-elements.md)** - Guide for adding new elements
- **[Element System Architecture](https://github.com/open-platform-model/core/docs/architecture/element-system.md)** - Deep architectural dive

---

**Last Updated**: 2025-10-01
