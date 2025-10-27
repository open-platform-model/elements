# Creating New Elements

> **Step-by-step guide for adding new elements to OPM**

This guide walks through the complete process of creating a new element, from design to registration to testing.

**Audience**: Element authors, core contributors
**Prerequisites**: Understanding of CUE and OPM architecture

**Related Documentation**:

- [Element System Architecture](https://github.com/open-platform-model/core/docs/architecture/element-system.md) - Deep dive into element design
- [Element Catalog](element-catalog.md) - Existing elements for reference
- [Core README](https://github.com/open-platform-model/core/README.md) - Core repository structure

---

## Table of Contents

- [Overview](#overview)
- [Element Design](#element-design)
- [Step-by-Step Process](#step-by-step-process)
- [Element Patterns](#element-patterns)
- [Testing Elements](#testing-elements)
- [Best Practices](#best-practices)
  - [Schema Design](#schema-design)
  - [Defining Defaults](#defining-defaults-encouraged)
  - [Element Naming](#element-naming)
  - [Label Design](#label-design)
  - [Modifier Design](#modifier-design)
  - [Composite Design](#composite-design)
  - [Documentation](#documentation)

---

## Overview

### Element Creation Flow

```shell
1. Design Element
   ↓
2. Create Element File with Co-located Schema (in elements/core/)
   ↓
3. Register Element (in elements/elements.cue)
   ↓
4. Validate with CUE
   ↓
5. Test in Example Component
   ↓
6. Document in Element Catalog
```

### Core Principles

**CRITICAL**: All new elements MUST be added to `elements/elements.cue` registry to be accessible.

**File Organization**:

- **Element files** → `core/elements/core/{category}_{kind}_{name}.cue`
- **Element registry** → `core/elements/elements.cue` (MAIN ENTRY POINT)

**File Naming Convention**: `{category}_{kind}_{name}.cue`

- Examples: `workload_primitive_container.cue`, `workload_modifier_replicas.cue`, `data_composite_simple_database.cue`

**Why This Structure**:

- Schemas co-located with elements (no circular imports)
- Flat structure for easy discovery
- Category-prefixed naming for clear organization
- Unified `package core` for simple imports
- Single source of truth (registry in elements.cue)

---

## Element Design

### Before You Start

**Ask These Questions**:

1. **Does an element already exist for this?**
   - Check [Element Catalog](element-catalog.md)
   - Could you compose existing elements instead?

2. **What kind of element is needed?**
   - **Primitive**: Creates standalone resource (Container, Volume)
   - **Modifier**: Enhances other elements (Replicas, HealthCheck)
   - **Composite**: Bundles primitives + modifiers (StatelessWorkload)
   - **Custom**: Last resort for platform-specific needs

3. **Where is it applied?**
   - **component**: Individual component configuration
   - **scope**: Cross-cutting concern across multiple components
   - **both**: Can be used in either context

4. **What category and labels should it have?**
   - **Category**: workload, data, connectivity, security, observability, etc.
   - **Additional labels**: platform compatibility, maturity level, compliance support
   - Use labels for flexible categorization and filtering

5. **What does it compose or modify?**
   - Composites: Which primitives/modifiers does it include?
   - Modifiers: Which primitives/composites does it enhance?

### Design Example: PodSecurity Element

Let's design a PodSecurity modifier:

**Decisions**:

- **Kind**: Modifier (enhances workloads, doesn't create separate resource)
- **Target**: component, scope (can apply to individual components or all via scope)
- **Modifies**: Container, All Workload Composites
- **Labels**:
  - `"core.opm.dev/category": "security"`
  - `"core.opm.dev/maturity": "stable"`

---

## Step-by-Step Process

### Step 1: Create Element File with Co-located Schema

Create element file in `core/elements/core/security_modifier_pod_security.cue`:

```cue
// core/elements/core/security_modifier_pod_security.cue
package core

import (
    opm "github.com/open-platform-model/core"
)

// Schema definition (co-located with element)
#PodSecuritySpec: {
    // Run container as non-root user (default: secure)
    runAsNonRoot?: bool | *true

    // User ID to run container (no default - application-specific)
    runAsUser?: int & >0

    // Group ID for filesystem (no default - application-specific)
    fsGroup?: int & >0

    // Make root filesystem read-only (default: secure)
    readOnlyRootFilesystem?: bool | *true

    // Prevent privilege escalation (default: secure)
    allowPrivilegeEscalation?: bool | *false

    // Linux capabilities to drop (default: drop all for security)
    dropCapabilities?: [...string] | *["ALL"]

    // Linux capabilities to add (default: none, opt-in only)
    addCapabilities?: [...string] | *[]

    // SELinux options (no defaults - environment-specific)
    seLinuxOptions?: {
        level?: string
        role?:  string
        type?:  string
        user?:  string
    }

    // Seccomp profile (default: use runtime default)
    seccompProfile?: {
        type: "RuntimeDefault" | "Localhost" | "Unconfined" | *"RuntimeDefault"
        localhostProfile?: string  // Only if type is Localhost
    }
}

// Element definition
#PodSecurityElement: opm.#Modifier & {
    name: "PodSecurity"
    description: "Pod security context for running containers securely"
    target: ["component", "scope"]

    // Which elements this can modify
    modifies: [
        "core.opm.dev/v0.Container",
        "core.opm.dev/v0.StatelessWorkload",
        "core.opm.dev/v0.StatefulWorkload",
        "core.opm.dev/v0.DaemonWorkload",
        "core.opm.dev/v0.TaskWorkload",
        "core.opm.dev/v0.ScheduledTaskWorkload"
    ]

    labels: {
        "core.opm.dev/category": "security"
    }

    schema: #PodSecuritySpec
}

// Usage pattern with #Component
#PodSecurity: opm.#Component & {
    #elements: PodSecurity: #PodSecurityElement

    podSecurity: #PodSecuritySpec
}
```

**Best Practices**:

- Use `?` for optional fields
- **Provide sensible defaults with `| *value`** (strongly encouraged!)
- Add constraints (`int & >0`, `string & =~"pattern"`)
- Document fields with inline comments explaining defaults
- Follow OpenAPIv3 compatibility
- Default to secure, production-ready values
- Co-locate schema and element in same file
- Use descriptive `name` and `description`
- Specify `target` accurately
- For modifiers, explicitly list what they `modifies`
- Add appropriate `labels` for categorization

### Step 2: Register in Element Registry

**CRITICAL STEP**: Add to `core/elements/elements.cue`:

```cue
// core/elements/elements.cue
package elements

import (
    // ... existing imports
    core "github.com/open-platform-model/elements/core"
)

// Export element for use in components
#PodSecurity: opm.#PodSecurity
#PodSecurityElement: opm.#PodSecurityElement

// Register in element registry
#CoreElementRegistry: {
    // ... existing elements

    // Add new element
    (#PodSecurityElement.#fullyQualifiedName): #PodSecurityElement
}
```

**Without this step, your element won't be accessible!**

### Step 3: Validate with CUE

Always validate after creating an element:

```bash
# Format code
cue fmt ./...

# Validate all definitions
cue vet ./...

# Check for errors
cue vet --all-errors ./...

# Verify element is registered
cue export ./elements/elements.cue -e '#CoreElementRegistry' --out json | jq 'keys' | grep PodSecurity
```

### Step 4: Test in Example Component

Create test component in `core/examples/`:

```cue
// core/examples/test_podsecurity.cue
package examples

import (
    opm "github.com/open-platform-model/core"
    elements "github.com/open-platform-model/core/elements"
)

// Test component using PodSecurity
secureApp: opm.#Component & {
    #metadata: {
        #id: "secure-app"
    }

    // Use StatelessWorkload composite
    elements.#StatelessWorkload
    elements.#PodSecurity

    stateless: {
        container: {
            image: "nginx:latest"
            ports: http: {targetPort: 80}
        }
        replicas: {count: 2}
    }

    // Configure security
    podSecurity: {
        runAsNonRoot: true
        runAsUser: 1000
        fsGroup: 2000
        readOnlyRootFilesystem: true
        dropCapabilities: ["ALL"]
    }
}
```

Validate the example:

```bash
cue export ./examples/test_podsecurity.cue -e secureApp --out json
```

### Step 5: Document in Element Catalog

Add to `elements/docs/element-catalog.md` in appropriate section:

```markdown
### Security Modifiers

| Status | Element | Kind | Target | Modifies | Description | Configuration |
|--------|---------|------|--------|----------|-------------|---------------|
| ✅ | **PodSecurity** | modifier | component, scope | Container, All Workload Composites | Pod security context (runAsUser, fsGroup, capabilities) | `podSecurity: #PodSecuritySpec` |
```

---

## Element Patterns

### Primitive Element Pattern

```cue
package core
import opm "github.com/open-platform-model/core"

// Schema definition (co-located)
#MyPrimitiveSpec: {
    field?: string | *"default"
}

// Primitive element
#MyPrimitiveElement: core.Primitive & {
    name: "MyPrimitive"
    description: "Description of what it does"
    target: ["component"]  // or ["scope"] or ["component", "scope"]
    workloadType: "stateless"  // Optional
    labels: {"core.opm.dev/category": "workload"}
    schema: #MyPrimitiveSpec
}

// Usage wrapper
#MyPrimitive: opm.#Component & {
    #elements: MyPrimitive: #MyPrimitiveElement
    myPrimitive: #MyPrimitiveSpec
}
```

### Primitive Element Pattern (Map-based)

```cue
package core
import opm "github.com/open-platform-model/core"

// Schema definition (co-located)
#MyResourceSpec: {
    size?: string | *"10Gi"
}

// Primitive element (map-based for multiple instances)
#MyResourceElement: opm.#Primitive & {
    name: "MyResource"
    description: "Description of resource"
    target: ["component"]
    labels: {"core.opm.dev/category": "data"}
    schema: #MyResourceSpec
}

// Usage wrapper (map allows multiple named instances)
#MyResource: opm.#Component & {
    #elements: MyResource: #MyResourceElement
    myResources: [string]: #MyResourceSpec
}
```

### Modifier Element Pattern

```cue
package core
import opm "github.com/open-platform-model/core"

// Schema definition (co-located)
#MyModifierSpec: {
    enabled?: bool | *true
}

// Modifier element
#MyModifierElement: opm.#Modifier & {
    name: "MyModifier"
    description: "Enhances other elements"
    target: ["component"]  // or ["scope"]
    modifies: [
        "core.opm.dev/v0.Container",
        "core.opm.dev/v0.StatelessWorkload"
        // List all compatible elements
    ]
    labels: {"core.opm.dev/category": "workload"}
    schema: #MyModifierSpec
}

// Usage wrapper
#MyModifier: opm.#Component & {
    #elements: MyModifier: #MyModifierElement
    myModifier: #MyModifierSpec
}
```

### Composite Element Pattern

```cue
package core
import opm "github.com/open-platform-model/core"

// Schema definition (co-located)
#MyCompositeSpec: {
    container: #ContainerSpec
    replicas?: #ReplicasSpec
}

// Composite element
#MyCompositeElement: opm.#Composite & {
    name: "MyComposite"
    description: "Combines multiple elements"
    target: ["component"]
    workloadType: "stateless"  // Required for composites
    composes: [
        #ContainerElement,
        #ReplicasElement,
        #HealthCheckElement
        // List all composed elements
    ]
    labels: {"core.opm.dev/category": "workload"}
    schema: #MyCompositeSpec
}

// Usage wrapper
#MyComposite: opm.#Component & {
    #elements: MyComposite: #MyCompositeElement
    myComposite: #MyCompositeSpec
}
```

---

## Testing Elements

### Unit Testing

Test element definition exports correctly:

```bash
# Test element definition
cue eval ./elements/core/{category}_{kind}_{name}.cue -e '#MyElement'

# Test element exports
cue export ./elements/core/{category}_{kind}_{name}.cue -e '#MyElement' --out json
```

### Integration Testing

Test in a component:

```bash
# Create test component
cue export ./examples/test_my_element.cue -e testComponent --out json

# Validate against module schema
cue vet ./examples/test_my_element.cue ./module.cue
```

### Registry Testing

Verify element is accessible:

```bash
# Check element is registered
cue export ./elements/elements.cue -e '#CoreElementRegistry' --out json | jq 'keys'

# Count registered elements
cue export ./elements/elements.cue -e '#CoreElementRegistry' --out json | jq 'length'

# Get specific element
cue export ./elements/elements.cue -e '#CoreElementRegistry."core.opm.dev/v0.MyElement"' --out json
```

---

## Best Practices

### Schema Design

✅ **DO**:

- Use optional fields with defaults: `field?: type | *default`
- Add type constraints: `port: int & >0 & <65536`
- Provide enum values for fixed choices: `type: "ClusterIP" | "NodePort" | "LoadBalancer"`
- Document complex fields with comments
- Follow OpenAPIv3 patterns
- **Provide sensible defaults wherever possible** - Users should rarely need to configure basic fields

❌ **DON'T**:

- Use overly permissive types (`_` instead of specific types)
- Omit defaults for common fields
- Create deeply nested structures (keep it flat)
- Use obscure abbreviations

### Defining Defaults (Encouraged!)

**CUE supports powerful default values using disjunctions.** Always provide sensible defaults for optional fields to improve user experience.

**Default Syntax**: `field?: type | *defaultValue`

**Examples**:

```cue
// Simple default values
replicas?: int & >=1 | *1              // Defaults to 1 replica
protocol?: "TCP" | "UDP" | *"TCP"      // Defaults to TCP
restartPolicy?: "Always" | "OnFailure" | "Never" | *"Always"

// Default boolean
enabled?: bool | *true                  // Defaults to enabled

// Default string
image?: string | *"nginx:latest"        // Defaults to nginx:latest

// Default array
dropCapabilities?: [...string] | *["ALL"]  // Defaults to drop all capabilities

// Default struct
resources?: {
    requests?: {
        cpu?: string | *"100m"           // Defaults to 100 millicores
        memory?: string | *"128Mi"       // Defaults to 128 MiB
    }
    limits?: {
        cpu?: string | *"500m"
        memory?: string | *"512Mi"
    }
}

// Conditional defaults based on context
updateStrategy?: {
    if #metadata.workloadType == "stateless" {
        type: "RollingUpdate" | "Recreate" | *"RollingUpdate"
        rollingUpdate?: {
            maxUnavailable: int | *1
            maxSurge:       int | *1
        }
    }
}
```

**Why Defaults Matter**:

1. **Better UX**: Users only configure what differs from sensible defaults
2. **Security**: Default to secure options (e.g., `runAsNonRoot: true`)
3. **Best Practices**: Encode recommended values (e.g., `protocol: "TCP"`)
4. **Reduced Boilerplate**: Minimize required configuration
5. **Clear Intent**: Defaults document expected/recommended values

**Default Guidelines**:

- ✅ **DO**: Default to secure, production-ready values
- ✅ **DO**: Use industry standards (e.g., port 80 for HTTP, port 443 for HTTPS)
- ✅ **DO**: Default to most common use case (e.g., `replicas: 1` for development)
- ✅ **DO**: Make required fields explicit, optional fields defaulted
- ❌ **DON'T**: Default to values that could cause issues (e.g., `replicas: 100`)
- ❌ **DON'T**: Default to platform-specific values (keep portable)
- ❌ **DON'T**: Over-default - some fields should be explicitly chosen by users

**Example: Good Default Strategy**:

```cue
// Security element with safe defaults
#PodSecuritySpec: {
    // Security defaults - default to secure
    runAsNonRoot?: bool | *true                    // Safe default
    readOnlyRootFilesystem?: bool | *true          // Safe default
    allowPrivilegeEscalation?: bool | *false       // Safe default

    // Drop all capabilities by default (most secure)
    dropCapabilities?: [...string] | *["ALL"]

    // Only add specific capabilities if needed (user must opt-in)
    addCapabilities?: [...string] | *[]            // Empty by default

    // Use runtime default seccomp profile
    seccompProfile?: {
        type: "RuntimeDefault" | "Localhost" | "Unconfined" | *"RuntimeDefault"
        localhostProfile?: string  // No default - only if type is Localhost
    }

    // User/group IDs - no defaults (application-specific)
    runAsUser?: int & >0        // Must be explicitly set
    fsGroup?: int & >0          // Must be explicitly set
}
```

### Element Naming

✅ **DO**:

- Use clear, descriptive names: `PodSecurity`, `HealthCheck`
- Follow PascalCase for element names
- Use singular form: `Volume` not `Volumes`
- Match common industry terms when applicable

❌ **DON'T**:

- Use abbreviations: `PodSec` instead of `PodSecurity`
- Use overly generic names: `Config`, `Settings`
- Include redundant prefixes: `TraitHealthCheck` (Trait is implicit)

### Label Design

✅ **DO**:

- Always include `"core.opm.dev/category"` label with appropriate category
- Use well-known label patterns for common categorization:
  - `"core.opm.dev/category"`: workload, data, connectivity, security, observability
  - `"core.opm.dev/maturity"`: alpha, beta, stable
  - `"core.opm.dev/compliance"`: pci-dss, soc2, etc
- Add custom organization labels with your own domain prefix
- Use labels for multi-dimensional categorization (elements can have multiple categories)

❌ **DON'T**:

- Omit the category label (required for filtering)
- Use inconsistent category values across elements
- Create labels with spaces or special characters (use kebab-case)

**Label Examples**:

```cue
// Workload element
labels: {
    "core.opm.dev/category": "workload"
    "core.opm.dev/maturity": "stable"
}

// Security element with compliance
labels: {
    "core.opm.dev/category": "security"
    "core.opm.dev/compliance": "pci-dss"
    "core.opm.dev/maturity": "beta"
}

// Multi-category element
labels: {
    "core.opm.dev/category": "observability"
    "core.opm.dev/platform": "kubernetes"
    "custom.company.com/team": "platform-engineering"
}
```

### Modifier Design

✅ **DO**:

- Explicitly list all compatible elements in `modifies`
- Make modifiers work with both primitives AND composites when applicable
- Keep modifier scope narrow and focused
- Document which primitives are required

❌ **DON'T**:

- Create modifiers that modify too many unrelated elements
- Leave `modifies` empty unless truly universal
- Create modifiers that duplicate primitive functionality

### Composite Design

✅ **DO**:

- Bundle commonly-used element combinations
- Set fixed `workloadType` for clarity
- List all composed elements in `composes`
- Provide clear use case in description

❌ **DON'T**:

- Create too many composites (adds cognitive load)
- Make composites that are too specific
- Include optional elements in composites

### Documentation

✅ **DO**:

- Add element to catalog immediately
- Include usage examples in element-patterns.md
- Document design decisions for complex elements
- Update architecture docs if introducing new patterns

❌ **DON'T**:

- Skip documenting the element
- Leave implementation status unmarked
- Forget to add platform mappings

---

## Checklist

Before submitting a new element:

- [ ] Element file created in `core/elements/core/{category}_{kind}_{name}.cue`
- [ ] Schema co-located in same file as element definition
- [ ] Element registered in `core/elements/elements.cue`
- [ ] `cue fmt ./...` runs successfully
- [ ] `cue vet ./...` passes without errors
- [ ] Element appears in registry: `cue export ./elements/elements.cue -e '#CoreElementRegistry' --out json`
- [ ] Test component created in `core/examples/`
- [ ] Test component exports successfully
- [ ] Element documented in `elements/docs/element-catalog.md`
- [ ] Usage pattern added to `elements/docs/element-patterns.md` (if applicable)
- [ ] Architecture docs updated (if introducing new patterns)

---

## Example: Complete Element Creation

Here's a complete example creating a `Resources` modifier:

**1. Create File** (`core/elements/core/workload_modifier_resources.cue`):

```cue
package core

import (
    opm "github.com/open-platform-model/core"
)

// Schema definition (co-located)
#ResourceRequirements: {
    requests?: {
        cpu?: string    // e.g., "100m", "1"
        memory?: string // e.g., "128Mi", "1Gi"
    }
    limits?: {
        cpu?: string
        memory?: string
    }
}

// Element definition
#ResourcesElement: opm.#Modifier & {
    name: "Resources"
    description: "CPU and memory resource requests and limits"
    target: ["component"]
    modifies: [
        "core.opm.dev/v0.Container",
        "core.opm.dev/v0.StatelessWorkload",
        "core.opm.dev/v0.StatefulWorkload",
        "core.opm.dev/v0.DaemonWorkload",
        "core.opm.dev/v0.TaskWorkload",
        "core.opm.dev/v0.ScheduledTaskWorkload"
    ]
    labels: {"core.opm.dev/category": "workload"}
    schema: #ResourceRequirements
}

