---
name: aif-architecture
description: Generate architecture guidelines for the project. Analyzes tech stack from DESCRIPTION.md, recommends an architecture pattern, and creates .ai-factory/ARCHITECTURE.md. Use when setting up project architecture, asking "which architecture", or after /aif setup.
argument-hint: "[layers | structured[-layers|-vertical] | explicit[-layers|-vertical|-flat] | microservices[-single|-monorepo] | clean | ddd | monolith | vertical] [--style=subfolders|unified] [--policy=adapt|strict] [--existing=overwrite|merge|skip]"
allowed-tools: Read Write Edit Glob Grep Bash(mkdir *) AskUserQuestion Questions
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

Generate `.ai-factory/ARCHITECTURE.md` with architecture decisions tailored to the project.

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

Follow this sequence to resolve the classification, pattern, structural variant, action style, and codebase alignment. Below, *non-interactive* means `HANDOFF_MODE=1` / subagent runs:

**1. Classify the Codebase:**
Inspect existing application directories using **`Glob` only** (e.g. `*`, `src/*`, `src/*/*`, `app/*`, `apps/*/*`, `packages/*/*`, `cmd/*`, `internal/*`).
- Do NOT run shell commands (`find`, `tree`, `ls`, `dir`) — arbitrary Bash execution is prohibited by `allowed-tools: Bash(mkdir *)`.
- Distinguish technical layers (role-named folders with homogeneous files across entities: `controllers/`, `models/`) from feature modules (domain-named folders grouping cohesive feature logic: `users/`, `billing/`) and bootstrap framing files.
- **Clean Project or Starter Scaffolding:** re-running the framework's scaffold command and
  restoring configuration values and utility files reproduces everything present — no
  custom domain code, no business logic anywhere.
- **Existing Codebase:** any file contains project-specific domain work the scaffold would
  not regenerate — custom domain models, business services, or domain routes. That work
  must be preserved and documented, not regenerated.

**2. Determine Pattern:**
- **From `$ARGUMENTS`:**
  - Direct mappings: `layers` → Layered Architecture; `structured` → Structured Modules; `explicit` → Explicit Architecture; `microservices` → Microservices.
  - Legacy aliases (deprecated): `clean` or `ddd` → Explicit Architecture, `monolith` → Structured Modules — these resolve only the pattern; the variant question in item 3 still applies. `vertical` → Explicit Architecture (Vertical Slice By Entity) — resolves both pattern and variant; only the action-style question in item 3 applies.
  - Explicit suffixed arguments (variant already determined): `structured-layers`, `structured-vertical`, `explicit-layers`, `explicit-vertical`, `explicit-flat`, `microservices-single`, `microservices-monorepo`.
- **From Evaluation (if no architecture specified in `$ARGUMENTS`):**
  - Evaluate the project against the decision matrix in `references/architecture.md` considering team size, domain complexity, scale requirements, and tech stack.
  - Non-interactive:
    - Clean projects or starter scaffolding: select the recommended pattern from the decision matrix evaluation, report assumption and rationale, and proceed to item 3.
    - Existing codebases: STOP and return `BLOCKED` with the recommendation and options — the pattern choice itself is the material decision here.
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
    separately in item 3):
    - **Layered Architecture**
    - **Structured Modules** — variants: Technical Layer | Vertical Slices By Entity
    - **Explicit Architecture** — variants: Technical Layer | Vertical Slice By Entity | Flat Vertical Slice (Simplified)
    - **Microservices** — variants: Single Service Repository | Monorepo
    (See `references/architecture.md` for detailed pattern characteristics). Wait for user response before proceeding.

