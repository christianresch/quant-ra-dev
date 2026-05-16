# CLAUDE.md — Quant RA Development Project

This is the **development** project for building the Quantitative RA agent. The artifact being built is itself a Claude Code configuration (skills, slash commands, global CLAUDE.md, project template) that the end user will install and use. Do not confuse this CLAUDE.md (governs how Claude helps me *build* the RA) with the RA's own user-level CLAUDE.md (governs how the RA behaves for the end user).

## Roles

- **Developer (me):** builds the RA in Claude Code CLI.
- **End user (my partner):** applied microeconomist; causal inference / impact evaluation; fields include education and labour. Uses the RA in the Claude Desktop **Code tab** (GUI, same engine as CLI). Primary tool is Stata; also works in R.

## What is being built

A **Quantitative RA** (v1): an agent specialized for data exploration, cleaning, merging, sample construction, statistical analysis, and regression-table production. Method-general — handles DiD, IV, RDD, event studies, RCT analysis, and other causal-inference designs.

Out of scope for v1: writing, literature review, slide production, referee work, grant work, figure production, Cowork integration, scheduled tasks, multi-agent patterns. Figures are slated for v2.

## Workflow

The RA's work is a fixed chain from raw data to regression table, with four explicit verification gates where the agent halts and waits for the user to approve a plan document before executing.

```
raw data → data-inventory → cleaning-plan [GATE 1] →
stata-cleaning → dta-to-parquet → merge-plan [GATE 2] →
data-merge → analysis-plan [GATE 3] → sample-construction →
estimation (exploratory, 1 lang) → causal-design-checklist [GATE 4] →
estimation (final, 3 langs) → triplication → regression-table
```

Three things to internalize about this chain:

1. **Gates sit at methodological commitment points**, not at every step. Inventory and conversion are mechanical and don't gate. Cleaning, merge, analysis direction, and final-spec elevation gate because mistakes there cost hours to days downstream.
2. **Exploration is unrestricted within a plan.** Once Gate 3 (the analysis plan) is approved, exploratory and candidate specs run freely. A new plan document is a new gate. Gating is per plan-document, not per spec.
3. **Triplication is downstream of the design check, not parallel to it.** A spec doesn't become `final: true` until Gate 4. The triplication skill refuses to run on non-final specs.

## Verification gate protocol

The gate protocol's canonical home, in production, is the RA's user-level CLAUDE.md. It's described here so the design is clear during development.

At each gate, the gated skill writes `plans/<step>-plan.md` and halts with a one-line summary in the chat. The user reviews the plan (typically by opening it in the Code tab sidebar or her editor) and replies with one of three tokens:

- `approve` — agent proceeds to execution
- `revise <notes>` — agent updates the plan and re-presents
- `reject` — agent stops

The agent parses the first word of the reply as the token. Revision content buried in other text without a leading token is ambiguous; the agent asks rather than guesses.

Plans are version-controlled markdown in `plans/` with front matter recording timestamp, run ID, and hashes of input files. If raw data changes after a plan was approved (file hash differs), the agent re-plans rather than re-executing the stale plan.

An approved plan is the contract. If execution surfaces something the plan didn't anticipate — most commonly, merge diagnostics far outside the expected range — the agent halts and re-asks rather than improvising. The conservative default reflects that domain knowledge often matters at these moments and the agent is unlikely to have it.

## Architecture

Layered configuration:

- **User-level** (`~/.claude/` on her machine): skills, slash commands, global CLAUDE.md, settings. Holds the stable methodological knowledge, the gate protocol, and the skill set.
- **Project template**: skeleton folder structure (including `plans/`, `specs/`, `data/raw/`, `data/clean/`, `data/analysis/`, `results/`), a minimal project CLAUDE.md, `verification.yaml`, and `decisions.md`. Copied once per new research project. Holds only project-specific facts.
- **Plugin packaging**: deferred. Plain user-level files for v1; convert to a plugin once the skill set stabilizes.

Test for layer placement: if the same instruction would appear in three project CLAUDE.md files, it belongs at the user level instead.

Raw data is multi-source by default — `data/raw/` is a directory containing multiple input files, not a single file. `data-inventory` and `cleaning-plan` iterate over its contents; `data-merge` is where they combine.

## Skill inventory (v1)

Twelve user-level skills.

| # | Skill | Role | Gate |
|---|---|---|---|
| 1 | `data-inventory` | Profile each raw dataset; flag outliers, sentinel codes, duplicate keys, suspicious missingness | — |
| 2 | `cleaning-plan` | Write `plans/cleaning-plan.md` | 1 |
| 3 | `stata-cleaning` | Execute cleaning plan in Stata | — |
| 4 | `dta-to-parquet` | Convert `.dta` → `.parquet` preserving types/labels | — |
| 5 | `merge-plan` | Write `plans/merge-plan.md` | 2 |
| 6 | `data-merge` | Execute merge with diagnostics; halt on divergence from plan | — |
| 7 | `analysis-plan` | Formalize her stated plan into specs; surface implicit choices | 3 |
| 8 | `sample-construction` | Apply ordered restrictions; produce waterfall + `sample_def.yaml` + analysis `.parquet` | — |
| 9 | `estimation-runner` | Run one spec in one language; write JSON (+ `.ster` for Stata) | — |
| 10 | `causal-design-checklist` | Method-specific pre-flight; declare triplication vs. duplication | 4 |
| 11 | `triplication` | Dispatch parallel runs; compare against tolerance; PASS/FAIL report | — |
| 12 | `regression-table` | `esttab` wrapper reading Stata stored estimates from passed final specs | — |

