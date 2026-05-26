---
name: hexagonal-architecture
description: Coding guidance for hexagonal architecture in ts-seed. Invoke when writing or reviewing domain/application/infrastructure code.
---

# Hexagonal Architecture in ts-seed

Hexagonal architecture (Alistair Cockburn, 2005) places the **domain at the centre** and treats everything external as an interchangeable detail. HTTP, filesystem, npm, databases: these are all details. The domain does not know they exist.

The core idea: the domain is surrounded by **ports** (interfaces) and **adapters** (concrete implementations).

## Driving vs Driven

```
                 ┌─────────────────────────────────┐
  HTTP request   │                                 │
  ──────────► [Adapter] ──► [Port]                 │
                 │               │                 │
  Next.js page   │               ▼                 │
  ──────────► [Adapter]       DOMAIN    [Port] ──► [Adapter] ──► filesystem
                 │               ▲                 │
  Unit test      │               │            [Port] ──► [Adapter] ──► npm registry
  ──────────► [Adapter] ──► [Port]                 │
                 │                                 │
                 └─────────────────────────────────┘

          DRIVING SIDE (left)                DRIVEN SIDE (right)
       What triggers the domain          What the domain drives
```

**Driving adapters** (left): what calls the domain. A Next.js API route, a React page action, a unit test. The domain does not know they exist.

**Driven adapters** (right): what the domain drives through its ports. The filesystem, an in-memory registry, a file writer. The domain defines what it needs; the adapter implements it.

## Ports

A port is a TypeScript interface defined in `brick/domain/`. It expresses what the domain needs — not how it is done.

```ts
// src/brick/domain/BrickRegistry.ts
// The domain defines what it needs from a registry — it does not know InMemoryBrickRegistry exists
export interface BrickRegistry {
  findBySlug(slug: BrickSlug): BrickDefinition | undefined
  listAll(): BrickDefinition[]
}

// src/brick/domain/ProjectWriter.ts
export interface ProjectWriter {
  apply(folder: ProjectFolder, patch: Patch): void
}
```

**Rule:** ports belong to the domain. Never define a port in `infrastructure/`.

## Adapters

An adapter implements a port. It lives in `infrastructure/secondary/` (driven) or `infrastructure/primary/` (driving).

```ts
// src/brick/infrastructure/secondary/InMemoryBrickRegistry.ts
// Implements the domain port — the domain never imports this file
export class InMemoryBrickRegistry implements BrickRegistry {
  constructor(private readonly definitions: BrickDefinition[]) {}

  findBySlug(slug: BrickSlug): BrickDefinition | undefined {
    return this.definitions.find((d) => d.slug.value === slug.value)
  }

  listAll(): BrickDefinition[] {
    return [...this.definitions]
  }
}
```

**Rule:** adapters translate between the outside world and the domain. They contain no business logic.

## The Composition Root

All wiring happens in one place: `src/brick/infrastructure/primary/container.ts`. Nowhere else creates concrete instances.

```ts
// container.ts — the only place where adapters are instantiated
const registry = new InMemoryBrickRegistry([typescriptBrickDefinition])
const writer = new FileSystemProjectWriter()
const service = new BrickApplicationService(registry, writer)
```

**Rule:** no `new ConcreteClass()` anywhere except the composition root. Everything else receives its dependencies as constructor parameters.

## The Application Layer

The application layer (`BrickApplicationService`) only orchestrates: get from registry, build patch, apply via writer. No business logic, no infrastructure knowledge.

```ts
// src/brick/application/BrickApplicationService.ts
export class BrickApplicationService {
  constructor(
    private readonly registry: BrickRegistry, // port, not implementation
    private readonly writer: ProjectWriter, // port, not implementation
  ) {}

  apply(slug: BrickSlug, context: BrickContext): void {
    const definition = this.registry.findBySlug(slug)
    if (!definition) throw new BrickNotFoundError(slug)
    const patch = definition.builder.build(context)
    this.writer.apply(context.folder, patch)
  }
}
```

## Value Objects

Value Objects are the primary building blocks in the domain. They have no identity — they are defined entirely by their value.

```ts
// src/shared/domain/BrickSlug.ts
export class BrickSlug {
  private constructor(readonly value: string) {}

  static of(value: string): BrickSlug {
    if (!value || !/^[a-z][a-z0-9-]*$/.test(value)) {
      throw new Error(`Invalid BrickSlug: "${value}"`)
    }
    return new BrickSlug(value)
  }
}
```

Pattern for every Value Object:

- `private constructor` — construction is controlled
- `static of()` factory — validates before creating
- Throws immediately on invalid input — invalid state never exists
- `readonly` fields — immutable after construction
- No ID, no identity — equality is based on value

## What belongs where

| Layer                       | Contains                                                   | Must NOT contain                               |
| --------------------------- | ---------------------------------------------------------- | ---------------------------------------------- |
| `domain/`                   | Entities, Value Objects, ports (interfaces), domain errors | Imports of `fs`, `path`, `next`, any framework |
| `application/`              | Use-case orchestration only                                | Business logic, infrastructure knowledge       |
| `infrastructure/primary/`   | HTTP controllers, Next.js route handlers, composition root | Business logic                                 |
| `infrastructure/secondary/` | Port implementations (DB, filesystem, in-memory)           | Business logic                                 |
| `catalog/*/domain/`         | BrickBuilder implementations                               | Imports of `fs`, `path`, `next`, any framework |

## Common mistakes

**Port defined in infrastructure:** the port belongs to the domain. If you are defining an interface in `infrastructure/`, move it to `domain/`.

**Business logic in an adapter:** `InMemoryBrickRegistry` should not validate a brick's slug format. That validation belongs in `BrickSlug.of()`.

**Adapter that imports another adapter:** adapters should only know about domain types and their own external dependency. No cross-adapter imports.

**`new ConcreteClass()` outside the composition root:** if domain or application code creates a concrete instance, it is coupled to that implementation. Pass it as a dependency instead.

## Testability

The real benefit of hexagonal architecture is testability. Domain types are pure — they need no mocking. Application services are testable with in-memory adapters. Infrastructure adapters are tested against the real external system.

```ts
// Testing the application service — no filesystem, no registry file
it('should apply the brick patch to the project folder', () => {
  const registry = new InMemoryBrickRegistry([typescriptBrickDefinition])
  const writer = new SpyProjectWriter()
  const service = new BrickApplicationService(registry, writer)

  service.apply(BrickSlug.of('typescript'), makeContext('5.x'))

  expect(writer.appliedPatch).toBeDefined()
  expect(writer.appliedPatch?.dependencies).toHaveLength(1)
})
```
