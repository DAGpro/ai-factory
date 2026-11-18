---
name: aif-architecture
description: Generate architecture guidelines for the project. Analyzes tech stack from DESCRIPTION.md, recommends an architecture pattern, and creates .ai-factory/ARCHITECTURE.md. Use when setting up project architecture, asking "which architecture", or after /aif setup.
argument-hint: "[layers | structured[-layers|-vertical] | explicit[-layers|-vertical|-flat] | microservices[-single|-monorepo] | clean | ddd | monolith | vertical] [--style=subfolders|unified] [--policy=adapt|strict] [--existing=overwrite|merge|skip]"
allowed-tools: Read Write Edit Glob Grep Bash(ls *) Bash(mkdir *) AskUserQuestion Questions
disable-model-invocation: false
---

# Architecture - Generate Architecture Guidelines

## Runtime-aware questions

Apply this contract whenever the workflow asks the user to decide:

- `AskUserQuestion` and `Questions` below describe the intended interaction, not a required tool name. Use only a question tool actually exposed and permitted in the current session/mode, with its advertised schema. Do not guess aliases, probe unavailable tools, or assume `questionnaire` exists. A frontmatter entry alone does not make a tool available.
- Ask one focused question at a time with clearly labelled options. Batch independent questions only if the actual tool schema supports it; respect its field names, limits, and selection semantics. For open-ended input, use native free-text support or plain text if the schema cannot represent the question. Use the resolved UI language.
- If no usable question tool exists (or it reports unavailable in this mode), ask the same question in plain text with numbered options and allow a free-text answer. Pause work that depends on that answer; independent read-only investigation may continue. Missing UI does not make the session autonomous.
- An empty/cancelled response, a preselected option, or elapsed time is not an answer or approval. Reuse an answer already given in the session instead of asking again.
- In a subagent/background run or explicit non-interactive mode (including `HANDOFF_MODE=1`), do not prompt the user directly. Use declared defaults only for optional preferences within the authorized scope and report assumptions. If a required decision, approval, or material scope/behavior choice remains unresolved, stop dependent work and return the missing decision and options to the parent/caller using its existing blocked/failure contract. Never invent consent or select a new scope to avoid blocking.

This delivery policy applies to this skill and its linked references. Preserve workflow-specific question scope, required confirmations, approval boundaries, and blocked-result rules.

## Shared project skill-context

Before mode routing or work, read existing files relative to the current project root in this order (reuse them if already loaded):

1. `.ai-factory/skill-context/global-skill-context.md` — all bundled skills.
2. `.ai-factory/skill-context/aif-architecture/SKILL.md` — this skill.

Missing files are skipped; an existing unreadable file stops dependent work with its path.
Precedence: individual > build (when applicable) > global > skill defaults; retain non-conflicting instructions and check all outputs against the effective rules.
These layers customize workflow; they do not replace the project's rules/architecture hierarchy or relax explicit requirements, ownership, scope, safety/privacy, validation, or approval boundaries. Report conflicts with those boundaries instead of overriding them.
`global` is a project-local layer, not home-directory configuration. Custom/extension skills do not automatically use it.

Generate the resolved architecture artifact (default `.ai-factory/ARCHITECTURE.md`; `paths.architecture` overrides) with architecture decisions tailored to the project.

## Workflow

### Step 0: Load Config & Project Context

**FIRST:** Read `.ai-factory/config.yaml` if it exists to resolve:
- **Paths:** `paths.description` and `paths.architecture`
- **Language:** `language.ui` for prompts and `language.artifacts` for generated architecture content

When invoked by `/aif`, assume `.ai-factory/config.yaml` has already been written for the current setup run and already contains the resolved `language.ui` / `language.artifacts` values.

If config.yaml doesn't exist, use defaults:
- DESCRIPTION.md: `.ai-factory/DESCRIPTION.md`
- ARCHITECTURE.md: `.ai-factory/ARCHITECTURE.md`
- Language: `en` (English)

**THEN:** Read `.ai-factory/DESCRIPTION.md` (use path from config) if it exists to understand:
- Tech stack (language, framework, database, ORM)
- Project size and complexity
- Core features and requirements
- Non-functional requirements

