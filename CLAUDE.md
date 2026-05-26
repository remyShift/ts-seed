# ts-seed

TypeScript project bootstrapper. The user picks composable **bricks** from a catalogue, configures their project, and the tool generates the corresponding files in a target folder.

## Commands

```bash
npm run dev          # starts Next.js in dev mode (port 3000)
npm test             # vitest in watch mode
npm run test:run     # vitest one-shot
npm run test:coverage
npm run lint         # eslint src/
npm run format       # prettier --write .
```

## Architecture

Two bounded contexts inside `src/`:

**`brick/`** — the generic engine. Knows nothing about TypeScript, Express, or any technology. Handles: cataloguing bricks, applying a brick, writing the result to the filesystem.

**`catalog/`** — concrete implementations. One subfolder per technology. Each technology knows which files to create and which dependencies to install.

```
src/
  brick/
    domain/           # business types + ports (interfaces) — pure TS, zero framework
    application/      # BrickApplicationService — orchestration only
    infrastructure/
      primary/        # Next.js API routes + container (composition root)
      secondary/      # InMemoryBrickRegistry, FileSystemProjectWriter

  catalog/
    <name>/
      domain/         # <Name>BrickBuilder implements BrickBuilder
      infrastructure/primary/  # <Name>BrickDefinition (catalogue registration)

  shared/
    domain/           # Value Objects: BrickSlug, BrickVersion, ProjectFolder, PackageName
```

**Absolute rule:** `brick/domain/` and `catalog/*/domain/` must have zero imports from Next.js, `fs`, or any framework/Node module. Fully testable in isolation.

## Vocabulary

| Term              | Role                                                           |
| ----------------- | -------------------------------------------------------------- |
| `BrickSlug`       | Kebab-case identifier of a brick, e.g. `"express"`             |
| `BrickVersion`    | Version chosen by the user, e.g. `"5.x"`                       |
| `BrickContext`    | What the user configures (folder, version, package manager...) |
| `Patch`           | What a brick produces (files, npm dependencies, scripts)       |
| `BrickDefinition` | Description of a brick in the catalogue                        |
| `BrickBuilder`    | Interface (port): `build(context): Patch`                      |
| `BrickRegistry`   | Port: access to the catalogue                                  |
| `ProjectWriter`   | Port: writes a `Patch` to the filesystem                       |

## Conventions

- **Value Objects**: private constructor, static factory `of()`, validation at construction, immutable.
- **Ports**: TypeScript interfaces in `brick/domain/`. Implementations live in `infrastructure/secondary/` or `catalog/`.
- **Tests**: `*.test.ts` files co-located with the source. One test = one observable behaviour, not an implementation.
- **TDD**: Red → Green → Refactor. At each refactor, ask: _"What does this code depend on that it doesn't need to know?"_
- **Commits**: conventional commits (commitlint configured).

## Code Quality Rules

### Domain purity

- `brick/domain/` and `catalog/*/domain/` have zero framework imports — no `fs`, `path`, `next`, or any Node/framework module.
- If a test needs a mock, that is a signal that an interface (port) is missing.
- Domain types must express the ubiquitous language: prefer `BrickSlug` over `string`, `ProjectFolder` over `string`.

### Value Objects

- Always private constructor + static `of()` factory.
- Validation at construction — throw immediately if the value is invalid, never let an invalid state exist.
- Immutable: no setters, no mutation after construction.
- Equality based on value, not identity.

### Tests

- Test observable behaviours, not implementations.
- A test must never assert that a method was called — only what it produced.
- Exception: asserting that `ProjectWriter.apply` was called in `BrickApplicationService` is legitimate because it is the observable side-effect of the service.
- Naming: `it('should <expected behaviour> when <condition>', ...)`

### Dependencies

- Depend on abstractions (ports/interfaces), not on concrete implementations.
- Concrete classes are wired only at the composition root (`container.ts`).
- No `new ConcreteClass()` inside domain or application code — receive dependencies as constructor parameters.

## Adding a brick

See the `add-brick` skill: `/add-brick`
