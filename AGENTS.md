# AGENTS.md

This file provides guidance to Coder Agents when working with code in this repository.

## Project Overview

This is a TypeScript library template designed to be cloned/forked for creating new npm packages. It uses the `ts-builds` toolchain for standardized build scripts with ESM output.

**Template Usage**: To apply this pattern to other TypeScript projects, see the canonical standardization guide in [ts-builds](https://github.com/jordanburke/ts-builds/blob/main/STANDARDIZATION_GUIDE.md) (or the **ts-builds** skill). This repo is the starting scaffold; ts-builds owns the standards.

## Toolchain Pins

- **Node.js**: exact version in `.nvmrc` (both workflows read it via `node-version-file`). Pin the patch, never a bare major: setup-node resolves a bare `24` to whatever the runner caches, and the bundled npm decides whether OIDC trusted publishing works (npm 11.17.0 fails with E401; 11.19.0 publishes).
- **pnpm 12**: exact version in the `packageManager` field. Change it with `pnpm self-update <version>`, run `pnpm install`, then commit `package.json` **and** `pnpm-lock.yaml` together. pnpm 12 records its own version in the lockfile, and CI's `pnpm install --frozen-lockfile` fails with `ERR_PNPM_FROZEN_LOCKFILE_WITH_OUTDATED_LOCKFILE` when the lockfile lacks that record.
- **CI pnpm setup**: `pnpm/action-setup@v6.1.0`, pinned exactly. It is the first release that supports pnpm 12, and the floating `v6` tag still points at v6.0.10. Pass no `version:` input; the action reads `packageManager`.
- **pnpm settings** live in `pnpm-workspace.yaml`; pnpm reads only auth/registry from `.npmrc`. pnpm 12 fails with `ERR_PNPM_UNRECOGNIZED_WORKSPACE_SETTINGS` on an unknown key in `pnpm-workspace.yaml` when the running pnpm satisfies the `packageManager` pin (otherwise it only warns).

## Development Commands

All commands delegate to `ts-builds` for consistency across projects:

```bash
pnpm validate        # Main command: format + lint + test + build (use before commits)

pnpm format          # Format code with Prettier
pnpm format:check    # Check formatting only

pnpm lint            # Fix ESLint issues
pnpm lint:check      # Check ESLint issues only

pnpm test            # Run tests once
pnpm test:watch      # Run tests in watch mode
pnpm test:coverage   # Run tests with coverage

pnpm build           # Production build (outputs to dist/)
pnpm dev             # Development build with watch mode

pnpm typecheck       # Check TypeScript types
```

### Running a Single Test

```bash
pnpm test -- --testNamePattern="pattern"    # Filter by test name
pnpm test -- test/specific.spec.ts          # Run specific file
```

## Architecture

### Build System: ts-builds + tsdown

- **ts-builds**: Centralized toolchain package providing all build scripts
- **tsdown**: Underlying bundler configured via `ts-builds/tsdown`
- **Configuration**: `tsdown.config.ts` imports default config from ts-builds
- **TypeScript**: `tsconfig.json` extends `ts-builds/tsconfig`
- **Prettier**: Uses `ts-builds/prettier` shared config

### Output Format

- **dist/**: Production builds containing:
  - `index.js` - ES module format
  - `index.d.ts` - TypeScript declarations
- **lib/**: Development builds (also published)

### Package Exports

```json
{
  "main": "./dist/index.js",
  "module": "./dist/index.js",
  "types": "./dist/index.d.ts",
  "exports": {
    ".": {
      "types": "./dist/index.d.ts",
      "import": "./dist/index.js",
      "default": "./dist/index.js"
    }
  }
}
```

### Testing: Vitest

- Tests located in `test/*.spec.ts`
- Uses Vitest with configuration from ts-builds
- Coverage via v8 provider

## Key Files

- `src/index.ts` - Main library entry point
- `test/*.spec.ts` - Test files
- `tsdown.config.ts` - Build config (imports from ts-builds)
- `tsconfig.json` - TypeScript config (extends ts-builds)
- `.nvmrc` - Exact Node.js version for local use and CI
- `pnpm-workspace.yaml` - pnpm settings (hoisting, release-age exclusions)
- `.github/workflows/node.js.yml` - CI: `pnpm validate` on pushes and PRs to `main`
- `.github/workflows/publish.yml` - Publishes to npm when a `v*` tag is pushed
- `.claude/skills/ts-builds-template/` - Claude Code skill for bootstrapping libraries from this template

## Publishing

Publishing runs in CI, triggered by a version tag:

```bash
npm version patch|minor|major   # bumps package.json, commits, and creates the vX.Y.Z tag
git push --follow-tags          # the tag triggers .github/workflows/publish.yml
```

`publish.yml` runs `pnpm validate` and `npm pack --dry-run`, publishes with npm OIDC trusted publishing and provenance (no npm token), then creates a GitHub release. The `prepublishOnly` hook also runs `pnpm validate`.