**If `.ai-factory/DESCRIPTION.md` does not exist:**
```
⚠️  No project description found.

Run /aif first to set up project context, or describe your project manually:
- What are you building?
- Tech stack (language, framework, database)?
- Team size?
- Expected scale?
```

Allow standalone usage — if user provides manual input, use that instead.

**Read `.ai-factory/skill-context/global-skill-context.md`** — shared learned rules for all
bundled skills (skip if missing; an unreadable file stops dependent work with its path).

**Read `.ai-factory/skill-context/aif-architecture/SKILL.md`** — MANDATORY if the file exists.

This file contains project-specific rules accumulated by `/aif-evolve` from patches,
codebase conventions, and tech-stack analysis. These rules are tailored to the current project.

**How to apply skill-context rules:**
- Treat them as **project-level overrides** for this skill's general instructions
- When a skill-context rule conflicts with a general rule written in this SKILL.md,
  **the skill-context rule wins** (more specific context takes priority — same principle as nested CLAUDE.md files)
- When there is no conflict, apply both: general rules from SKILL.md + project rules from skill-context
- Do NOT ignore skill-context rules even if they seem to contradict this skill's defaults —
  they exist because the project's experience proved the default insufficient
- **CRITICAL:** skill-context rules apply to ALL outputs of this skill — including the
  ARCHITECTURE.md template. The template in this SKILL.md is a **base structure**. If a skill-context
  rule says "architecture doc MUST include X" or "MUST cover section Y" — you MUST augment the
  template accordingly. Generating ARCHITECTURE.md that violates skill-context rules is a bug.

**Enforcement:** After generating any output artifact, verify it against all skill-context rules.
If any rule is violated — fix the output before presenting it to the user.

### Step 1: Select Architecture Pattern & Structure

Follow this sequence in two phases: items 1-2 gather facts, items 3-6 resolve decisions — each decision consumes the facts before it. A flag with no effect in the resolved situation (`--policy` on a clean/conforming codebase — provisionally, the item 5 re-check can still make it effective,
`--style` for Layered, `--existing` when the resolved path does not exist yet) is a no-op —
report it as an assumption. An unrecognized `--style`, `--policy`, or `--existing` value is
reported the same way and the run continues without it (defaults and detection apply).
Below, *non-interactive* means `HANDOFF_MODE=1` / subagent runs:

**1. Read the Architecture Reference:**
Read `references/architecture.md` first — its decision matrix, terminology, styles, folder trees, and dependency directions are authoritative for everything below and for generation.

