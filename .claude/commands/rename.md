---
description: Rename the repo from its generic template identity to its real identity (repo name, org, package scope) by executing the commit-verified RENAME.md playbook.
argument-hint: "<new-identity> [org=<org>] [scope=<scope>]"
---

You are the **executing agent** for the repo-identity rename. The complete, verified
procedure lives in [`RENAME.md`](../../RENAME.md) — read it **in full, every run, before
touching anything**. This command tells you how to drive it; RENAME.md is the contract
for *what* changes and *what must never change*.

## Arguments

Raw args: `$ARGUMENTS`

- The first token is the **new identity** — required. If missing, STOP and ask the user
  for it (never invent or infer a name).
- One identity serves as repo name, org, and package scope by default. Optional
  `org=<value>` / `scope=<value>` tokens override those layers individually.
- Any remaining text is user instructions for this run (they do not override RENAME.md's
  non-negotiable constraints or keep-list).

## Guards — verify BEFORE starting, stop on any failure

1. **Timing:** this runs on a FINALIZED template (Phase 9 done) or any later state. If
   build scaffolding (`docs/phase-*.md`, `/implement`) is still present, warn the user:
   renaming mid-build invalidates the guides' literal skeletons.
2. **Name validation:** every chosen value must be word-safe (kebab/lowercase, no spaces)
   and must NOT equal or contain any fourth-layer token from RENAME.md's identity model
   (the product token, the stamp name, the org placeholder, the generic repo name) — the
   stamper and the audits depend on those staying distinct.
3. **Clean tree** (`git status` clean) and the local Supabase stacks **stopped**
   (RENAME.md constraint 1).
4. **Confirm with the user** before executing: state the resolved identity per layer and
   that this rewrites the repo's identity everywhere. Do not proceed on inference.

## Procedure

1. Read `RENAME.md` in full. Build the per-layer, per-file replacement table from its
   Layer 1/2/3 sections, then verify it against the CURRENT tree with an inventory grep
   (the repo may have drifted since the playbook's counts were recorded — the tree wins;
   RENAME.md's file lists tell you where to look, your grep tells you the true counts).
2. Execute **one layer at a time**, exactly as RENAME.md prescribes: exact-count
   per-file replacements via a script that dies on mismatch (constraint 2), whole-word
   tokens only (constraint 3), stamp invariant across `_template`/`demo` (constraint 4),
   `pnpm run format` after EACH layer (constraint 5), `pnpm install` after the scope
   layer (constraint 6). Honor the keep-list — it is exactly what a blind replace
   corrupts.
3. **Commit per layer** (identity → org → scope) and verify each commit actually landed
   (`git log`) before starting the next — see RENAME.md's Ship section for the
   hook-abort trap.
4. Run RENAME.md's **Verify** section — every gate, uncached, in its stated order (E2E
   before the api pytest suites; note the machine-local `api/.env` and
   `TEST_DATABASE_URL` prerequisites it documents).
5. Run the residual audits; every hit must be on the keep-list. Then prove the generator:
   stamp a throwaway product, audit it with the substring grep, remove it with
   `pnpm remove-product <name> --yes` (or the manual inverse RENAME.md documents if
   the script is absent).
6. Ship per RENAME.md (one PR per layer when a remote exists; otherwise sequential
   commits) and report: layers applied, per-gate results with real output, residual-audit
   result, and anything you had to stop on. If any gate failed, say so honestly — do not
   claim done.

## Hard rules

- RENAME.md's **non-negotiable constraints and keep-list win** over any instinct to
  "just sed the repo". No blind repo-wide replaces, ever.
- **Never rename the fourth layer** (product token, `products/_template`, brand modes,
  workflow filters, `TODO-*` ids) — it is machinery, not identity.
- Generated files (`pnpm-lock.yaml`, api-client `src/`, `openapi.json`) are never
  hand-edited — they regenerate.
- This playbook is an involution: run with OLD/NEW swapped, it renames back. Mention
  that in the final report so the user knows the rename is reversible.