Two skills carry conventions worth noting:

- **`analysis-plan`** primary mode is *formalize her stated plan and surface implicit choices* (sample, cluster level, baseline controls, which spec is final). Proposal-from-scratch is a secondary mode used only when she explicitly asks.
- **`sample-construction`** is mechanical — it executes restrictions declared in the analysis plan. No separate gate; the restrictions are covered by Gate 3.

Each skill needs a verifiable success criterion before it ships (see "Development principles" below).

## Language and execution model

- **Cleaning:** Stata is canonical (her primary tool). Output is `.dta`. A standalone conversion step produces `.parquet` for downstream analysis.
- **Exploration and candidate analysis:** one language, her choice (typically Stata or R).
- **Final analysis:** triplicated across R, Stata, and Python (or duplicated R + Stata for estimators without a third-language equivalent — see triplication section).
- **Stata execution:** automated via batch mode (`stata -b do …` or Mac equivalent), the same way `Rscript` and `python` are invoked.
- **Final regression tables:** produced in Stata via `esttab`, reading `.ster` files persisted by the Stata variant of `estimation-runner`. Triplication is a precondition (the table skill refuses if a cited final spec hasn't passed), but the published numbers come from Stata. The asymmetry is deliberate: verification is symmetric across three languages; publication output privileges her primary tool.

## Cross-language verification (triplication)

Triplication runs only on `final: true` specs. The default is `final: false`; the flag is flipped explicitly, typically as part of the design-check plan at Gate 4.

**Supported estimators in v1:** `ols`, `ols_fe` (high-dim fixed effects), `iv_2sls`, `iv_2sls_fe`, `poisson_fe` (PPML). Covers the bulk of applied micro causal-inference work.

**RDD is duplicated, not triplicated**, across R `rdrobust` and Stata `rdrobust` only. No Python equivalent matches defaults. The design-check plan declares per spec whether triplication or duplication applies, based on a documented supported-estimator list. No silent fallback to a Python implementation that doesn't match. The same mechanism generalizes to any future estimator without three-language coverage.

**Stata SE conventions are canonical.** When R, Stata, and Python differ on default degrees-of-freedom adjustments for clustered or robust SEs, R (`fixest`) and Python (`linearmodels`) are configured to match Stata. Eliminates the most common source of spurious failures. Equivalence settings live inside the triplication skill, not per call.

**Results are written as structured JSON**, not parsed from logs. Schema: coefficients, SEs, vcov, N, FE counts, cluster variables, options used, package versions.

**Tolerance:** `1e-6` relative for linear estimators, `1e-4` relative for nonlinear. Project-level `verification.yaml` can override per spec.

**What triplication catches:** implementation bugs, transcription errors, and package-version drift. **What it doesn't catch:** methodological errors (wrong estimator, wrong cluster level, wrong identifying assumption). It is a correctness floor, not a ceiling. Document this honestly in the skill's own description.

## Development principles

Karpathy-adapted, for use while building:

1. **Think before coding.** State assumptions about how she works before encoding them in a skill. If a design choice has multiple reasonable interpretations, surface them rather than picking silently.
2. **Simplicity first.** Minimum viable skill set. No speculative skills "in case she needs them." If a skill would only be invoked once a year, don't ship it in v1.
3. **Surgical changes.** When editing the RA configuration, touch only what's needed. Don't refactor adjacent skills.
4. **Goal-driven execution.** Each skill needs a verifiable success criterion (e.g., "triplication: given a known specification, produces a comparison report and exits cleanly when results agree within tolerance; catches a deliberately injected disagreement"). No skill ships without one.

## Open questions

Carried forward and updated:

- **Data sensitivity perimeter.** How the RA handles IRB-protected or restricted-use admin data. Likely a `settings.json` permission-rules question (e.g., RA never enters directories matching `data/restricted/`). Not blocking, but worth resolving before the RA touches a real project.
- **Collaborator reproducibility.** Whether her coauthors need to reproduce the RA's behavior, or only its outputs. Defaulted to outputs-only; worth confirming with her.
- **Spec file format and location.** YAML in a per-project `specs/` directory is the working assumption. Not formally decided.
- **`regression-table` input contract.** Working assumption: refuses to run unless every cited final spec has passed triplication. Not formally decided.
- **Definition of "v1 done."** End-to-end success criterion for the whole RA, not per skill. From the prior conversation; still not written.

Resolved since the previous version of this file: method-aware vs. method-agnostic triplication is settled (method-aware, via the supported-estimator list and the duplication/triplication declaration in the design-check plan).

## What lives in DECISIONS.md vs. here

`DECISIONS.md`: terse log of irreversible-ish choices, dated, with the reason. Append-only.
`CLAUDE.md` (this file): living description of the current design. Edited freely as the design evolves.

If a decision in `DECISIONS.md` is overturned, add a new entry there rather than editing the old one, and update this file to reflect the new state.
