---
name: migrate-polyrepo-to-monorepo
description: Migrate a customer from N source repos into a Turborepo monorepo with zero traffic disruption.
argument-hint: "<client-slug>"
---

# Migrate polyrepo to monorepo

Drives the end-to-end polyrepo-to-monorepo migration for a customer engagement.
Wraps `packages/scripts/src/migrate-polyrepo-to-monorepo.ts` and the
[Polyrepo-to-monorepo migration template](../../docs/templates/polyrepo-to-monorepo-migration.md).

## Prerequisites

- A `docs/customers/<slug>/migration-inventory.json` file exists. If not,
  create it first:

  ```json
  {
    "monorepoName": "<org>/<monorepo-name>",
    "scope": "@<scope>",
    "repos": [
      {
        "name": "<name>",
        "remote": "https://github.com/<org>/<repo>.git",
        "branch": "main",
        "type": "app",
        "deployTarget": "cloud-run | vercel | github-pages | none"
      }
    ]
  }
  ```

- All source repos have passing CI on their default branch.
- The monorepo skeleton (Phase 2 of the migration template) is committed and
  accessible locally.

## Steps

### 1. Dry-run the migration plan

```bash
bun run packages/scripts/src/migrate-polyrepo-to-monorepo.ts \
  --client <slug> \
  --phase plan \
  --dry-run
```

Review the plan output. It lists:
- Each source repo that will be imported
- The target path (`apps/<name>` or `packages/<name>`)
- The `git subtree add` commands that will be run
- The workspace globs that will be added to `pnpm-workspace.yaml`
- The Turborepo task entries that will be merged into `turbo.json`

Do not proceed until the plan output matches expectations.

### 2. Import repos (one at a time)

```bash
bun run packages/scripts/src/migrate-polyrepo-to-monorepo.ts \
  --client <slug> \
  --phase import \
  --repo <name>
```

Repeat for each repo in the inventory. Each import:
- Adds the source repo as a `git` remote
- Fetches the default branch
- Runs `git subtree add --prefix=<target> <remote>/<branch> --squash`
- Commits the result

After each import, run the parity check:

```bash
pnpm turbo run build lint tsc test --filter=<name>...
```

Fix any failures before importing the next repo.

### 3. Wire workspaces and Turbo tasks

```bash
bun run packages/scripts/src/migrate-polyrepo-to-monorepo.ts \
  --client <slug> \
  --phase wire \
  --dry-run
```

Review the diff, then apply:

```bash
bun run packages/scripts/src/migrate-polyrepo-to-monorepo.ts \
  --client <slug> \
  --phase wire
```

Then install:

```bash
CI=true pnpm install --frozen-lockfile
```

### 4. Full parity verification

```bash
bun run packages/scripts/src/migrate-polyrepo-to-monorepo.ts \
  --client <slug> \
  --phase verify
```

The script prints a checklist of parity steps and runs what it can
automatically. Steps requiring human judgment are flagged for manual
confirmation. The checklist covers:

- `pnpm turbo run build lint tsc test`: all green
- Docker build per app workspace; all exit 0
- Health-endpoint smoke test per service

Get sign-off from the migration lead and one other engineer before proceeding
to cutover.

### 5. Cutover

Cutover is NOT automated. The script emits a prioritized checklist:

```bash
bun run packages/scripts/src/migrate-polyrepo-to-monorepo.ts \
  --client <slug> \
  --phase cutover \
  --dry-run
```

Work through the checklist manually, confirming each step:

1. Migrate CI pipelines (run in parallel for 24 hours before disabling)
2. Migrate deploy targets (one service at a time, monitor 30 min each)
3. Update internal documentation
4. Archive source repos (do not delete)
5. Set a 30-day calendar reminder for decommission review

### 6. Report

Tell the user:
- Which repos were imported and their target paths.
- Whether parity verification passed for all workspaces.
- Cutover checklist status (completed items vs. pending).
- Any deviations from the canonical template and their rationale.

## Rules

- Never run `git push` against any source repository during the migration.
  The migration is read-only from the perspective of source repos.
- Never run `--phase cutover` without parity sign-off from at least two
  engineers.
- If parity fails after import, fix the workspace in the monorepo. Do not
  patch the source repo. The monorepo is the new source of truth from Phase 3
  onward.
- All script phases support `--dry-run`. When in doubt, dry-run first.
