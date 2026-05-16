# DECISIONS.md — Quant RA

Append-only log of design decisions. Date each entry. If a decision is overturned, add a new entry; do not edit the old one.

---

## 2026-05-14 — v1 scope: Quantitative RA

**Decision:** v1 is limited to data exploration, cleaning, merging, and statistical analysis. Method-general (DiD, IV, RDD, event studies, RCT analysis, etc.).

**Excluded from v1:** writing, literature review, slide production, referee work, grant work, Cowork-side workflows, scheduled tasks, multi-agent / council patterns.

**Reason:** Tight v1 scope avoids the most common agent failure mode (trying to do everything). Picks tasks that are tedious and verifiable.

---

## 2026-05-14 — End user uses Claude Desktop Code tab, not Cowork

**Decision:** End user (partner) interacts with the RA through the **Code tab** in Claude Desktop, not through Cowork. Developer works in Claude Code CLI.

**Reason:** v1's tasks (running R/Stata/Python, producing tables) require local toolchain access. Cowork is sandboxed and cannot reach local R, Stata, or Python environments. The Code tab gives full Claude Code engine access with a GUI (visual diffs, sidebar sessions), so terminal comfort is not required. Original assumption that Cowork was the user interface was relaxed.

**Consequence:** Cowork integration deferred indefinitely. No Cowork-side skills, scheduled tasks, or pairing protocol in v1.

---

## 2026-05-14 — Layered configuration architecture

**Decision:** Methodological knowledge lives at the **user level** (`~/.claude/`). Project-specific facts live in a **project template** copied once per new project.

**Reason:** Most of what the RA knows (preferred packages, table conventions, DiD checklist, verification rules) is invariant across projects. Copying it into each project creates a maintenance trap. Project-level config holds only what genuinely varies: research question, dataset, sample restrictions, clustering choice.

**Test:** If an instruction would be identical across three project CLAUDE.md files, it belongs at the user level.

**Plugin packaging deferred.** Plain user-level files for v1; convert to a plugin once the skill set stabilizes.

---

## 2026-05-14 — Cross-language triplication for final analyses

**Decision:** Final specifications are run in R, Stata, and Python independently. Each writes structured results; a comparison step flags disagreements above tolerance.

**Scope of triplication:** only **final** specifications. Exploratory and candidate specs run in one language.

**Reason:** Triplication catches implementation bugs, transcription errors, and ecosystem version drift. Triplicating exploration is wasteful and produces noise. Triplicating data cleaning produces spurious disagreement from incidental differences (date parsing, missing codes, encoding) rather than real errors.

**Limitation acknowledged:** triplication does not catch methodological errors (wrong estimator, wrong cluster level, wrong identifying assumption). It is a correctness floor, not a ceiling.

---

## 2026-05-14 — Stata is canonical for data cleaning; `.dta` + `.parquet` for downstream

**Decision:** Cleaning is written in Stata (her primary tool). Stata produces `.dta` as the canonical clean output. A small post-cleaning step converts `.dta` to `.parquet` for R and Python to read during analysis.

**Reason:** She is most fluent in Stata and will trust/debug Stata cleaning code. `.dta` keeps Stata authoritative. `.parquet` gives downstream analysis a fast, typed, language-neutral input without depending on SSC `parquet` (less battle-tested) or forcing R/Python to read `.dta` directly (slower, labels handled imperfectly).

**Rejected alternatives:**
- R/Python read `.dta` directly: simpler but slower at scale and loses type safety.
- Stata writes `.parquet` directly via SSC `parquet`: cleanest in principle but adds a non-core dependency.

---

## 2026-05-14 — Stata execution is automated

**Decision:** Stata is invoked by the agent in batch mode (`stata -b do script.do` or Mac equivalent), the same way R and Python are invoked.

**Reason:** Originally assumed Claude Code could not run Stata. Corrected: Claude Code can call any locally installed binary, including Stata. Manual .do-file handoff would add a human step without benefit for v1.

