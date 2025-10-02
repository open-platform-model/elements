# OPM Elements Documentation

> **Official element catalog and usage guides for the Open Platform Model**

This directory contains documentation for OPM's element system - the building blocks you use to compose modules and applications.

## Quick Links

- **[Element Catalog](element-catalog.md)** - Complete list of all available elements (traits, resources, composites)
- **[Element Patterns](element-patterns.md)** - Common composition patterns and best practices
- **[Creating Elements](creating-elements.md)** - Guide for adding new elements to OPM

## What Are Elements?

Elements are the atomic building blocks of OPM. Like LEGO blocks, primitive elements (Container, Volume, Network) combine into composite elements (StatelessWorkload, Database), which compose into complete modules.

**Key Principle**: Everything is an element.

### Element Kinds

- **Primitive**: Basic building blocks (Container, Volume)
- **Modifier**: Enhance primitives (Replicas, HealthCheck, Expose)
- **Composite**: Combine primitives + modifiers (StatelessWorkload, StatefulWorkload)
- **Custom**: Platform-specific extensions

## Using Elements

### In Components

Elements compose together in components:

```cue
import (
    elements "github.com/open-platform-model/core/elements"
)

components: {
    web: {
        // Use composite element for common pattern
        elements.#StatelessWorkload

        stateless: {
            container: {
                image: "nginx:latest"
                ports: http: {targetPort: 80}
            }
            replicas: {count: 3}
            healthCheck: {
                liveness: {
                    httpGet: {path: "/health", port: 80}
                }
            }
        }
    }
}
```

### Finding Elements

**By Category**:

- **Workload**: Container, StatelessWorkload, StatefulWorkload, Replicas, HealthCheck
- **Data**: Volume, ConfigMap, Secret
- **Connectivity**: Expose, NetworkScope

**By Use Case**:

- Need a web service? → `StatelessWorkload`
- Need a database? → `StatefulWorkload` or `SimpleDatabase`
- Need to scale? → `Replicas`
- Need health checks? → `HealthCheck`
- Need to expose service? → `Expose`

See [Element Catalog](element-catalog.md) for complete list.

## Documentation Structure

### [Element Catalog](element-catalog.md)

**Purpose**: Reference guide for all available elements
**Audience**: Module developers, anyone using elements
**Contains**:

- Implementation status (✅ implemented, 📋 planned)
- Complete element tables organized by category
- Configuration examples
- Platform mappings (OPM → Kubernetes, Docker, etc.)

### [Element Patterns](element-patterns.md)

**Purpose**: Common composition patterns and best practices
**Audience**: Module developers learning effective patterns
**Contains**:

- Component composition examples
- Multi-element patterns
- Anti-patterns to avoid
- Real-world usage examples

### [Creating Elements](creating-elements.md)

**Purpose**: Guide for adding new elements to OPM
**Audience**: Element authors, core contributors
**Contains**:

- Step-by-step element creation process
- Element design guidelines
- Schema definition patterns
- Registration process

## Architecture Documentation

For deep understanding of how the element system works internally:

- **[Element System Architecture](https://github.com/open-platform-model/core/docs/architecture/element-system.md)** - Type system, #ElementBase pattern, validation logic
- **[Core Architecture](https://github.com/open-platform-model/core/docs/architecture/)** - Full architectural documentation

## Related Repositories

- **[Core](https://github.com/open-platform-model/core)** - Core framework and element type system
- **[OPM](https://github.com/open-platform-model/opm)** - Main project documentation and guides
- **[Providers](https://github.com/open-platform-model/providers)** - Platform-specific implementations

## Implementation Status

**Current State** (Phase 1):

✅ **Implemented**:

- Workload: Container (primitive), 8 modifiers, 5 composites
- Data: Volume, ConfigMap, Secret (primitives), SimpleDatabase (composite)
- Connectivity: NetworkScope (primitive), Expose (modifier)

📋 **Planned** (Phases 2-3):

- Additional security, governance, observability elements
- Cloud provider-specific elements
- Advanced networking elements

See [ROADMAP](https://github.com/open-platform-model/opm/ROADMAP.md) for details.

## Contributing

Want to add a new element or improve existing ones?

1. Read [Creating Elements](creating-elements.md)
2. Review [Element System Architecture](https://github.com/open-platform-model/core/docs/architecture/element-system.md)
3. Check existing elements in [Element Catalog](element-catalog.md)
4. Follow the contribution guidelines in the [OPM repo](https://github.com/open-platform-model/opm/CONTRIBUTING.md)

## Need Help?

- 💬 [GitHub Discussions](https://github.com/open-platform-model/opm/discussions)
- 🐛 [Issue Tracker](https://github.com/open-platform-model/elements/issues)
- 📖 [Full Documentation](https://github.com/open-platform-model/opm/docs)
