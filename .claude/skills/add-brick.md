---
name: add-brick
description: Full workflow for adding a new brick to the ts-seed catalogue. Follows the TDD cycle.
---

# Adding a brick to ts-seed

## Before starting

Ask the user:

1. The brick name (kebab-case, e.g. `express`, `prisma`, `vitest`)
2. Available versions (e.g. `4.x`, `5.x`)
3. Required bricks (e.g. `typescript-core` for `express`)
4. What the brick generates: files, npm dependencies, scripts

## Structure to create

```
src/catalog/<name>/
  domain/
    <Name>BrickBuilder.ts       # logic — implements BrickBuilder
    <Name>BrickBuilder.test.ts  # TDD tests
  infrastructure/primary/
    <Name>BrickDefinition.ts    # registration in the catalogue
```

And modify:

```
src/brick/infrastructure/primary/container.ts  # add the definition
```

## Mandatory TDD cycle

### 1. Tests first (`<Name>BrickBuilder.test.ts`)

Test at minimum:

- [ ] Each version generates the correct npm dependency version
- [ ] Expected files are present in the Patch (by `destination`)
- [ ] Expected scripts are present
- [ ] All dependencies are correctly marked `dev: true` or `dev: false`

Test template:

```ts
import { describe, it, expect } from 'vitest'
import { <Name>BrickBuilder } from './<Name>BrickBuilder'
import { BrickVersion } from '../../../shared/domain/BrickVersion'
import { ProjectFolder } from '../../../shared/domain/ProjectFolder'
import type { BrickContext } from '../../../brick/domain/BrickContext'

const makeContext = (version: string): BrickContext => ({
  folder: ProjectFolder.of('/tmp/test'),
  projectName: 'my-app',
  packageManager: 'npm',
  version: BrickVersion.of(version),
})

describe('<Name>BrickBuilder', () => {
  it('should install <pkg> ^X.0.0 for version X.x', () => {
    const patch = new <Name>BrickBuilder().build(makeContext('X.x'))
    const dep = patch.dependencies.find((d) => d.name.value === '<pkg>')
    expect(dep?.version).toBe('^X.0.0')
  })

  it('should generate <file> in files', () => {
    const patch = new <Name>BrickBuilder().build(makeContext('X.x'))
    expect(patch.files.some((f) => f.destination === '<file>')).toBe(true)
  })
})
```

### 2. Verify tests fail

```bash
npm run test:run
```

Expected: FAIL — module not found.

### 3. Implement `<Name>BrickBuilder`

```ts
import { PackageName } from '../../../shared/domain/PackageName'
import type { BrickBuilder } from '../../../brick/domain/BrickBuilder'
import type { BrickContext } from '../../../brick/domain/BrickContext'
import type { Patch } from '../../../brick/domain/Patch'

export class <Name>BrickBuilder implements BrickBuilder {
  build(context: BrickContext): Patch {
    return {
      files: [...],
      dependencies: [...],
      scripts: [...],
    }
  }
}
```

**Rule:** no import of `fs`, `path`, `next`, or any external module. Only `PackageName` and other types from `shared/domain/`.

### 4. Verify tests pass

```bash
npm run test:run
```

Expected: PASS.

### 5. Refactor

Ask: _"What does this code depend on that it doesn't need to know?"_

### 6. Create `<Name>BrickDefinition`

```ts
import { BrickSlug } from '../../../../shared/domain/BrickSlug'
import { BrickVersion } from '../../../../shared/domain/BrickVersion'
import { <Name>BrickBuilder } from '../../domain/<Name>BrickBuilder'
import type { BrickDefinition } from '../../../../brick/domain/BrickDefinition'

export const <name>BrickDefinition: BrickDefinition = {
  slug: BrickSlug.of('<name>'),
  name: '<Human readable name>',
  description: '<short description>',
  availableVersions: [BrickVersion.of('X.x'), BrickVersion.of('Y.x')],
  requiredBricks: [],  // or [BrickSlug.of('typescript-core')]
  builder: new <Name>BrickBuilder(),
}
```

### 7. Register in the container

In `src/brick/infrastructure/primary/container.ts`, add the import and the definition:

```ts
import { <name>BrickDefinition } from '../../../catalog/<name>/infrastructure/primary/<Name>BrickDefinition'

const registry = new InMemoryBrickRegistry([
  typescriptBrickDefinition,
  <name>BrickDefinition,  // add here
])
```

### 8. Verify + commit

```bash
npm run test:run  # all tests must pass
npm run lint
git add src/catalog/<name>/ src/brick/infrastructure/primary/container.ts
git commit -m "feat(catalog): add <name> brick"
```
