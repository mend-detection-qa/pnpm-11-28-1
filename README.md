# dev-optional-git-dep-index-fallback

Probe targeting pnpm 11.28.1 features: `index.fallback` lockfile
field, `--dev` flag installing optional deps of devDeps, and
improved git-hosted dependency handling.

## Pattern

`dev-optional-git-dep-index-fallback`

Categories: `lockfile_format`, `tree_structure`, `install_command`

## Feature exercised

This probe exercises three pnpm 11.28.1 behaviors simultaneously:

1. **`index.fallback` lockfile field** — A new `index.fallback: true`
   top-level field is present in `pnpm-lock.yaml`. The UA YAML parser
   must tolerate unknown fields without throwing. A parser that rejects
   unknown fields will produce zero results for the entire scan.

2. **`--dev` installs optional deps of devDependencies** — `vitest`
   is declared as a `devDependency`. `vitest`'s own snapshot entry in
   the lockfile lists `fsevents@2.3.3` under `optionalDependencies`.
   pnpm 11.28.1's `--dev` flag guarantees this optional dep is
   installed and recorded. Mend must surface `fsevents` in the tree
   with `group: "dev"` and `optional: true`.

3. **Git-hosted dependency** — `debug` is declared as
   `github:debug-js/debug#3.2.7`. The lockfile records the resolution
   as `{git: 'https://github.com/debug-js/debug', commit: <sha>}`.
   Mend must report `source: "git"` with the commit SHA in
   `source_detail.commit`, not `source: "registry"`.

## Dependencies used

| Package | Version | Group | Source | Notes |
|---------|---------|-------|--------|-------|
| `debug` | 3.2.7 | main | git | git-hosted; commit SHA in lockfile |
| `zod` | 3.22.4 | main | registry | baseline registry dep |
| `ms` | 2.1.3 | main | registry | transitive of debug |
| `vitest` | 1.6.0 | dev | registry | devDep with optional sub-dep |
| `fsevents` | 2.3.3 | dev | registry | optional dep OF vitest; darwin-only |
| `@types/node` | 20.14.2 | dev | registry | peer dep of vitest |
| `@vitest/expect` | 1.6.0 | dev | registry | transitive of vitest |
| `@vitest/snapshot` | 1.6.0 | dev | registry | transitive of vitest |
| `@vitest/spy` | 1.6.0 | dev | registry | transitive of vitest |
| `@vitest/utils` | 1.6.0 | dev | registry | transitive of vitest |

## Expected dependency tree

- Root direct deps: `debug`, `zod` (main); `vitest` (dev);
  `fsevents` (optional/dev, hoisted from vitest's optionalDeps)
- `debug` → `source: "git"`, `source_detail.commit` populated
- `fsevents` → `optional: true`, `group: "dev"`
- All vitest transitive deps present under vitest's dep list
- `ms` present as transitive of `debug`

## Mend config

**Bucket A** — `js-pnpm` has no dynamic version detection from the
manifest. This probe ships `.whitesource` pinning:

```json
{
  "scanSettings": {
    "configMode": "AUTO",
    "versioning": {
      "pnpm": "11.28.1",
      "node": "20.14.2"
    }
  }
}
```

No `whitesource.config` is present; `configMode` is `AUTO`. The
`strict-ssl=false` line in `.npmrc` exercises the pnpm 11.28.1
install-command change for TLS behavior during git-protocol cloning
— it is an install-time hint only and does not affect the detected
dependency tree shape.

## Failure modes this probe targets

| Failure | Expected signal |
|---------|----------------|
| `index.fallback` field causes parse error | Zero deps reported |
| `fsevents` absent from tree | Optional dep of devDep not followed |
| `fsevents` present but `optional: false` | Flag not propagated |
| `debug` reported as `source: "registry"` | Git resolution lost |
| `debug` missing `source_detail.commit` | Commit SHA not captured |

## Probe metadata

```
pm: pnpm
pm_version_tested: 11.28.1
pattern: dev-optional-git-dep-index-fallback
generated_at: 2026-09-28T08:07:15Z
resolver_sha: 351915ae1d53c6db20db0a5d65b637fe7899d7ea
resolver_fetched_at: 2026-09-28T08:06:01Z
```