**2. Survey the Codebase (Glob + ls):**
Inspect the structure with read-only listing — **`Glob`** and targeted **`ls`**; adapt the probes to what the results reveal — no fixed root list (the reference's stack conventions are the vocabulary, not a probe script).
- `allowed-tools` restricts shell to `ls` and `mkdir`: no `find`/`tree`/`dir`, no mutating commands; `ls` only for targeted directory evidence.
- **Technique:** probe in widening passes — seed `*`, `*/*`, then widen roughly one level per pass into the roots revealed by the results, DESCRIPTION.md's stack, and manifest/config files (e.g. `src/*/*`, `apps/*/*/*`) until the structure is resolved. Glob returns file paths — infer directories from path prefixes; disambiguate with targeted `ls <dir>`, not deeper blanket globs. A shallow pass is incomplete, not evidence of absence.
- **Bounded evidence:** exclude dependency/build/vendor directories (the project's ignore files — `.gitignore` first — are the authority); targeted probes, never full recursion.
- **Classification outcome:** distinguish technical layers (role-named folders with homogeneous files across entities: `controllers/`, `models/`) from feature modules (domain-named folders grouping cohesive feature logic: `users/`,
`billing/`) and bootstrap framing files (framework entry points, layouts, and generated
boilerplate the scaffold owns — delivery/configuration plumbing, never domain logic).
  - **Clean Project or Starter Scaffolding:** re-running the framework's scaffold command and
    restoring configuration values and utility files reproduces everything present — no
    custom domain code, no business logic anywhere. Projects without a framework scaffold
    count as Clean only while they hold no domain code (configuration, scripts, and framing
    files only); any domain work makes them Existing.
  - **Existing Codebase:** any file contains project-specific domain work the scaffold would
    not regenerate — custom domain models, business services, or domain routes. That work
    must be preserved and documented, not regenerated; verify ambiguous verdicts with a
    bounded `Read` sample — names suggest roles, only content proves domain work.
- **Structural inventory (evidence for items 3-6):** layer-role vs module/context roots;
  delivery folders and routes; per-entity slice folders; unified per-entity files;
  `[FeatureName]/` subfolders vs side-by-side. Probe deep enough to resolve these signals
  (slice/feature nesting sits below the top two levels); a signal is unambiguous only when
  independent probes agree — otherwise record ambiguous (items 5-6 use free choice).
  Item 3 cites the evidence, item 4 compares it against the pattern's canon, items 5-6
  detect the de-facto variant and style.
- **Artifact-path check (fact for the blocking contract):** record whether the resolved
  architecture path (Step 0) already exists — the contract below consumes this for the
  `--existing` decision.

**3. Determine Pattern:**
- **From `$ARGUMENTS`:**
  - Direct mappings: `layers` → Layered Architecture; `structured` → Structured Modules; `explicit` → Explicit Architecture; `microservices` → Microservices.
  - Legacy aliases (deprecated): `clean` or `ddd` → Explicit Architecture, `monolith` → Structured Modules — these resolve only the pattern; the variant question in item 5 still applies. `vertical` → Explicit Architecture (Vertical Slice By Entity) — resolves both pattern and variant; only the action-style question in item 6 applies.
  - Explicit suffixed arguments (variant already determined): `structured-layers`, `structured-vertical`, `explicit-layers`, `explicit-vertical`, `explicit-flat`, `microservices-single`, `microservices-monorepo`.
  - Unrecognized positional tokens (anything outside the mappings above) are not a pattern: report them. When no positional token resolved a pattern, follow the evaluation path below; in interactive mode, first ask whether a token was meant as a pattern — a `yes` cannot create a pattern: list the accepted pattern tokens and re-ask, and a token named in the answer resolves the pattern like a direct mapping.
  - Two positional tokens compose when the second names a variant of the pattern the first maps to (e.g. `explicit vertical` ≡ `explicit-vertical`, `structured layers` ≡ `structured-layers`, `microservices monorepo` ≡ `microservices-monorepo`). In any other combination a token that maps to a pattern still resolves it, and only the non-mapping tokens are reported as unrecognized; three or more positional tokens are all unrecognized.
- **From Evaluation (if no architecture specified in `$ARGUMENTS`):**
  - Evaluate the project against the decision matrix in `references/architecture.md` considering team size, domain complexity, scale requirements, tech stack, and the item 2 survey evidence.
  - Non-interactive:
    - Clean projects or starter scaffolding: select the recommended pattern from the decision matrix evaluation, report assumption and rationale, and proceed to item 4.
    - Existing codebases: add the pattern decision to the `BLOCKED` payload (see the blocking contract) with the recommendation, alternatives, and the decisions it gates (variant options for the recommendation, style flags, `--policy` applicability).
  - Interactive: Present recommendation via `AskUserQuestion`:
    ```
    Based on your project context:
    - [reason 1 from project analysis]
    - [reason 2 from project analysis]

    Which architecture pattern should we use?

    1. [Recommended pattern] (Recommended) — [why it fits]
    2. [Alternative 1] — [brief reason]
    3. [Alternative 2] — [brief reason]
    4. [Alternative 3] — [brief reason]
    ```
    Architecture options (menu = the four base patterns; the structural variant is chosen
    separately in item 5 — an answer that names a variant for the chosen pattern, e.g.
    `Structured Modules — Vertical Slices By Entity`, also resolves it and item 5 treats it
    as determined):
    - **Layered Architecture**
    - **Structured Modules** — variants: Technical Layer | Vertical Slices By Entity
    - **Explicit Architecture** — variants: Technical Layer | Vertical Slice By Entity | Flat Vertical Slice - Simplified
    - **Microservices** — variants: Single Service Repository | Monorepo
    (See `references/architecture.md` for detailed pattern characteristics). Wait for user response before proceeding.

**4. Resolve the Alignment Policy (existing codebases only):**
- **Pattern pending (non-interactive existing):** when item 3 routed the pattern decision to
  the `BLOCKED` payload, skip items 4-6 entirely — no comparisons and no variant/style
  entries; the pending pattern entry already carries the decisions it gates, and the run
  returns ONE `BLOCKED` (plus the existing-artifact entry when the item 2 check found the
  path occupied).
- **Clean projects skip this item:** record `strict` in the Decision Rationale (strict
  conformity — no discrepancy, no legacy; the classification line distinguishes this from
  the quarantine meaning below). Proceed to item 5.
- **Compare the item 2 inventory against the chosen pattern's canonical expectations — those of the resolved variant when one is already resolved at this point (a suffixed form, `vertical`, a two-token composition, or a variant named in the item 3 answer — e.g. `explicit-vertical` compares against the vertical-slice canon); when no variant is resolved yet, compare against the pattern's variant-agnostic boundary and dependency canon:**
  - **Conforming** — no conflicting layer/module placements: record `strict` without asking or adding quarantine policies; proceed to item 5. This verdict is provisional until the variant is resolved — the item 5 re-check below re-tests it against the resolved variant's canon.
  - **Discrepancies:**
    - If `--policy=adapt|strict` is specified (one value; repeated — the last value): apply the **adapt policy** (document reality) or the **strict policy** (quarantine legacy, refactor later). Proceed to item 5.
    - Otherwise: interactive → ask via `AskUserQuestion`:
      ```
      The current project structure differs from the ideal [Pattern Name] architecture.
      [Briefly list 1-2 major differences]

      How should we generate ARCHITECTURE.md?
      1. Adapt guidelines to fit the existing application structure (document reality).
      2. Generate pure, strict architecture guidelines (quarantine legacy, refactor later).
      ```
      Wait for their decision before proceeding to item 5.
      Non-interactive → add the policy decision to the `BLOCKED` payload with codebase
      status, discrepancies, and the `--policy=adapt|strict` options.

**5. Resolve the Structural Variant:**
- **Layered has no variant:** when the resolved pattern is Layered Architecture, skip this
  item entirely (no detection, no ask) and proceed to item 6.
- **Determined by `$ARGUMENTS`** (suffixed forms, `vertical`, or a variant named in the item 3 menu answer): proceed to item 6.
- **Detected from the survey:** existing + conforming/adapt (item 4) and an unambiguous
  de-facto variant in the item 2 inventory (per-entity slice folders → vertical; side-by-side
  action/handler files without layer subfolders → flat — Explicit Architecture only, the same
  signal under Structured Modules counts as ambiguous → free choice; role-named layer folders → technical
  layer; one service root vs several → single/monorepo):
  - Non-interactive: use the detected variant, report it as an assumption, proceed to item 6.
  - Interactive: confirm — "The existing code is organized as [detected variant]. Use it?
    1. [Detected variant] (Recommended) 2. [remaining variant options for the pattern]" — and wait.
- **Free choice** (Clean classification, strict over discrepancies, or ambiguous detection):
  - Non-interactive: a Clean-classified project takes the canonical default of its resolved
    pattern (`structured-layers` for Structured Modules, `explicit-layers` for Explicit
    Architecture, `microservices-single` for Microservices; Layered skips this item) and
    reports it; existing → add the variant decision to the `BLOCKED` payload with the
    available variant options.
  - Interactive: ASK the user to choose a variant:
    - If the resolved pattern is Structured Modules (via `structured`, `monolith`, or the menu): "Which folder structure variant do you prefer for Structured Modules? 1. Technical Layer (simpler, Recommended) or 2. Vertical Slices By Entity (better for large modules)".
    - If the resolved pattern is Explicit Architecture (via `explicit`, `clean`, `ddd`, or the menu): "Which folder structure variant do you prefer for Explicit Architecture? 1. Technical Layer (simpler, Recommended) or 2. Vertical Slice By Entity or 3. Flat Vertical Slice - Simplified".
    - If the resolved pattern is Microservices: "Which repository structure do you prefer for Microservices? 1. Single Service Repository (dedicated repository, Recommended) or 2. Monorepo (multiple services with shared Libs/Gateway)".
    Wait for user response before proceeding.
- **Re-check (existing codebases with a provisional conforming verdict):** once the variant
  is resolved above while item 4 recorded conforming under the variant-agnostic canon,
  re-run the item 4 comparison against the resolved variant's canon — before item 6. Still
  conforming → keep `strict`, proceed to item 6. Downgraded to discrepancies → item 4's
  outcome becomes the downgraded one (item 6 gating reads it): resolve the policy by the
  item 4 discrepancy branch (flag / interactive ask / `BLOCKED` entry) and continue to
  item 6 — not back to item 5.

**6. Resolve the Action Organization Style (all patterns except Layered):**
- `--style=subfolders` → Style 1 (Feature Subfolders), `--style=unified` → Style 2 (Unified
  Multi-Action); `--style` wins over every default and detection below, a repeated flag —
  the last value.
- **Detected from the survey:** existing + conforming/adapt (item 4) with an unambiguous
  de-facto style (`[FeatureName]/` subfolders → Style 1; unified per-entity files → Style 2):
  non-interactive → use it and report; interactive → confirm with the detected style
  Recommended, and wait.
- **Free choice:** Style 1 default for vertical slice variants, Style 2 for technical-layer
  variants and Microservices; non-interactive → default + report; interactive → ASK with the
  variant's default style Recommended, and wait.
- Layered has no style choice: `--style` is a no-op there (unified-only) — report it as
  an assumption.

**Non-interactive blocking contract:** after items 3-6 resolve everything resolvable
from `$ARGUMENTS`, flags, defaults, and unambiguous detection, collect every remaining
material decision (pattern for existing codebases; variant when it is a free choice on an
existing codebase — strict over discrepancies or ambiguous detection; policy when
unresolved under discrepancies, including a conforming verdict downgraded by the item 5
re-check; and, when the item 2 artifact-path check found the
resolved architecture path already existing, the `--existing=overwrite|merge|skip` choice)
and return ONE `BLOCKED` listing each with its options and the flag that resolves it —
never proceed to Step 2 with any unresolved.
Resolved pattern + `--policy=adapt` + unambiguous variant detection + no existing artifact
at the resolved path (or an explicit `--existing` flag) ⇒ a fully non-interactive run on
reported assumptions; an ambiguous style never blocks — its default is reported.

`BLOCKED` payload shape (one numbered entry per pending decision; each entry lists its
options and the flag or argument that resolves it):

```text
BLOCKED — unresolved decisions for /aif-architecture (example: Explicit Architecture)
1. Pattern — options: Layered | Structured Modules | Explicit Architecture | Microservices; resolves with: explicit pattern argument
2. Structural variant — only once a pattern is resolved and the variant is a free choice on an existing codebase; options: Technical Layer | Vertical Slice By Entity | Flat Vertical Slice - Simplified; resolves with: suffixed token or `vertical`
3. Alignment policy — only once a pattern is resolved and discrepancies are found; options: adapt | strict; resolves with: --policy=<value>
4. Existing artifact — only when the resolved path already exists; options: overwrite | merge | skip; resolves with: --existing=<value>
```

### Step 2: Generate the Architecture Artifact

Create the parent directory for the resolved architecture path if needed.

**If the resolved architecture path already exists**, resolve how to treat it before generating:
- An explicit `--existing=overwrite|merge|skip` flag applies in both interactive and
  non-interactive runs without asking; report the resolved choice.
- Interactive without the flag: ask the user — 1. overwrite (replace the whole file),
  2. merge (regenerate the template sections, preserve everything outside the template),
  3. skip (keep the file as-is).
- Non-interactive without the flag: normally already collected by the Step 1 blocking
  contract (the item 2 check feeds it); if reached anyway (e.g. the file appeared after
  Step 1), stop and return `BLOCKED` with the existing-file status and these options.
- `overwrite` discards sections outside the template — report the loss (state it explicitly
  when the existing file lacks the `## Regeneration` marker — not generated by this skill).
- `merge` regenerates in place the sections of the existing artifact whose `##` headings
  match the template sections this run generates; a matched section is regenerated in full,
  including its subsections (matching is by heading only, never by content similarity, and
  happens at the `##` level — `###` subheadings never match independently). Template
  sections missing from the existing artifact are added (conditional policy sections
  included — an outcome change between runs is reflected by adding; a stale policy section
  left by a previous run is never deleted, it is preserved and named below for manual
  cleanup). When the file lacks the `## Regeneration` marker, state that merge rewrites any
  section whose heading coincides with a template heading. An artifact that carries the
  `## Regeneration` section was generated by this skill: when its headings are in a
  different `language.artifacts` and none or almost none of its template headings match the
  current template headings, regenerate every template section by its role — each
  regenerated section replaces its foreign-language counterpart in place instead of
  duplicating the document; a foreign section with no current template role is preserved
  and named below like any other user content. The `# Architecture:`
  document title is always regenerated with the resolved canonical pattern name (template,
  never matched content); every other section is preserved as user content (renamed
  template sections included). After merging, list preserved sections that may duplicate
  regenerated ones or contradict the resolved alignment policy (e.g. a policy section left
  from a previous run) for manual cleanup — report the list in Step 5.
- `skip` keeps the artifact untouched and continues with the pointer updates (Steps 3-5),
  with one guard: the Step 3 pointer's `Pattern:` line describes the skipped artifact, not
  this run's resolution — reuse the pattern the artifact itself names (the Decision
  Rationale's canonical `Pattern/variant` line when present, else the document title);
  omit the line when the artifact names none.

Generate the resolved architecture artifact (default: `.ai-factory/ARCHITECTURE.md`) with the following structure, **adapted to the project's tech stack and language**:

```markdown
# Architecture: [Pattern Name]

## Overview
[1-2 paragraphs: what this architecture is and why it was chosen for THIS project]

## Decision Rationale
- **Pattern/variant:** [canonical `Pattern (Variant)` name — e.g. `Explicit Architecture (Vertical Slice By Entity)`; Layered has no variant suffix; Microservices records `Microservices (Single Service Repository)` or `Microservices (Monorepo)`]
- **Codebase classification:** [clean | existing (conforming) | existing (discrepancies)]
- **Project type:** [from DESCRIPTION.md, or from the user's manual input in standalone runs]
- **Tech stack:** [language, framework]
- **Key factor:** [primary reason for this choice]
- **Action organization:** [Style 1: Feature Subfolders or Style 2: Unified Multi-Action; Layered records Style 2 (unified by design)]
- **Alignment policy:** [adapt or strict; for `strict`, the classification line above
  tells conformity from quarantine apart]

## Folder Structure
\`\`\`
[folder structure adapted to the project's tech stack]
[use actual framework conventions — e.g., Next.js app/ dir, Laravel app/ dir, Go cmd/ dir]
\`\`\`

## Dependency Rules
[What depends on what. Inner vs outer layers. Module boundaries.]

- ✅ [allowed dependency direction]
- ❌ [forbidden dependency direction]

## Layer/Module Communication
[How layers or modules communicate with each other]
- [pattern 1]
- [pattern 2]

## Key Principles
1. [Principle 1 — adapted to this project]
2. [Principle 2]
3. [Principle 3]

[If Clean Project or Starter Scaffolding, add:]
## Code Organization Policy
- **Starter Scaffolding:** Existing entry points/layouts belong to application framing; integrate them organically into presentation/entry adapters without rigid naming.
- **Anti-Junk-Drawer Rule:** Starter files, page components, and framework route handlers must never accumulate domain entities, business logic, or direct database queries. New business features must reside in their own modules, contexts, or slices.
- **New Code:** All subsequent development must strictly follow the rules defined in this document.

[If the strict policy was selected to resolve discrepancies (legacy quarantine applies), add:]
## Legacy vs New Code Policy
- **New Features:** All new code MUST strictly follow the pure target architecture defined in this document.
- **Legacy Quarantine:** Existing non-conforming code is quarantined as legacy; do not automatically refactor unrelated legacy code.
- **Legacy Isolation:** place legacy code by usage scope.
  - *For Explicit Architecture:* context-scoped → `[ContextName]/Infrastructure/Legacy/`; cross-context → the application-wide root `Infrastructure/Legacy/`. Legacy models or queries must NEVER leak into the Domain layer.
  - *For Structured Modules (Technical Layer):* module-scoped → `[ModuleName]/Legacy/` at the module root; cross-module → the `Legacy/` folder at the `src/` root.
  - *For Structured Modules (Vertical):* module-scoped → the module's own `Infrastructure/Legacy/`; cross-module → the same root `Legacy/` folder at the `src/` root. Keep rich models uncontaminated.
  - *For Layered Architecture:* Isolate legacy data access behind new Service/Repository interfaces; prevent new controllers from invoking legacy endpoints directly.
  - *For Microservices:* quarantine inside the owning service: `[ServiceName]/src/Infrastructure/Legacy/`; never place legacy code into `Libs/` or another service.
- **Opportunistic Migration:** Migrate legacy code incrementally only when a planned task touches that domain.

[If the adapt policy is selected, add:]
## Code Organization Note
- **Existing Code:** Documented as-is to reflect current reality without forced rewrites.
- **New Features:** Follow these architectural conventions and allowed dependency directions where practical.
- **Boundary Discipline:** Prevent further degradation by adhering strictly to the allowed dependency directions.

## Code Examples

### [Example 1 title]
\`\`\`[language]
[code example in the project's language/framework]
\`\`\`

### [Example 2 title]
\`\`\`[language]
[code example showing dependency rule]
\`\`\`

## Anti-Patterns
- ❌ [What NOT to do in this architecture]
- ❌ [Common mistake to avoid]

## Regeneration
To regenerate this document's template sections while preserving custom sections and
content, rerun `/aif-architecture` with `--existing=merge`.
```

**Rules for generation:**
- Name the pattern canonically wherever the artifact names it (heading, Decision Rationale, Step 3 pointer, Step 5 confirmation): `[Pattern] ([Variant])` — e.g. `Explicit Architecture (Vertical Slice By Entity)`; Layered without a suffix; the Step 2 `skip` mirror echoes the skipped artifact's own name verbatim instead of canonizing it
- Adapt ALL examples to the project's language and framework (don't use TypeScript examples for a Go project)
- Use the project's actual conventions (import paths, naming, etc.)
- Keep it practical — focus on rules that affect day-to-day development
- Base the generated folder structure on the resolved pattern, variant, and style:
  **adapt** → render the resolved variant/style as the target and document the existing
  structure's real deviations from it beneath the tree as current-state deviations;
  **strict**, clean, or conforming → render the canonical target structure (strict
  selected for discrepancies also documents the legacy quarantine location).
  Dependency Rules render the resolved variant's target rules; under **adapt** with a
  target that differs from reality, paths reality contradicts are named in the deviations
  list instead of being presented as allowed.
  An explicit `$ARGUMENTS` variant (`vertical` or a suffixed form) wins over detected
  structure; report the mismatch. An explicit `--style` value wins over the detected
  de-facto style the same way; report that mismatch too.
- Render the chosen style per the annotations of the variant's folder tree in references — Style 1 puts each use case in its `[FeatureName]/` folder across every layer it touches (delivery included), Style 2 keeps per-entity unified Controller/Service files with DTOs next to the service; multi-audience slices add channel folders per `[Channel]`. Layered renders unified Controller/Service per entity.
- **Domain/Model placement:**
  - *For Explicit Architecture:* in the Domain layer, place entities directly in `Domain/` (or under `Domain/Entities/` per stack conventions), or in `Domain/[AggregateName]/` for aggregates with child entities. Pure cross-entity business rules belong in `Domain/Services/`, and outbound interfaces belong in `Domain/Ports/`.
  - *For Structured Modules:* place rich domain models in `[ModuleName]/Models/`.
  - *For Layered Architecture:* place data models and entities in `src/Models/`.
  - *For Microservices:* render each service with the reference single-service template
    (`Api/`, `Application/`, `Domain/`, `Infrastructure/`): entities in `Domain/`,
    outbound interfaces in `Domain/Ports/`.

**Generation self-check (fix before presenting):**
- Tree matches the chosen style: Style 1 → `[FeatureName]/` in every layer it touches,
  delivery included; Style 2 → unified per-entity files with DTOs beside the service;
  Layered → unified per entity.
- Conditional sections match the outcome: clean → Code Organization Policy; discrepancies +
  strict → Legacy vs New Code Policy; adapt → Code Organization Note; conforming → none.
- Strict renders the chosen variant's quarantine locations.
- `[Channel]` folders only where the reference prescribes them (Style 2 in flat slices uses
  the file prefix).

### Step 3: Update DESCRIPTION.md

If the resolved DESCRIPTION.md path exists, add or update an architecture-pointer section
in resolved `language.artifacts` (append at the end unless a pointer section already exists
elsewhere — recognize it by the localized heading, by its `Pattern:` label line, or by the
sentence that references the architecture artifact path (the currently resolved one or any
earlier architecture path); the path reference is language-independent and also covers
sections written under a different `language.artifacts` and a relocated `paths.architecture`;
update that one in place). After a Step 2 `skip`, the `Pattern:` line describes the skipped artifact's own
pattern, or is omitted when the artifact names none.
Use the resolved architecture path from config, not the default path literal.

```markdown
## [Localized heading: Architecture]
[Localized sentence in resolved artifacts language referencing the resolved architecture artifact path for detailed architecture guidelines.]
[Localized label: Pattern]: [canonical pattern-variant name]
```

### Step 4: Update AGENTS.md

If `AGENTS.md` exists in the project root, add the resolved architecture artifact path to the localized "AI Context Files" table in resolved `language.artifacts`:

```markdown
| [resolved-architecture-path] | [Localized architecture artifact description in resolved artifacts language] |
```

Recognize the architecture row by its path cell: the resolved architecture path, the default
`.ai-factory/ARCHITECTURE.md`, or a row whose path cell points inside `.ai-factory/` and
whose purpose text names architecture guidelines (e.g. a relocated `paths.architecture`).
If a row already points to the resolved path, leave it; if such an architecture row
references a different path, update it in place. A row outside `.ai-factory/` whose purpose
text merely mentions architecture is user content — never rewrite it; add a separate
architecture row instead. If `AGENTS.md` has
no "AI Context Files" table, first look for any `File | Purpose` context-files table under
another heading — when several exist, use the first in document order and add the row only
there; reuse it in its existing language instead of duplicating; only when none
exists, create a minimal one per the `/aif` setup conventions (localized heading,
`File | Purpose` header) and add the architecture row; report the created table in Step 5.

### Step 5: Confirm

Present the confirmation in resolved `language.ui` and report the resolved architecture path:

```
[Localized success heading in `language.ui`]

[Localized pattern label in `language.ui`]: [canonical pattern-variant name]
[Localized style label in `language.ui`]: [resolved action organization style; omit for Layered, and under a Step 2 `skip` — the artifact is untouched]
[Localized policy label in `language.ui`]: [resolved alignment policy: adapt or strict; omit under a Step 2 `skip` — the artifact is untouched]
[Localized existing-file label in `language.ui`]: [overwrite | merge | skip — only when Step 2 resolved an existing artifact; omit the line otherwise]
[Localized file label in `language.ui`]: [resolved architecture path]
[Localized assumptions label in `language.ui`]: [one line per reported assumption (no-op flags, detected variant/style, canonical defaults, explicit-argument mismatches) plus the merge cleanup list when merging and a note when Step 4 created the AGENTS.md table; omit when none]

[Localized key-rules heading in `language.ui`]:
- [rule 1]
- [rule 2]
- [rule 3]

[Localized closing sentence in `language.ui` about workflow skills following these architecture guidelines.]
```

Under a Step 2 `skip`, the pattern line mirrors the skipped artifact's own pattern — the
same source the Step 3 pointer uses — so the confirmation and the pointer cannot name
different patterns; the style and policy lines are omitted there for the same reason (this
run's resolutions were never written into the untouched artifact), and the key-rules
bullets mirror the skipped artifact's own key rules (its Key Principles) — the block is
omitted when the artifact has none.

## Artifact Ownership

- Primary ownership: the resolved architecture artifact path (default: `.ai-factory/ARCHITECTURE.md`).
- Respect config overrides: write to the resolved architecture path from `config.yaml` when provided.
- Allowed companion updates: architecture pointer in the resolved DESCRIPTION path from `config.yaml`, architecture row in `AGENTS.md` context table.
- Read-only context: roadmap, rules, research, and plan artifacts unless user explicitly requests otherwise.

---