// Usage pattern
#Resources: opm.#Component & {
    #elements: Resources: #ResourcesElement
    resources: #ResourceRequirements
}
```

**2. Register** (`core/elements/elements.cue`):

```cue
import core "github.com/open-platform-model/elements/core"

#Resources: opm.#Resources
#ResourcesElement: opm.#ResourcesElement

#CoreElementRegistry: {
    (#ResourcesElement.#fullyQualifiedName): #ResourcesElement
}
```

**3. Test** (`core/examples/test_resources.cue`):

```cue
package examples

import (
    opm "github.com/open-platform-model/core"
    elements "github.com/open-platform-model/elements/core"
)

testApp: opm.#Component & {
    #metadata: #id: "test-app"

    elements.#StatelessWorkload
    elements.#Resources

    stateless: {
        container: {image: "nginx:latest"}
    }

    resources: {
        requests: {cpu: "100m", memory: "128Mi"}
        limits: {cpu: "200m", memory: "256Mi"}
    }
}
```

**4. Validate**:

```bash
cue fmt ./...
cue vet ./...
cue export ./examples/test_resources.cue -e testApp --out json
```

**Done!**

---

## Related Documentation

- **[Element System Architecture](https://github.com/open-platform-model/core/docs/architecture/element-system.md)** - Deep dive into design
- **[Element Catalog](element-catalog.md)** - All available elements
- **[Element Patterns](element-patterns.md)** - Usage patterns
- **[Core README](https://github.com/open-platform-model/core/README.md)** - Core repository guide

---

**Questions?**

- 💬 [GitHub Discussions](https://github.com/open-platform-model/opm/discussions)
- 🐛 [Issue Tracker](https://github.com/open-platform-model/core/issues)