**Fallback option (not implemented in v1):** a flag to drop back to manual .do-file production for cases where she wants to step through Stata interactively.

---

## 2026-05-14 — Tolerance: two-tier default, single override knob

**Decision:** Default relative tolerance is `1e-6` for linear estimators and `1e-4` for nonlinear (logit, probit, Poisson, IV-GMM, etc.). A project-level `verification.yaml` can override per spec.

**Reason:** A single global threshold is either too strict for nonlinear (constant false alarms from optimizer differences) or too loose for linear (misses real bugs). Two tiers captures the only meaningful distinction without a lookup table per estimator. Override is rarely used in practice — keeps the configuration burden minimal.

---

## 2026-05-14 — Results written as structured data, not parsed from logs

**Decision:** Each language writes specification results to JSON (or equivalent structured format): coefficients, SEs, N, fixed effects, cluster variable, options used. The comparison step reads these files directly.

**Reason:** Log parsing is fragile. Structured output is stable, version-controllable, and unambiguous about which options produced which numbers. Standard tools exist in each ecosystem (`broom::tidy` + `jsonlite` in R; `linearmodels` results dict in Python; `estout` to CSV or `e()` macros via `file write` in Stata).

---

## 2026-05-15 — Workflow has four verification gates

**Decision:** The v1 workflow is a fixed chain with four explicit verification gates, after: (1) the cleaning plan, (2) the merge plan, (3) the analysis plan, (4) the design check.

**Workflow:**

```
raw data → data-inventory → cleaning-plan [GATE 1] →
stata-cleaning → dta-to-parquet → merge-plan [GATE 2] →
data-merge → analysis-plan [GATE 3] → sample-construction →
estimation (exploratory, 1 lang) → causal-design-checklist [GATE 4] →
estimation (final, 3 langs) → triplication → regression-table
```

**Reason:** Research data operations are largely one-shot. A silent bad merge or wrong-sample analysis costs hours to days of downstream waste. The four gates correspond to irreversible-ish methodological commitments where review pays for itself.

---

## 2026-05-15 — Gate protocol lives in user-level CLAUDE.md

**Decision:** The protocol governing how gates work (what counts as a plan, what counts as approval, what happens on divergence) lives once in user-level CLAUDE.md. Each gated skill follows it rather than reimplementing it.

**Reason:** Implementing gate behavior inside each gated skill produces four slightly different gate implementations that drift over time. One protocol, one place to edit.

---

## 2026-05-15 — Approval tokens

**Decision:** Approval at a gate requires the user to type one of:
- `approve` — proceed to execution
- `revise <notes>` — update the plan based on notes and re-present for approval
- `reject` — stop

The agent parses the first word of the reply as the token. Anything after `revise` is treated as revision notes.

**Behavior on ambiguous replies:** if a reply contains revision-like content but doesn't lead with a token (e.g., "this looks mostly good but drop firms under 5 employees"), the agent asks which it is rather than guessing.

**Reason:** Unambiguous, parser-trivial, doesn't depend on natural-language interpretation of "looks good." Token-first parsing avoids ambiguity from approval-like phrases buried in other text.

---

## 2026-05-15 — Plans are version-controlled artifacts with hashed inputs

**Decision:** Each gate writes `plans/<step>-plan.md`. The directory is committed to git. Each plan's front matter records timestamp, run ID, and hashes of input files.

**Consequence:** If raw data changes after a plan was approved (file hash differs), the agent re-plans rather than re-executing the stale plan.

**Reason:** Plans are research artifacts in their own right — they document what was approved against what data. Hashing the inputs lets you tell, post-hoc, whether a plan was approved against the data it ultimately ran on.

---

## 2026-05-15 — Approved plan is the contract; halt on divergence

**Decision:** Once a plan is approved, execution must match it. If execution surfaces something the plan didn't anticipate, the agent halts and re-asks rather than improvising.

