# pnpm auto-install-peers installs optional peer dependencies

Minimal reproduction for [pnpm#8142](https://github.com/pnpm/pnpm/issues/8142) and [pnpm#10046](https://github.com/pnpm/pnpm/issues/10046).

## Setup

A monorepo with 3 workspace packages:

- **`fake-orm`** — a library declaring 5 **optional** peer dependencies (`pg`, `mysql2`, `better-sqlite3`, `ioredis`, `chalk`) via `peerDependenciesMeta: { optional: true }`
- **`app-a`** — depends on `fake-orm` and `pg` (one legitimate peer)
- **`app-b`** — depends on `fake-orm` only (provides no peers)

## Reproduction

```bash
pnpm install
```

## Expected behavior

Only `pg` should be installed (the one peer that `app-a` explicitly depends on). The other 4 optional peers (`mysql2`, `better-sqlite3`, `ioredis`, `chalk`) should NOT be installed since:

1. No workspace package depends on them
2. They are marked `optional: true` in `peerDependenciesMeta`
3. [pnpm docs](https://pnpm.io/settings) state: `autoInstallPeers` — *"When true, any missing **non-optional** peer dependencies are automatically installed."*

## Actual behavior

**All 5 optional peers are installed** as dependencies of `fake-orm` in the lockfile:

```yaml
packages/fake-orm:
    dependencies:
      better-sqlite3:    # ← NOT a dependency of any workspace package
        specifier: ^11.0.0 || ^12.0.0
        version: 12.8.0
      chalk:             # ← NOT a dependency of any workspace package
        specifier: ^5.0.0
        version: 5.6.2
      ioredis:           # ← NOT a dependency of any workspace package
        specifier: ^5.0.0
        version: 5.10.1
      mysql2:            # ← NOT a dependency of any workspace package
        specifier: ^3.0.0
        version: 3.20.0(@types/node@25.5.0)
      pg:                # ← Only this one is expected (app-a depends on it)
        specifier: ^8.0.0
        version: 8.20.0
```

76 packages are installed. With `auto-install-peers=false`, only 17 packages are installed (pg and its deps).

## Environment

- pnpm 10.28.0
- Node 24.13.0
- macOS (Darwin arm64)

## Impact

This is especially problematic for packages like `typeorm` which declare ~17 database drivers as optional peers. In a real monorepo, this results in **134+ unnecessary packages** being installed, including native modules like `@sap/hana-client` and `better-sqlite3` that have build steps which can fail on CI.

## Workaround

Use a `.pnpmfile.cjs` hook to strip unwanted optional peers at resolution time:

```js
function readPackage(pkg, context) {
    if (pkg.name === 'fake-orm') {
        const unwanted = ['mysql2', 'better-sqlite3', 'ioredis', 'chalk'];
        for (const peer of unwanted) {
            delete pkg.peerDependencies[peer];
            delete pkg.peerDependenciesMeta[peer];
        }
    }
    return pkg;
}
module.exports = { hooks: { readPackage } };
```