**3. Resolve Structural Variants and Action Organization Style:**
- **Unresolved Variant:** If a base pattern without suffix was chosen (`structured`, `explicit`, `microservices`, `clean`, `ddd`, `monolith`):
  - Non-interactive:
    - For clean projects or starter scaffolding: use canonical default (`structured-layers` for `structured`/`monolith`, `explicit-layers` for `explicit`/`clean`/`ddd`, `microservices-single` for `microservices`), report this assumption, and proceed.
    - For existing codebases: STOP and return `BLOCKED` with the available variant options.
  - Interactive: ASK the user to choose a variant:
    - If `structured` or `monolith`: "Which folder structure variant do you prefer for Structured Modules? 1. Technical Layer (simpler, Recommended) or 2. Vertical Slices by Entity (better for large modules)".
    - If `explicit`, `clean`, or `ddd`: "Which folder structure variant do you prefer for Explicit Architecture? 1. Technical Layer (simpler, Recommended) or 2. Vertical Slice by Entity or 3. Flat Vertical Slice - Simplified".
    - If `microservices`: "Which repository structure do you prefer for Microservices? 1. Single Microservice (dedicated repository, Recommended) or 2. Multiple Microservices (monorepo with Gateway/Libs)".
    Wait for user response before proceeding.
- **Action Organization Style (all patterns except Layered):** resolve via the reference
  Action Organization Styles. `--style` wins; both values → last wins. Default per variant:
  Style 1 (Feature Subfolders) for vertical slice variants; Style 2 (Unified Multi-Action)
  for technical-layer variants and Microservices; Layered ignores `--style` (unified-only).
  Unspecified: non-interactive → use the variant's default and report the assumption;
  interactive → ASK which style, marking the variant's default as Recommended. Wait for
  the user's answer before proceeding when asked.

**4. Codebase Alignment Check (existing codebases only):**
- Clean projects and starter scaffolding skip this item: proceed directly to Step 2 with
  the clean classification. A clean project means strict conformity: record `strict` in
  the Decision Rationale; an explicitly passed `--policy` changes nothing here (no
  discrepancy to adapt to, no legacy to quarantine).
- **If the existing structure already conforms to the chosen architecture:** (existing folders already follow the chosen pattern's conventions without conflicting layer/module placements):
  Proceed directly to Step 2 without asking or adding legacy quarantine policies (interactive and non-interactive); record `strict` in the Decision Rationale. An explicitly passed `--policy` changes nothing here: there is no legacy to quarantine and no discrepancy to adapt to.
- **If the existing structure has discrepancies with the chosen architecture:**
  - If `--policy=adapt|strict` is specified (one value): apply the **adapt policy** (document reality) or the **strict policy** (quarantine legacy, refactor later) respectively. Proceed directly to Step 2.
  - If no policy option is specified:
    - Non-interactive: STOP and return `BLOCKED` with codebase status, discrepancies, and options (resolve via `--policy=adapt|strict`).
    - Interactive: Ask the user via `AskUserQuestion`:
      ```
      The current project structure differs from the ideal [Pattern Name] architecture.
      [Briefly list 1-2 major differences]

      How should we generate ARCHITECTURE.md?
      1. Adapt guidelines to fit the existing application structure (document reality).
      2. Generate pure, strict architecture guidelines (quarantine legacy, refactor later).
      ```
      Wait for their decision before proceeding to Step 2.

**CRITICAL INSTRUCTION:** You MUST read `references/architecture.md` before generating the `ARCHITECTURE.md` artifact to ensure correct terminology, dependency directions.

### Step 2: Generate the Architecture Artifact

Create the parent directory for the resolved architecture path if needed.

**If the resolved architecture path already exists**, resolve how to treat it before generating:
- Interactive: ask the user — 1. overwrite (replace the whole file), 2. merge (regenerate
  the template sections, preserve everything outside the template), 3. skip (keep the file
  as-is).
- Non-interactive: use the explicit `--existing=overwrite|merge|skip` flag when passed;
  if absent, stop and return `BLOCKED` with the existing-file status and these options.
- `overwrite` discards sections outside the template — report the loss. `merge` regenerates
  in place the sections of the existing artifact whose headings match the artifact template
  in this SKILL.md, and preserves every other section as user content (renamed template
  sections included — matching is by heading only, never by content similarity); after
  merging, list preserved sections that may duplicate regenerated ones for manual cleanup.
  `skip` keeps the artifact untouched and continues with the pointer updates (Steps 3-5).

Generate the resolved architecture artifact (default: `.ai-factory/ARCHITECTURE.md`) with the following structure, **adapted to the project's tech stack and language**:

```markdown
# Architecture: [Pattern Name]

## Overview
[1-2 paragraphs: what this architecture is and why it was chosen for THIS project]

## Decision Rationale
- **Project type:** [from DESCRIPTION.md]
- **Tech stack:** [language, framework]
- **Key factor:** [primary reason for this choice]
- **Action organization:** [Style 1: Feature Subfolders or Style 2: Unified Multi-Action; Layered records Style 2 (unified by design)]
- **Alignment policy:** [adapt or strict]

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
```

**Rules for generation:**
- Adapt ALL examples to the project's language and framework (don't use TypeScript examples for a Go project)
- Use the project's actual conventions (import paths, naming, etc.)
- Keep it practical — focus on rules that affect day-to-day development
- Base the generated folder structure on the Step 1 (item 4) outcome: **adapt** → reflect the existing structure and its allowed dependency paths; **strict**, clean, or conforming → render the canonical target structure (strict selected for discrepancies also documents the legacy quarantine location).
- Render the chosen style per the annotations of the variant's folder tree in references — Style 1 puts each use case in its `[FeatureName]/` folder across every layer it touches (delivery included), Style 2 keeps per-entity unified Controller/Service files with DTOs next to the service; multi-audience slices add channel folders per `[Channel]`. Layered renders unified Controller/Service per entity.
- **Domain/Model placement:**
  - *For Explicit Architecture:* in the Domain layer, place entities directly in `Domain/` (or under `Domain/Entities/` per stack conventions), or in `Domain/[AggregateName]/` for aggregates with child entities. Pure cross-entity business rules belong in `Domain/Services/`, and outbound interfaces belong in `Domain/Ports/`.
  - *For Structured Modules:* place rich domain models in `[ModuleName]/Models/`.
  - *For Layered Architecture:* place data models and entities in `src/Models/`.
  - *For Microservices:* render each service with the reference single-service template
    (`Api/`, `Application/`, `Domain/`, `Infrastructure/`): entities in `Domain/`,
    outbound interfaces in `Domain/Ports/`.

