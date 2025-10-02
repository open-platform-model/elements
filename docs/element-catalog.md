# OPM Element Catalog

> **Complete reference of all OPM elements - the building blocks for composing modules**

This catalog provides a comprehensive reference of all available elements in OPM. Elements are organized by category and type for easy discovery.

**For architecture details**: See [Element System Architecture](https://github.com/open-platform-model/core/docs/architecture/element-system.md)
**For usage patterns**: See [Element Patterns](element-patterns.md)
**For creating elements**: See [Creating Elements](creating-elements.md)

---

## Quick Navigation

- [Understanding the Catalog](#understanding-the-catalog)
- [Workload Elements](#workload-elements)
- [Data Elements](#data-elements)
- [Connectivity Elements](#connectivity-elements)
- [Security Elements](#security-elements)
- [Governance Elements](#governance-elements)
- [Observability Elements](#observability-elements)
- [Platform Mappings](#platform-mappings)

---

## Understanding the Catalog

### Element Kinds

- **Primitive** - Basic building blocks (Container, Volume, ConfigMap)
- **Modifier** - Enhance primitives/composites (Replicas, HealthCheck, Expose)
- **Composite** - Bundle primitives + modifiers (StatelessWorkload, StatefulWorkload)
- **Custom** - Platform-specific extensions

### Element Targets

- **component** - Applied to individual components
- **scope** - Applied across multiple components
- **component, scope** - Can be used in either context

### How to Read Tables

| Status | Element | Kind | Target | Description | Configuration Field |
|--------|---------|------|--------|-------------|---------------------|
| ✅/📋 | Name | primitive/modifier/composite | component/scope | What it does | `fieldName: #SpecType` |

- **Status**: ✅ Implemented, 📋 Planned
- **Element**: Name used in code (e.g., `#Container`, `#Replicas`)
- **Kind**: How it composes (primitive, modifier, composite, custom)
- **Target**: Where it's used (component, scope, both)
- **Configuration Field**: How you configure it in CUE

---

## Workload Elements

Elements for defining containerized workloads, scaling, lifecycle, and runtime behavior.

### Workload Primitives

Basic workload building blocks implemented by the platform.

| Status | Element | Kind | Target | Description | WorkloadType | Configuration |
|--------|---------|------|--------|-------------|--------------|---------------|
| ✅ | **Container** | primitive | component | Base container definition - flexible workload primitive | stateless \| stateful \| daemon \| task \| scheduled-task | `container: #ContainerSpec` |
| 📋 | **Function** | primitive | component | Serverless function workload | function | `function: #FunctionSpec` |

**Container** is the foundational workload primitive. All workload composites build on Container.

### Workload Composites

Pre-composed workload patterns that combine Container with appropriate modifiers.

| Status | Element | Kind | Target | Description | Composes | WorkloadType | Configuration |
|--------|---------|------|--------|-------------|----------|--------------|---------------|
| ✅ | **StatelessWorkload** | composite | component | Horizontally scalable workload with no stable identity requirement | Container, Replicas, RestartPolicy, UpdateStrategy, HealthCheck, SidecarContainers, InitContainers | stateless | `stateless: #StatelessSpec` |
| ✅ | **StatefulWorkload** | composite | component | Workload requiring stable identity, persistent storage, and ordered lifecycle | Container, Volume, Replicas, RestartPolicy, UpdateStrategy, HealthCheck, SidecarContainers, InitContainers | stateful | `stateful: #StatefulWorkloadSpec` |
| ✅ | **DaemonWorkload** | composite | component | Node-level workload running one instance per node | Container, RestartPolicy, UpdateStrategy, HealthCheck, SidecarContainers, InitContainers | daemon | `daemon: #DaemonSpec` |
| ✅ | **TaskWorkload** | composite | component | Run-to-completion workload that executes and exits | Container, RestartPolicy, SidecarContainers, InitContainers | task | `task: #TaskWorkloadSpec` |
| ✅ | **ScheduledTaskWorkload** | composite | component | Recurring task triggered on a schedule | Container, RestartPolicy, SidecarContainers, InitContainers | scheduled-task | `scheduledTask: #ScheduledTaskWorkloadSpec` |

**Usage**: Start with composites for standard patterns. Use Container primitive directly only for advanced custom compositions.

### Container Modifiers

Enhance container workloads with additional containers.

| Status | Element | Kind | Target | Modifies | Description | Configuration |
|--------|---------|------|--------|----------|-------------|---------------|
| ✅ | **SidecarContainers** | modifier | component | Container, All Workload Composites | Additional containers running alongside main container | `sidecarContainers: [#ContainerSpec]` |
| ✅ | **InitContainers** | modifier | component | Container, All Workload Composites | Pre-start initialization containers | `initContainers: [#ContainerSpec]` |
| ✅ | **EphemeralContainers** | modifier | component | Container, All Workload Composites | Debug/troubleshooting containers (can be added post-deployment) | `ephemeralContainers: [#ContainerSpec]` |

### Scaling Modifiers

Control workload scaling behavior.

| Status | Element | Kind | Target | Modifies | Description | Configuration |
|--------|---------|------|--------|----------|-------------|---------------|
| ✅ | **Replicas** | modifier | component | Container, StatelessWorkload, StatefulWorkload | Static replica count | `replicas: #ReplicasSpec` |
| 📋 | **HorizontalAutoscaler** | modifier | component | Container, StatelessWorkload, StatefulWorkload | Automatic horizontal scaling based on metrics | `horizontalAutoscaler: #HPASpec` |
| 📋 | **VerticalAutoscaler** | modifier | component | Container, StatelessWorkload, StatefulWorkload | Automatic vertical scaling (CPU/memory) | `verticalAutoscaler: #VPASpec` |

### Lifecycle & Behavior Modifiers

Define workload lifecycle, health, and runtime behavior.

| Status | Element | Kind | Target | Modifies | Description | Configuration |
|--------|---------|------|--------|----------|-------------|---------------|
| ✅ | **HealthCheck** | modifier | component | Container, All Workload Composites | Liveness, readiness, and startup probes | `healthCheck: #HealthCheckSpec` |
| ✅ | **RestartPolicy** | modifier | component | Container, All Workload Composites | Pod restart behavior (Always, OnFailure, Never) | `restartPolicy: #RestartPolicySpec` |
| ✅ | **UpdateStrategy** | modifier | component | Container, StatelessWorkload, StatefulWorkload, DaemonWorkload | Rollout and update strategy | `updateStrategy: #UpdateStrategySpec` |
| 📋 | **Resources** | modifier | component | Container, All Workload Composites | CPU and memory requests/limits | `resources: #ResourceRequirements` |
| 📋 | **LifecycleHooks** | modifier | component | Container, All Workload Composites | Pre-stop and post-start hooks | `lifecycle: #LifecycleSpec` |
| 📋 | **Scheduling** | modifier | component | Container, All Workload Composites | Node affinity, anti-affinity, tolerations | `scheduling: #SchedulingSpec` |
| 📋 | **Runtime** | modifier | component | Container, All Workload Composites | Container runtime selection | `runtime: #RuntimeSpec` |
| 📋 | **Termination** | modifier | component | Container, All Workload Composites | Graceful shutdown configuration | `termination: #TerminationSpec` |

---

## Data Elements

Elements for managing configuration data, secrets, and persistent storage.

### Data Primitives

Basic data and storage building blocks.

| Status | Element | Kind | Target | Description | Creates | Configuration |
|--------|---------|------|--------|-------------|---------|---------------|
| ✅ | **Volume** | primitive | component | Persistent storage volumes | PersistentVolumeClaim / Volume | `volumes: [string]: #VolumeSpec` |
| ✅ | **ConfigMap** | primitive | component | Non-sensitive configuration data | ConfigMap | `configMaps: [string]: #ConfigMapSpec` |
| ✅ | **Secret** | primitive | component | Sensitive data (credentials, tokens) | Secret | `secrets: [string]: #SecretSpec` |
| 📋 | **ProjectedVolume** | primitive | component | Multi-source volumes (combine ConfigMaps, Secrets, etc.) | Projected Volume | `projected: #ProjectedVolumeSpec` |

**Note**: Volume names are user-defined keys in the map (e.g., `volumes: {data: {...}, logs: {...}}`).

### Data Composites

Pre-composed data patterns.

| Status | Element | Kind | Target | Description | Composes | WorkloadType | Configuration |
|--------|---------|------|--------|-------------|----------|--------------|---------------|
| ✅ | **SimpleDatabase** | composite | component | Simple containerized database pattern | Volume | stateful | `database: #SimpleDatabaseSpec` |

### Data Modifiers

Enhance data resources with backup, recovery, and caching policies.

| Status | Element | Kind | Target | Modifies | Description | Configuration |
|--------|---------|------|--------|----------|-------------|---------------|
| 📋 | **BackupPolicy** | modifier | scope | Volume, PersistentClaims | Backup schedule and retention | `backupPolicy: #BackupPolicySpec` |
| 📋 | **DisasterRecovery** | modifier | scope | Volume, PersistentClaims | Disaster recovery policies | `disasterRecovery: #DRSpec` |
| 📋 | **CachingPolicy** | modifier | scope | ConfigMap, Secret | Caching strategies | `cachingPolicy: #CachingSpec` |

---

## Connectivity Elements

Elements for networking, service exposure, and traffic management.

### Connectivity Primitives

Basic networking building blocks.

| Status | Element | Kind | Target | Description | Creates | Configuration |
|--------|---------|------|--------|-------------|---------|---------------|
| ✅ | **NetworkScope** | primitive | scope | Network boundary and isolation | NetworkPolicy | `networkScope: #NetworkScopeSpec` |
| 📋 | **HTTPRoute** | primitive | component, scope | HTTP routing rules | Ingress / Route | `httpRoute: #HTTPRouteSpec` |
| 📋 | **ServiceMesh** | primitive | scope | Service mesh integration | Mesh configuration | `serviceMesh: #ServiceMeshSpec` |

### Connectivity Modifiers

Enhance workloads with service exposure and network policies.

| Status | Element | Kind | Target | Modifies | Description | Configuration |
|--------|---------|------|--------|----------|-------------|---------------|
| ✅ | **Expose** | primitive | component | Container, All Workload Composites | Service exposure (ClusterIP, NodePort, LoadBalancer) | `expose: #ExposeSpec` |
| 📋 | **NetworkPolicy** | modifier | scope | NetworkScope, HTTPRoute | Network access control rules | `networkPolicy: #NetworkPolicySpec` |
| 📋 | **TrafficPolicy** | modifier | scope | Expose, ServiceMesh | Traffic routing, load balancing, retries | `trafficPolicy: #TrafficPolicySpec` |
| 📋 | **DNSPolicy** | modifier | scope | Container, All Workload Composites, Expose | DNS configuration | `dnsPolicy: #DNSPolicySpec` |
| 📋 | **RateLimiting** | modifier | component, scope | Expose, HTTPRoute | Request rate limits | `rateLimit: #RateLimitSpec` |

---

## Security Elements

Elements for pod security, identity, and access control.

### Security Modifiers

All security elements are modifiers that enhance workloads and scopes.

| Status | Element | Kind | Target | Modifies | Description | Configuration |
|--------|---------|------|--------|----------|-------------|---------------|
| 📋 | **PodSecurity** | modifier | component, scope | Container, All Workload Composites | Pod security context (runAsUser, fsGroup, etc.) | `podSecurity: #PodSecuritySpec` |
| 📋 | **ServiceAccount** | modifier | component | Container, All Workload Composites | Kubernetes service account for pod identity | `serviceAccount: #ServiceAccountSpec` |
| 📋 | **PodSecurityStandards** | modifier | scope | Container, All Workload Composites | Pod Security Standards enforcement (privileged, baseline, restricted) | `standards: #SecurityStandardsSpec` |
| 📋 | **Sysctls** | modifier | component | Container, All Workload Composites | Kernel parameter configuration | `sysctls: #SysctlsSpec` |

---

## Governance Elements

Elements for resource quotas, priorities, and cost management.

### Governance Modifiers

| Status | Element | Kind | Target | Modifies | Description | Configuration |
|--------|---------|------|--------|----------|-------------|---------------|
| 📋 | **Priority** | modifier | component | Container, All Workload Composites | Pod scheduling priority class | `priority: #PrioritySpec` |
| 📋 | **DisruptionBudget** | modifier | component | Container, StatelessWorkload, StatefulWorkload | Pod disruption budget for availability | `disruptionBudget: #PDBSpec` |
| 📋 | **ResourceQuota** | modifier | scope | Volume, ConfigMap, Secret | Resource consumption limits at scope level | `resourceQuota: #ResourceQuotaSpec` |
| 📋 | **ResourceLimit** | modifier | scope | Volume, ConfigMap, Secret | Resource boundaries and constraints | `resourceLimits: #ResourceLimitSpec` |
| 📋 | **CostAllocation** | modifier | scope | Volume, PersistentClaims | Cost tracking and allocation tags | `costAllocation: #CostAllocationSpec` |

---

## Observability Elements

Elements for metrics, logging, and telemetry.

### Observability Modifiers

| Status | Element | Kind | Target | Modifies | Description | Configuration |
|--------|---------|------|--------|----------|-------------|---------------|
| 📋 | **OTelMetrics** | modifier | component | Container, All Workload Composites | OpenTelemetry metrics export configuration | `otelMetrics: #OTelMetricsSpec` |
| 📋 | **OTelLogging** | modifier | component | Container, All Workload Composites | OpenTelemetry logging export configuration | `otelLogs: #OTelLogsSpec` |
| 📋 | **ObservabilityPolicy** | modifier | scope | Container, All Workload Composites | Scope-level telemetry policies | `observabilityPolicy: #ObservabilityPolicySpec` |

---

## Cross-Cutting Elements

Elements that apply to multiple categories.

### Audit & Compliance Modifiers

| Status | Element | Kind | Target | Modifies | Description | Configuration |
|--------|---------|------|--------|----------|-------------|---------------|
| 📋 | **AuditPolicy** | modifier | scope | All Workload Composites, Secret, ConfigMap, Volume | Audit logging requirements | `auditPolicy: #AuditPolicySpec` |
| 📋 | **CompliancePolicy** | modifier | scope | All Workload Composites, Secret, ConfigMap, Volume | Compliance framework rules (PCI-DSS, SOC2, etc.) | `compliance: #CompliancePolicySpec` |

---

## Platform Mappings

How OPM elements map to platform-specific resources.

### Kubernetes Mappings

**Composite Elements → Kubernetes Resources** (Recommended Pattern):

| OPM Composite | Kubernetes Resource | Notes |
|---------------|---------------------|-------|
| StatelessWorkload | Deployment | Horizontally scalable pods |
| StatefulWorkload | StatefulSet | Pods with stable identity and storage |
| DaemonWorkload | Daemon | One pod per node |
| TaskWorkload | Job | Run-to-completion workload |
| ScheduledTaskWorkload | CronJob | Scheduled recurring jobs |
| SimpleDatabase | StatefulSet + PVC + Secret | Complete database pattern |

**Primitive Elements → Kubernetes Resources** (Advanced Pattern):

| OPM Primitive | Kubernetes Resource | Notes |
|---------------|---------------------|-------|
| Container | Deployment | When used with modifiers |
| Volume | PersistentVolumeClaim | Persistent storage |
| ConfigMap | ConfigMap | Configuration data |
| Secret | Secret | Sensitive data |
| NetworkScope | NetworkPolicy | Network isolation |

**Modifier Elements → Kubernetes Field Mappings**:

| OPM Modifier | Kubernetes Field | Resource Type |
|--------------|------------------|---------------|
| Expose | Service | Creates separate Service resource |
| SidecarContainers | `spec.template.spec.containers[1:]` | Additional containers in pod |
| InitContainers | `spec.template.spec.initContainers` | Pre-start containers |
| EphemeralContainers | `spec.template.spec.ephemeralContainers` | Debug containers |
| Replicas | `spec.replicas` | Deployment, StatefulSet |
| UpdateStrategy | `spec.strategy` or `spec.updateStrategy` | Deployment, StatefulSet, Daemon |
| HealthCheck | `containers[*].livenessProbe`, `readinessProbe`, `startupProbe` | Pod spec |
| RestartPolicy | `spec.template.spec.restartPolicy` | Pod spec |
| Resources | `containers[*].resources` | Pod spec |
| PodSecurity | `spec.template.spec.securityContext` | Pod spec |

### Docker Compose Mappings (Planned)

**Composite Elements → Docker Compose**:

| OPM Composite | Docker Compose | Notes |
|---------------|----------------|-------|
| StatelessWorkload | `services.<name>` with `deploy.replicas` | Multi-replica service |
| StatefulWorkload | `services.<name>` with volumes | Service with persistent storage |

**Primitive Elements → Docker Compose**:

| OPM Primitive | Docker Compose | Notes |
|---------------|----------------|-------|
| Container | `services.<name>` | Service definition |
| Volume | `volumes:` | Named volume |
| ConfigMap | `configs:` | Configuration file |
| Secret | `secrets:` | Secret file |

**Modifier Elements → Docker Compose Fields**:

| OPM Modifier | Docker Compose Field | Notes |
|--------------|---------------------|-------|
| Expose | `ports:` | Port mappings |
| Replicas | `deploy.replicas` | Replica count |
| UpdateStrategy | `deploy.update_config` | Rolling update config |
| Resources | `deploy.resources` | Resource limits |
| RestartPolicy | `restart:` | Restart policy |

---

## Element Organization

### By Category

- **Workload**: Container primitives, scaling, lifecycle, health
- **Data**: Storage, configuration, secrets
- **Connectivity**: Networking, service exposure, traffic management
- **Security**: Pod security, identity, access control
- **Governance**: Quotas, priorities, budgets, cost tracking
- **Observability**: Metrics, logging, telemetry

### By Kind

- **Primitives** (7): Container, Function, Volume, ConfigMap, Secret, ProjectedVolume, NetworkScope, HTTPRoute, ServiceMesh
- **Modifiers** (30+): Replicas, HealthCheck, Expose, PodSecurity, etc.
- **Composites** (6): StatelessWorkload, StatefulWorkload, DaemonWorkload, TaskWorkload, ScheduledTaskWorkload, SimpleDatabase

### By Workload Type Compatibility

**Works with All Workload Types**:

- SidecarContainers, InitContainers, EphemeralContainers
- HealthCheck, RestartPolicy
- Resources, LifecycleHooks, Scheduling, Runtime, Termination
- PodSecurity, ServiceAccount, Sysctls
- OTelMetrics, OTelLogging

**Works with Scalable Workloads** (Stateless, Stateful):

- Replicas, HorizontalAutoscaler, VerticalAutoscaler
- DisruptionBudget

**Works with Stateful Workloads Only**:

- Volume (built into StatefulWorkload composite)

---

## Quick Reference

### Common Use Cases

**Need a web service?**
→ Use `StatelessWorkload` + `Expose`

**Need a database?**
→ Use `StatefulWorkload` or `SimpleDatabase`

**Need to scale?**
→ Add `Replicas` or `HorizontalAutoscaler`

**Need health checks?**
→ Add `HealthCheck`

**Need persistent storage?**
→ Use `Volume` (or `StatefulWorkload` which includes it)

**Need configuration data?**
→ Use `ConfigMap` or `Secret`

**Need service exposure?**
→ Add `Expose`

**Need network isolation?**
→ Use `NetworkScope`

### Finding Elements

1. **Browse by category** above (Workload, Data, Connectivity, etc.)
2. **Check implementation status** (✅ available now, 📋 coming soon)
3. **Read element description** to understand purpose
4. **Check "Modifies" column** for modifiers to see compatibility
5. **Review configuration field** for usage syntax

---

## Related Documentation

- **[Element System Architecture](https://github.com/open-platform-model/core/docs/architecture/element-system.md)** - Deep dive into element type system and design
- **[Element Patterns](element-patterns.md)** - Common composition patterns and best practices
- **[Creating Elements](creating-elements.md)** - Guide for adding new elements
- **[Core Documentation](https://github.com/open-platform-model/core/docs/)** - Core framework documentation

---

**Last Updated**: 2025-10-01
**Elements Count**: 20 implemented, 29+ planned
**Version**: Phase 1 (Core Elements)