**Specific application — `data-merge`:** when merge diagnostics diverge from plan expectations (e.g., match rate well outside the expected range), the agent pauses for investigation rather than auto-rolling-back or proceeding. Domain knowledge often matters at this point and the agent is unlikely to have it.

**Reason:** Improvisation against an approved plan reintroduces the silent-error mode the gates exist to prevent. Pausing is the conservative default; it can be relaxed once trust is established.

---

## 2026-05-15 — Analysis gate is per plan-document, not per spec

**Decision:** Gate 3 (analysis plan) fires once per approved plan, not once per spec. After approval, exploratory and candidate specs run freely. A new plan document = a new gate.

**Reason:** Per-spec gating would fire on every regression and produce friction she would route around. Per-plan-document gating preserves strategic review of the analytical direction without pestering on each spec.

---

## 2026-05-15 — Multi-source raw data by default

**Decision:** `data/raw/` is a directory containing multiple input files, not a single file. `data-inventory` and `cleaning-plan` iterate over its contents; `data-merge` is where they combine.

**Reason:** Most of her projects combine administrative records, survey data, and supplementary sources. A single-file assumption would force awkward scaffolding for the common case.

---

## 2026-05-15 — Figures deferred to v2

**Decision:** Event-study plots, coefficient plots, binscatters, and other figure production are out of scope for v1.

**Reason:** Figures have publication-quality requirements that vary by journal and by her own style preferences. Building this without concrete examples produces something she will discard. Defer until v2 with stated pattern requirements.

---

## 2026-05-15 — RDD is duplicated (R + Stata), not triplicated

**Decision:** RDD specs are verified by duplication across R `rdrobust` and Stata `rdrobust`, not triplication. No Python equivalent exists with matching defaults.

**Mechanism:** the design-check plan (Gate 4) declares per spec whether triplication or duplication applies, based on a documented supported-estimator list. The triplication skill reads this declaration. No silent fallback to a Python implementation that doesn't match.

**Generalizes to:** any estimator without a triplication-grade implementation in all three languages. The supported-estimator list grows as equivalences are validated.

**Reason:** A Python `rdrobust` substitute would either replicate the wrong thing or hide methodological differences. An explicit duplicate-only declaration is more honest.

---

## 2026-05-15 — Final regression tables come from Stata

**Decision:** `regression-table` is a Stata wrapper around `esttab`. It reads Stata stored estimates (`.ster` files) produced by the Stata variant of `estimation-runner` for final specs, and emits LaTeX + Markdown.

**Consequence:** the Stata `estimation-runner` must persist `.ster` files alongside the standardized JSON.

**Triplication's role:** triplication remains the verification floor. `regression-table` refuses to run if any cited final spec hasn't passed triplication.

**Reason:** she will read and customize the table-producing `.do` file. Consistency with her primary tool wins over language-neutral table generation. The setup is deliberately asymmetric — verification is symmetric across three languages; publication output privileges Stata.

---

## 2026-05-15 — v1 skill inventory (twelve skills, all user-level)

**Decision:** v1 ships twelve user-level skills:

| # | Skill | Role |
|---|---|---|
| 1 | `data-inventory` | Profile each raw dataset; flag outliers, sentinel codes, key violations |
| 2 | `cleaning-plan` | Write `plans/cleaning-plan.md` (Gate 1) |
| 3 | `stata-cleaning` | Execute cleaning plan in Stata |
| 4 | `dta-to-parquet` | Convert `.dta` → `.parquet` preserving types/labels |
| 5 | `merge-plan` | Write `plans/merge-plan.md` (Gate 2) |
| 6 | `data-merge` | Execute merge plan with diagnostics; halt on divergence |
| 7 | `analysis-plan` | Formalize her stated plan into specs; surface implicit choices (Gate 3) |
| 8 | `sample-construction` | Apply ordered restrictions; produce waterfall + `sample_def.yaml` + analysis `.parquet` |
| 9 | `estimation-runner` | Run one spec in one language; write JSON (+ `.ster` for Stata) |
| 10 | `causal-design-checklist` | Method-specific pre-flight; declare triplication vs. duplication (Gate 4) |
| 11 | `triplication` | Dispatch parallel runs; compare against tolerance; PASS/FAIL report |
| 12 | `regression-table` | `esttab` wrapper reading Stata stored estimates from passed final specs |

