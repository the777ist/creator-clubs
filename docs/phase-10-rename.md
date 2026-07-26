# Phase 10 — Rename (give the finalized repo its real identity)

> Authoritative playbook: [`RENAME.md`](../RENAME.md) (repo root) — commit-verified in the
> reference build and re-proven end-to-end on a fresh clone (every local gate + remote CI +
> nightly E2E/VR on the renamed state). **Read it in full, every run, before touching
> anything.** This guide wraps it in the phase workflow; RENAME.md is the contract for
> *what* changes and *what must never change*. Unlike every other guide, RENAME.md
> **survives** this phase — renames are an involution (run with old/new swapped to rename
> back), so the playbook stays as runtime surface for future rebrands.

**Goal:** Swap the generic template identity — repo name `platform`, org placeholder
`example`, package scope `@platform/*` — for the USER-SUPPLIED real identity, layer by
layer with full gates; then strip the last build scaffolding (`/implement` + this guide),
leaving the repo fully yours.

---

## Prerequisites

- **Phase 9 complete** (finalized). If build scaffolding beyond `/implement` + this guide
  is still present, STOP — finalize first (renaming mid-build invalidates the guides'
  literal skeletons).
- **The new identity, from the user**: `/implement 10 <new-identity> [org=<org>]
  [scope=<scope>]`. One identity serves as repo name, org, and scope by default; the
  optional tokens override those layers individually. **Required — never inferred or
  invented.** If missing, STOP and ask.
- **Name validation:** every chosen value word-safe (kebab/lowercase, no spaces) and NOT
  equal to or containing any fourth-layer token from RENAME.md's identity model (the
  `template` product token, the `demo` stamp, the `example` org placeholder, the generic
  `platform` name) — the stamper and the audits depend on those staying distinct.
- **Clean tree** (`git status` clean) and the local Supabase stacks **stopped**
  (RENAME.md constraint 1).
- **Explicit user confirmation**: state the resolved identity per layer and that this
  rewrites the repo's identity everywhere. Do not proceed on inference.

---

## Build steps

### Step 1 — Read the playbook; inventory the tree

Read `RENAME.md` in full. Build the per-layer, per-file replacement table from its Layer
1/2/3 sections, then verify it against the CURRENT tree with an inventory grep — the repo
may have drifted since the playbook's counts were recorded; **the tree wins** (RENAME.md's
file lists say where to look, your grep says the true counts).

### Step 2 — Layer 1: repo identity

Exact-count per-file replacements via a script that dies on mismatch (constraint 2), then
`pnpm run format` (constraint 5), then commit — and **verify the commit landed**
(`git log`) before continuing (see RENAME.md's Ship section for the hook-abort trap).

### Step 3 — Layer 2: bake the org

Same discipline, `products/_template` AND `products/demo` identically (stamp invariant,
constraint 4). Format, commit, verify landed.

### Step 4 — Layer 3: package scope

Uniform scope replace per the playbook (respecting its exclusions), then `pnpm install`
to regenerate the lockfile (constraint 6), format, commit, verify landed.

### Step 5 — Run every verification gate

RENAME.md's **Verify** section, uncached, in its stated order — E2E before the api pytest
suites; note the machine-local `api/.env` and `TEST_DATABASE_URL` prerequisites it
documents, and `build-storybook` before VR.

### Step 6 — Residual audits + stamp round-trip

Residual greps: every hit must be on RENAME.md's keep-list. Then prove the generator under
the new identity: stamp a throwaway product, audit it with the SUBSTRING grep, remove it
with `pnpm remove-product <name> --yes`.

### Step 7 — Strip the last scaffolding

`rm .claude/commands/implement.md docs/phase-10-rename.md` — this phase removes the
command that runs it and its own guide (the same pattern as Phase 9). Rewrite the
README's rename-step mention to its post-rename form (the step is done; keep the
`RENAME.md` pointer — it documents reversal). **Keep `RENAME.md`.** Commit.

---

## Verification

- Every gate in RENAME.md's Verify section green, with real output shown.
- Residual audits return keep-list hits only.
- The stamp round-trip came out clean under the new identity (zero old-token residuals in
  the stamped tree; `remove-product` restored a clean tree).
- `ls docs/phase-*.md` returns nothing; `.claude/commands/implement.md` gone;
  `RENAME.md` present.

## Definition of done

- [ ] User supplied the identity and confirmed the destructive run.
- [ ] Three layer commits (identity → org → scope), each format-clean and verified landed.
- [ ] All gates green; residual audits keep-list-only; stamp round-trip proven.
- [ ] Last scaffolding stripped (`/implement` + this guide); `RENAME.md` kept.
- [ ] Report: layers applied, per-gate results with evidence, anything stopped on —
      honestly (no claiming done over a failed gate).

## Commits

One commit per layer (RENAME.md's Ship section: one PR per layer when a remote exists —
diffs stay reviewable, failures isolate), plus a final
`chore: strip the last build scaffolding (phase 10 complete)`.

## Gotchas & pitfalls

- **No blind repo-wide replaces, ever.** RENAME.md's constraints and keep-list win over
  any instinct to "just sed the repo" — the keep-list is exactly what a blind sed
  corrupts.
- **Never rename the fourth layer** (product token, `products/_template`, brand modes,
  workflow filters, `TODO-*` ids) — machinery, not identity.
- **Generated files are never hand-edited** (`pnpm-lock.yaml`, api-client `src/`,
  `openapi.json`) — they regenerate.
- **It removes the command that runs it** — do Step 7's deletions last, after all gates
  are green.