### Step 3: Update DESCRIPTION.md

If the resolved DESCRIPTION.md path exists, add or update an architecture-pointer section in resolved `language.artifacts`.
Use the resolved architecture path from config, not the default path literal.

```markdown
## [Localized heading: Architecture]
[Localized sentence in resolved artifacts language referencing the resolved architecture artifact path for detailed architecture guidelines.]
[Localized label: Pattern]: [chosen pattern name]
```

### Step 4: Update AGENTS.md

If `AGENTS.md` exists in the project root, add the resolved architecture artifact path to the localized "AI Context Files" table in resolved `language.artifacts`:

```markdown
| [resolved-architecture-path] | [Localized architecture artifact description in resolved artifacts language] |
```

Only add if the resolved architecture path is not already present. If `AGENTS.md` exists
but has no "AI Context Files" table, create a minimal one following the `/aif` setup
conventions — localized `## AI Context Files` heading, localized `File | Purpose` header
row — and add the architecture row. Report the created table in Step 5.

### Step 5: Confirm

Present the confirmation in resolved `language.ui` and report the resolved architecture path:

```
[Localized success heading in `language.ui`]

[Localized pattern label in `language.ui`]: [chosen pattern]
[Localized file label in `language.ui`]: [resolved architecture path]

[Localized key-rules heading in `language.ui`]:
- [rule 1]
- [rule 2]
- [rule 3]

[Localized closing sentence in `language.ui` about workflow skills following these architecture guidelines.]
```

## Artifact Ownership

- Primary ownership: the resolved architecture artifact path (default: `.ai-factory/ARCHITECTURE.md`).
- Respect config overrides: write to the resolved architecture path from `config.yaml` when provided.
- Allowed companion updates: architecture pointer in the resolved DESCRIPTION path from `config.yaml`, architecture row in `AGENTS.md` context table.
- Read-only context: roadmap, rules, research, and plan artifacts unless user explicitly requests otherwise.

---