**Reason:** each skill is one focused capability with a verifiable success criterion. The three plan-writing skills (`cleaning-plan`, `merge-plan`, `analysis-plan`) are small structured-markdown generators that halt; they exist as separate skills (rather than baked into their adjacent execution skills) to keep the gate protocol uniform.

**Deliberately not included in v1:**
- figures (deferred to v2)
- method-specific diagnostic skills (folded into `causal-design-checklist`)
- reproducibility log (handled by a project-template `runs.log` convention, not its own skill)
- memo / methods write-up (out of v1 scope by prior decision)

---

## 2026-05-15 — `analysis-plan` primary mode is formalize-her-plan

**Decision:** the primary mode of `analysis-plan` is to take her informal description of the analysis ("the main DiD with worker and year FE, clustered at firm") and formalize it into spec files while surfacing implicit choices (sample, cluster level, baseline controls, which spec is final).

**Secondary mode:** propose specs from the data + research question. Used only when she explicitly asks.

**Reason:** most of the time she knows what regression she wants. The skill's value isn't proposing the spec — it's catching the choices she's making implicitly so they're explicit in the plan.

---

## 2026-05-15 — `sample-construction` as a separate step

**Decision:** sample restrictions are applied in a dedicated skill (`sample-construction`) that sits between merge and analysis. It applies an ordered list of restrictions, produces a waterfall table (N at each step), saves a `sample_def.yaml` snapshot, and writes the analysis `.parquet`.

**Restrictions are declared in the analysis plan** (Gate 3), so this skill executes mechanically without its own gate.

**Rejected alternatives:**
- Fold restrictions into the cleaning plan: scatters filters across cleaning steps and breaks down for multi-sample papers.
- Fold restrictions into each spec's YAML: maximum flexibility but no canonical "main sample" and no waterfall artifact.

**Reason:** most applied micro papers have a clearly-defined main analysis sample with a waterfall table the journal expects. A dedicated step produces the artifact and makes the sample auditable post-hoc.

---

## 2026-05-15 — Spec finality is explicit, not inferred

**Decision:** spec YAML has `final: false` as the default. The triplication skill refuses to run on `final: false` specs. To upgrade a spec to final, the user explicitly flips the flag (typically via the design-check plan at Gate 4).

**Reason:** without an explicit flag, the agent will drift toward triplicating everything to be safe, eroding the cost-control discipline that motivated triplication-for-final-only.

---

## 2026-05-15 — v1 supported estimators for triplication

**Decision:** the triplication skill supports these estimators in v1: `ols`, `ols_fe` (high-dim fixed effects), `iv_2sls`, `iv_2sls_fe`, `poisson_fe` (PPML).

**Out of scope for v1 (single-language with warning, or duplication where feasible):**
- RDD: duplicated (R + Stata) — see RDD decision above.
- Logit / probit / multinomial: single-language with warning.
- Matching estimators, GMM beyond 2SLS, ML methods: single-language with warning.

**Reason:** covers the bulk of applied microeconomic work in causal inference. Each additional estimator requires validating equivalence settings across three packages — a real cost. The list grows as demand surfaces.

---

## 2026-05-15 — Stata SE conventions are canonical across languages

**Decision:** when R, Stata, and Python differ on default degrees-of-freedom adjustments for clustered or robust standard errors, Stata's conventions are treated as canonical. R (`fixest`) and Python (`linearmodels`) are configured to match.

**Reason:** Stata's conventions are the most common reference in applied micro published work. Choosing one canonical convention eliminates spurious triplication failures from package-default mismatches. Equivalence configuration lives inside the triplication skill, not per call.

---
