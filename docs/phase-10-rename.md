# Phase 10 — Rename (give the finalized repo its real identity)

> **This guide IS the complete renaming playbook** — self-contained, commit-verified:
> extracted from the actual rename of the reference build
> ([`the777ist/platform`](https://github.com/the777ist/platform)) to `the777incident`
> (its commits `fa2eb9f` → `eea8a91` → `114a0ed`, 117 file changes), validated by running
> it in REVERSE (reproducing the pre-rename tree **byte-exactly**), and re-proven by a
> fresh-clone end-to-end test run (`sevenfold`) that passed every local gate plus remote
> CI and the dispatched nightly E2E/VR on the renamed state. When this phase completes,
> ALL rename scaffolding goes — `/implement` and this guide (git history keeps them; the
> procedure is an involution, so the historical guide covers a later rebrand or reversal).

**Goal:** Swap the generic template identity — repo name `platform`, org placeholder
`example`, package scope `@platform/*` — for the USER-SUPPLIED real identity, layer by
layer with full gates; then strip the last build scaffolding (`/implement` + this guide),
leaving the repo fully yours with zero template artifacts.

Throughout, `<repo>` = your new identity (worked example: `the777incident`). One identity
serves as repo name, org, and scope by default; if yours differ, substitute per layer.

---

## Prerequisites

- **Phase 9 complete** (finalized). If build scaffolding beyond `/implement` + this guide
  is still present, STOP — finalize first (renaming mid-build invalidates the guides'
  literal skeletons).
- **The new identity, from the user**: `/implement 10 <new-identity> [org=<org>]
  [scope=<scope>]`. **Required — never inferred or invented.** If missing, STOP and ask.
- **Name validation:** every chosen value word-safe (kebab/lowercase, no spaces) and NOT
  equal to or containing any fourth-layer token below (`template`, `demo`, `example`,
  `platform`) — the stamper and the audits depend on those staying distinct.
- **Clean tree** (`git status` clean) and the local Supabase stacks **stopped**
  (constraint 1 below).
- **Explicit user confirmation**: state the resolved identity per layer and that this
  rewrites the repo's identity everywhere. Do not proceed on inference.

---

## The identity model

| Layer            | Generic value                          | Renamed to  | Reference commit |
| ---------------- | -------------------------------------- | ----------- | ---------------- |
| 1. Repo identity | `Cross-Platform Template` / `platform` | `<repo>`    | `fa2eb9f`        |
| 2. Org           | `example` (placeholder)                | `<repo>`    | `eea8a91`        |
| 3. Package scope | `@platform/*`                          | `@<repo>/*` | `114a0ed`        |

**The fourth layer is machinery — NEVER rename it:**

- **The `template` product token + `products/_template`** — the generator's
  find-and-replace mold (`pnpm new-product blog` rewrites whole-word `template` → `blog`).
  The token must NEVER equal the org/repo name: the stamper rewrites EVERY occurrence of
  the token, so it would mangle the org names too (`<repo>-blog-blog-api-stg`,
  `com.blog.blog`…).
- **Brand modes `template` | `demo`** — per-PRODUCT by design (Figma modes ARE brand
  modes); every stamped product gets its own mode named after it.
- **Workflow paths/filters** (`products/_template/**`, `*template-*`) — they point at the
  mold directory and product-token package names.
- **`products/demo`'s `demo` tokens** — it's a stamp; only its ORG half renames (layer 2),
  applied identically to `_template` and `demo` (stamp invariant below).
- **`TODO-*` ids** (EAS project id, Figma file key/mode ids, Supabase URLs, DSNs) — real
  external accounts that only exist on infra day. After the rename,
  `git grep -inE 'TODO'` is the swap-point audit.

## Non-negotiable constraints (each one bit us or was proven in the real run)

1. **Stop the local Supabase stacks BEFORE touching `supabase/config.toml`**
   (`supabase stop` in `products/_template` AND `products/demo`) — the CLI resolves
   containers by the CURRENT `project_id`; change it first and the old containers/volumes
   orphan. Restart after; fresh volumes/DBs are re-migrated + seeded by the E2E run.
2. **Exact-count, per-file replacements — never a blind repo-wide sed.** Assert the
   expected occurrence count per string per file (a script that dies on mismatch). The
   keep-list below is exactly what a blind sed corrupts.
3. **Whole-word tokens only.** Rewrites (yours and the generator's) cannot see into longer
   identifiers — `template_api_rls_test` survived a stamp untouched once and collided on
   CI's shared Postgres. Derived names in `_template` keep the token word-delimited
   (`"template_api" + "_suffix"`). Audit with SUBSTRING grep
   (`git grep -i template products/<name>`), never just `-iw`.
4. **Stamp invariant.** Every product-file edit lands in `_template` AND `demo`
   identically; verify each pair is byte-identical modulo the token rewrite — EXCEPT the
   generator's own port math (`supabase/config.toml` 543xx block = `54321+100·portIndex`;
   `api/package.json` dev script `--port 8000+10·portIndex`), which legitimately differs.

   ```python
   import re
   def stamp(s):  # mirror of the generator's whole-word rewrite
       s = re.sub(r"\btemplate_api\b", "demo_api", s)
       s = re.sub(r"products/_template\b", "products/demo", s)
       s = re.sub(r"\bTemplate\b", "Demo", s)
       return re.sub(r"\btemplate\b", "demo", s)
   # stamp(template_file) == demo_file  (config.toml / api dev-script ports excluded)
   ```

5. **Re-run prettier AFTER EACH layer's replacements** (`pnpm run format`), not once at
   the end. Markdown tables pad to their widest cell; a different-length name changes
   widths and a pure string swap leaves stale padding that `format:check` fails. Found by
   the reverse-run test — it was the ONLY difference from byte-exactness. WHICH layer
   drifts depends on the new name's length vs the old string in each table (the
   `sevenfold` run drifted only at layer 3) — and the ship model requires every layer's
   PR to be CI-green on its own, so format each layer before committing it.
6. **Generated files are never hand-edited.** `pnpm-lock.yaml` regenerates via
   `pnpm install` (run it after the scope layer — the workspace names live in it). The
   api-client `src/` + `openapi.json` regenerate via typegen; only the api-client's own
   `package.json`/`tsconfig.json` are yours to edit. After the scope rename, "drift" on
   api-client means changes under `src/` or `openapi.json` — your config edits are not
   drift.
7. **TOML keys literally named `template`** (supabase `config.toml` `[auth.sms]` /
   `[auth.mfa.phone]` message template) are config-schema names, not product tokens — the
   generator masks them; your edits must not touch them either.

## The keep-list — strings a blind replace corrupts (verified every one)

- `@example.com` / `example.test` — RFC-reserved fixture domains in tests/e2e specs
- `.env.example` — filenames (product CLAUDE/README, generator skip-list comments)
- Code Connect's `example:` — the Figma SDK's API property in `*.figma.tsx`
- swagger "View examples" text in the GENERATED api-client — never hand-edited
- `.npmrc` `registry.example.com` sample comment; supabase config's commented Clerk domain
- the English words "example"/"template" in prose; `snapshotPathTemplate` (Playwright API)
- TOML `template =` keys (constraint 7)
- **this guide itself** (`docs/phase-10-rename.md`) — it documents the generic identity
  and the worked example. Exclude it from every layer's replacements AND from the
  residual audits (`':!docs/phase-10-rename.md'`), or layer 3 corrupts the procedure
  mid-run and the 239-occurrence count won't reproduce. (It is deleted at Step 8 anyway.)
- the word `platform` outside the identity spots: generic prose ("each platform's native
  store", PHILOSOPHY's "platform/template monorepo"), APIs (`process.platform`,
  Playwright's `{platform}` token), and the `PATCH(platform)` tag inside
  `patches/*.patch` (patch content is hash-pinned by `patchedDependencies` — renaming
  the tag is pure lockfile churn)

---

## Build steps

### Step 1 — Inventory the tree

Build the per-layer, per-file replacement table from the layer sections below, then
verify it against the CURRENT tree with an inventory grep — the repo may have drifted
since these counts were recorded; **the tree wins** (the tables say where to look, your
grep says the true counts). Classify every hit against the keep-list before replacing.

### Step 2 — Layer 1: repo identity (6 files, 7 lines — reference commit `fa2eb9f`)

| File                   | Change                                                                                                                                                                                       |
| ---------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `README.md`            | title `# Cross-Platform Template` → `# <repo>`                                                                                                                                               |
| `package.json`         | `"name": "platform"` → `"name": "<repo>"` (nothing filters on it; keep the `packageManager` field — eas-cli workspace detection)                                                             |
| `CLAUDE.md`            | header `platform monorepo` → `<repo> monorepo`                                                                                                                                               |
| `PHILOSOPHY.md`        | title gains the repo name                                                                                                                                                                    |
| `products/*/README.md` | "in the platform monorepo" → "in the `<repo>` monorepo" — `_template` AND `demo` identically (stamp-invariant pair). MISSED by the original `fa2eb9f` run; found by the `sevenfold` test run |

Exact-count replacements (script dies on mismatch), `pnpm run format`, commit — and
**verify the commit landed** (`git log`) before continuing (see the hook-abort trap in
Gotchas).

### Step 3 — Layer 2: bake the org (29 files, 80 lines — reference commit `eea8a91`)

Per product — `products/_template` AND `products/demo`, identically:

| File                                               | What carries the org                                                                      |
| -------------------------------------------------- | ----------------------------------------------------------------------------------------- |
| `api/fly.staging.toml` + `api/fly.production.toml` | `app = "<org>-<product>-api-stg\|prod"`                                                   |
| `api/src/<module>/tasks.py`                        | 4× fly-run docstring app names                                                            |
| `app/app.config.ts`                                | `bundleIdentifier` + `package` (`com.<org>.<product>`), Sentry `organization` + `project` |
| `app/.env.staging` / `.env.production`             | API URL value + its comment (2 each)                                                      |
| `app/.maestro/login.yaml`                          | `appId`                                                                                   |
| `desktop/electron-builder.yml`                     | `appId`, `copyright`, publish `owner`                                                     |
| `supabase/config.toml`                             | `project_id` (+ its comment) — STACKS STOPPED FIRST                                       |
| `CLAUDE.md`                                        | 6 spots: desktop appId, releases repo, project_id, Fly/Supabase/Sentry names, org note    |
| `.claude/commands/release.md`                      | desktop-releases repo owner                                                               |

Root level:

- `README.md` — the create-a-product infra checklist (3 `<org>` mentions)
- `CLAUDE.md` — the naming-convention line
- `PHILOSOPHY.md` — 11 spots: naming-conventions header block, keep-placeholders sentence,
  key ruling #3 (desktop-releases), directory-tree annotations (project_id, bundle ids,
  appId, publish target, fly app), generator checklist spec, the naming-audit verification
  item (post-rename it becomes `git grep TODO`), the multi-product infra-naming line
- `.github/workflows/deploy-api.yml` — the org comment
- `.claude/commands/ptfm-product.md` + `.claude/commands/release.md` — infra-name mentions
- **`scripts/new-product.mjs`** — `const org = "..."` + checklist wording (this is what
  makes every FUTURE stamp come out under the real org with a matching checklist)
- **`scripts/remove-product.mjs`** — `const org = "..."` + the hardcoded
  `com.supabase.cli.project=<org>-${name}` docker-volume hint

Same discipline: exact counts, stamp invariant, format, commit, verify landed.

### Step 4 — Layer 3: package scope (239 occurrences across 83 files + lockfile — reference commit `114a0ed`)

Method: uniform string replace `@platform` → `@<repo>` in every tracked file EXCEPT
`pnpm-lock.yaml` AND this guide (see keep-list), then `pnpm install` to regenerate the
lockfile. The one string uniformly covers every context it hides in — the full
hiding-spot list from the reference commit:

- 11 package.json `name`s + all `workspace:*` dependency entries
- every TS/TSX import in `packages/*` and both products' app/feature/route files
- `tsconfig.json` `extends` (`@<repo>/config/tsconfig/expo`) — 5 files
- `tailwind.config.js` preset `require`s AND the content-glob
  `require.resolve("@<repo>/ui/package.json")` — ui + both apps
- `packages/ui/jest.config.js` — the `transformIgnorePatterns` REGEX (`@<repo>/.*`)
- `.github/workflows/e2e-nightly.yml` — 4 `pnpm --filter @<repo>/...` lines
- root `eslint.config.mjs` (re-exports `@<repo>/config/eslint`) + `lefthook.yml` comment
- `packages/config/tailwind-preset.cjs`, `packages/core/src/api.ts`,
  `products/*/desktop/turbo.json` — scope in comments
- `packages/ui/.storybook/visual-regression.spec.ts` — command strings in comments/errors
- all docs: root + product README/CLAUDE.md, `packages/ui` CLAUDE.md, every ptfm command,
  the thin commands (`add-component`, `dev`, `add-feature`)

Semantic follow-up: reword the root `CLAUDE.md` naming line — the org prefix now derives
from the repo, but the PRODUCT segment still always derives from the product name.

The generator is scope-agnostic (it rewrites the product token INSIDE package names), so
stamps come out `@<repo>/<name>-app` automatically — proven by the first post-rename stamp
(`stream` → `@the777incident/stream-app`, `the777incident-stream-api-stg`, clean sweep).

`pnpm install`, format, commit, verify landed.

### Step 5 — Verify: every gate, uncached (as actually run)

```bash
supabase start                      # both products — new project_ids, fresh volumes
pnpm run format:check
pnpm turbo run lint typecheck build openapi --force
pnpm turbo run test --filter='!@<repo>/template-api' --filter='!@<repo>/demo-api' --force
git status --porcelain products/*/api-client/src products/*/api/openapi.json  # real drift only
# E2E prerequisite the original run had ambiently: each product needs its machine-local
# (gitignored) api/.env — the API webServer boots with cwd api/ so pydantic-settings
# reads it (CI provides env vars instead). On a fresh clone, build it from
# products/<p>/.env.example with the product's OWN ports (DB 54322+100·portIndex — the
# local pooler is disabled, use the direct port for BOTH URLs; SUPABASE_URL
# 54321+100·portIndex) and the local stack's SERVICE_ROLE_KEY + JWT secret
# (`supabase status`), or the E2E dies with "config.webServer was not able to start"
# on missing database_url/database_migration_url. OMIT keys you have no value for —
# do NOT copy `SUPABASE_JWKS_URL=` (empty) verbatim from .env.example: pydantic reads
# it as "" (not None), the API then builds PyJWKClient("") and the JWKS PRIMARY auth
# path silently dies, surfacing as 401 "The specified alg value is not allowed" on
# every authed call (the HS256 fallback rejecting the ES256 token).
cd products/_template/app && CI=1 pnpm exec playwright test     # full-stack E2E
cd products/demo/app      && CI=1 pnpm exec playwright test     # full-stack E2E (stamp)
# The api pytest suites read TEST_DATABASE_URL and default to localhost:5432 (the CI
# service-container port mapping). Locally each product's DB is on its own port
# (54322 + 100·portIndex) and 5432 may be a FOREIGN Postgres — one turbo invocation
# cannot carry two URLs, so run the api tests per product. ORDER MATTERS: run them
# AFTER the E2E — pytest's create_all() tolerates the alembic-built schema, but
# alembic (the E2E's migrate step) dies with DuplicateTable on a create_all-built one
# (`supabase db reset` in the product dir un-wedges a contaminated stack DB).
TEST_DATABASE_URL=postgresql+psycopg://postgres:postgres@127.0.0.1:54322/postgres \
  pnpm turbo run test --filter=@<repo>/template-api --force
TEST_DATABASE_URL=postgresql+psycopg://postgres:postgres@127.0.0.1:54422/postgres \
  pnpm turbo run test --filter=@<repo>/demo-api --force
pnpm --filter @<repo>/ui build-storybook                        # VR serves storybook-static —
cd packages/ui            && pnpm exec playwright test          #   build it first (as CI does),
                                                                #   else webServer times out
git grep -in '<old-tokens>' -- ':!pnpm-lock.yaml' ':!docs/phase-10-rename.md'  # residual → keep-list only
```

Plus the stamp-invariant script (constraint 4) over every changed product-file pair.

### Step 6 — Residual audits + stamp round-trip

Residual greps: every hit must be on the keep-list. Then prove the generator under the
new identity: stamp a throwaway product, audit it with the SUBSTRING grep (constraint 3),
remove it with `pnpm remove-product <name> --yes`.

### Step 7 — Ship

One PR per layer (identity → org → scope), each CI-green before the next — diffs stay
reviewable and failures isolate to their layer (sequential commits when no remote
exists). After merging: the main-push deploy workflows skip green (secret-gated until
infra day); dispatch `e2e-nightly` once to prove the full nightly path on the renamed
state.

### Step 8 — Strip the last scaffolding

`rm .claude/commands/implement.md docs/phase-10-rename.md` — this phase removes the
command that runs it and its own guide (the same self-removing pattern as Phase 9;
everything is recoverable from git history). Remove the README's rename-step mention —
the step is done and the pointer would dangle. Commit.

---

## Verification

- Every gate in Step 5 green, with real output shown.
- Residual audits return keep-list hits only.
- The stamp round-trip came out clean under the new identity (zero old-token residuals in
  the stamped tree; `remove-product` restored a clean tree).
- `ls docs/phase-*.md` returns nothing; `.claude/commands/implement.md` gone; no dangling
  rename references in the README.

## Definition of done

- [ ] User supplied the identity and confirmed the destructive run.
- [ ] Three layer commits (identity → org → scope), each format-clean and verified landed.
- [ ] All gates green; residual audits keep-list-only; stamp round-trip proven.
- [ ] Last scaffolding stripped (`/implement`, this guide, the README mentions).
- [ ] Report: layers applied, per-gate results with evidence, anything stopped on —
      honestly (no claiming done over a failed gate).

## Commits

One commit per layer, plus a final
`chore: strip the last build scaffolding (phase 10 complete)`.

## Gotchas & pitfalls

- **No blind repo-wide replaces, ever.** The constraints and keep-list win over any
  instinct to "just sed the repo" — the keep-list is exactly what a blind sed corrupts.
- **Never rename the fourth layer** (product token, `products/_template`, brand modes,
  workflow filters, `TODO-*` ids) — machinery, not identity.
- **The hook-abort trap:** lefthook runs prettier+eslint pre-commit, and a broken hook
  environment (e.g. an untrusted `mise.toml` on a fresh clone — run `mise trust` first)
  aborts the commit while a chained follow-up command happily keeps going — the next
  layer then silently amends/mixes into the wrong commit. Hence "verify landed" after
  every layer.
- **It removes the command that runs it** — do Step 8's deletions last, after all gates
  are green.
- **Post-rename symptoms that are NOT bugs:** VS Code Tailwind IntelliSense "can't
  resolve `@<repo>/config/tailwind-preset`" (its language server caches module resolution
  from before the rename — `Developer: Reload Window`; Node itself resolves fine:
  `node -e "require.resolve('@<repo>/config/tailwind-preset',{paths:['packages/ui']})"`).
  Docker cruft — old volumes remain under the previous project ids
  (`docker volume ls --filter label=com.supabase.cli.project=<old-org>-template`; safe to
  `docker volume rm`). Turbo cache — new package names = new hashes; the first gate run
  is fully uncached.
- **New products post-rename** — `pnpm new-product <name>` stamps correctly under the new
  identity, but remember its checklist's workflow item: `deploy-api.yml`/`eas-update.yml`
  enumerate products explicitly in their `changes` filters — add the new product's
  entries or main pushes never deploy it.

## Reversibility

The whole procedure is an involution: run it with OLD/NEW swapped and it restores the
generic identity — proven by reverse-applying it and reproducing the pre-rename tree
byte-exactly. This guide is deleted at Step 8 — recover it from git history for a later
rebrand or reversal.
