---
name: aif-architecture
description: Generate architecture guidelines for the project. Analyzes tech stack from DESCRIPTION.md, recommends an architecture pattern, and creates .ai-factory/ARCHITECTURE.md. Use when setting up project architecture, asking "which architecture", or after /aif setup.
argument-hint: "[layers | structured[-layers|-vertical] | explicit[-layers|-vertical|-flat] | microservices[-single|-monorepo]] [--style=subfolders|--style=unified] [--strict|--adapt]"
allowed-tools: Read Write Glob Grep Bash(mkdir *) AskUserQuestion Questions
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

### Step 1: Recommend Architecture

Based on project context, evaluate against the decision matrix and recommend an architecture:

**If `$ARGUMENTS` specifies an architecture** (e.g., `/aif-architecture explicit`):
- **Direct mapping** (no suffix needed):
  - `layers` → Layered Architecture
  - `structured` → Structured Modules (ask variant — see below)
  - `explicit` → Explicit Architecture (ask variant — see below)
  - `microservices` → Microservices (ask variant — see below)
- **Legacy aliases** (deprecated, mapped to current patterns — may be removed in future):
  - `clean` → Explicit Architecture
  - `ddd` → Explicit Architecture
  - `monolith` → Structured Modules
  - `vertical` → Explicit Architecture (Vertical Slice By Entity)
- **With suffix** (variant is determined, no need to ask):
  - `structured-layers` → Structured Modules (Technical Layer)
  - `structured-vertical` → Structured Modules (Vertical Slices By Entity)
  - `explicit-layers` → Explicit Architecture (Technical Layer)
  - `explicit-vertical` → Explicit Architecture (Vertical Slice By Entity)
  - `explicit-flat` → Explicit Architecture (Flat Vertical Slice - Simplified)
  - `microservices-single` → Microservices (Single Service Repository)
  - `microservices-monorepo` → Microservices (Monorepo)
- **Without suffix** (resolve variant):
  - In non-interactive mode (`HANDOFF_MODE=1` / subagent):
    - For clean projects or starter scaffolding: use the canonical default (`structured-layers` for `structured`, `explicit-layers` for `explicit`, `microservices-single` for `microservices`), report this assumption, and proceed.
    - For existing codebases: STOP and return `BLOCKED` with the available variant options (do not guess architecture scope on existing code).
  - In interactive mode: ASK the user to choose a variant:
    - If `structured` without suffix: "Which folder structure variant do you prefer for Structured Modules? 1. Technical Layer (simpler, Recommended) or 2. Vertical Slices by Entity (better for large modules)".
    - If `explicit` without suffix: "Which folder structure variant do you prefer for Explicit Architecture? 1. Technical Layer (simpler, Recommended) or 2. Explicit Architecture (Vertical Slice By Entity) or 3. Explicit Architecture (Flat Vertical Slice - Simplified)".
    - If `microservices` without suffix: "Which repository structure do you prefer for Microservices? 1. Single Microservice (dedicated repository, Recommended) or 2. Multiple Microservices (monorepo with Gateway/Libs)".
    Wait for their answer before generating the artifact.
- **For Vertical Slices (action organization choice):**
  - When any vertical slice variant is chosen (`structured-vertical`, `explicit-vertical`, `explicit-flat`, or legacy `vertical`):
    - **If action organization style is specified in `$ARGUMENTS`:**
      - `--style=subfolders` (or `--subfolders`) → use **Feature Subfolders** directly without asking.
      - `--style=unified` (or `--unified`) → use **Unified Multi-Action** directly without asking.
    - **If no style option specified:**
      - In non-interactive mode (`HANDOFF_MODE=1` / subagent): Default to **Feature Subfolders** and report this assumption.
      - In interactive mode: ASK the user: "Which action organization style do you prefer inside vertical slices? 1. Feature Subfolders (Recommended) — dedicated folder `[EntityName]/[FeatureName]/` with Action, DTO, Handler (best for isolation) or 2. Unified Multi-Action — single `[EntityName]Controller` and `[EntityName]Service` per entity". Wait for their answer before generating the artifact.
- Use the resolved architecture directly, skip the recommendation step and proceed to Step 1.5

**If no specific architecture requested:**
- Evaluate the project against the decision matrix (see `references/architecture.md`)
- Consider: team size, domain complexity, scale requirements, tech stack
- In non-interactive mode (`HANDOFF_MODE=1` / subagent):
  - For clean projects or starter scaffolding: select the recommended pattern from the decision matrix evaluation, report the assumption and rationale, and proceed directly to Step 1.5.
  - For existing codebases: STOP and return `BLOCKED` with the recommendation and options.
- In interactive mode: Present recommendation via `AskUserQuestion`:

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

Architecture options:
- **Structured Modules (Technical Layer)**
- **Structured Modules (Vertical Slices By Entity)**
- **Explicit Architecture (Technical Layer)**
- **Explicit Architecture (Vertical Slice By Entity)**
- **Explicit Architecture (Flat Vertical Slice - Simplified)**
- **Microservices**
- **Layered Architecture**
(See `references/architecture.md` for detailed descriptions of these patterns to formulate your recommendation).

**CRITICAL INSTRUCTION:** You MUST read `references/architecture.md` before generating the `ARCHITECTURE.md` artifact to ensure correct terminology, dependency directions.

### Step 1.5: Codebase Alignment Check

**CRITICAL:** Inspect existing application directories using **`Glob` only** (e.g. `*`, `src/*`, `src/*/*`, `app/*`).
- Do NOT run shell commands (`find`, `tree`, `ls`, `dir`) — arbitrary Bash execution is prohibited by `allowed-tools: Bash(mkdir *)`.
- Distinguish technical layers (role-named folders with homogeneous files across entities: `controllers/`, `models/`) from feature modules (domain-named folders grouping cohesive feature logic: `users/`, `billing/`) and bootstrap framing files.

1. **Clean Project or Starter Scaffolding:**
   - If empty or containing only bootstrap framing: proceed directly to Step 2 without asking.
   - In Step 2, generate `## Code Organization Policy`.

2. **Existing Codebase (any existing code):**
   - **If the existing structure already conforms to the chosen architecture:**
     Proceed directly to Step 2 without asking or adding starter/legacy quarantine policies (both in interactive and non-interactive mode).
   - **If the existing structure has discrepancies with the chosen architecture:**
     - **If policy option is specified in `$ARGUMENTS`:**
       - `--strict` (or `strict`) → choose Option 2: Pure strict architecture guidelines (quarantine legacy under `Infrastructure/Adapters/Legacy/`, refactor later). Proceed directly to Step 2.
       - `--adapt` (or `adapt`) → choose Option 1: Adapt guidelines to fit existing application structure (document reality). Proceed directly to Step 2.
     - **If no policy option is specified:**
       - In non-interactive mode (`HANDOFF_MODE=1` / subagent): STOP and return `BLOCKED` with codebase status, discrepancies, and options (pass `--strict` or `--adapt` to resolve in non-interactive mode).
       - In interactive mode: Ask the user via `AskUserQuestion`:
         ```
         The current project structure differs from the ideal [Pattern Name] architecture.
         [Briefly list 1-2 major differences]

         How should we generate ARCHITECTURE.md?
         1. Adapt guidelines to fit the existing application structure (document reality).
         2. Generate pure, strict architecture guidelines (quarantine legacy, refactor later).
         ```
         Wait for their decision before proceeding to Step 2.

### Step 2: Generate the Architecture Artifact

Create the parent directory for the resolved architecture path if needed.

Generate the resolved architecture artifact (default: `.ai-factory/ARCHITECTURE.md`) with the following structure, **adapted to the project's tech stack, programming language, and resolved artifacts language (`language.artifacts`)**:

```markdown
# [Localized heading: Architecture]: [Pattern Name]

## [Localized heading: Overview]
[1-2 paragraphs: what this architecture is and why it was chosen for THIS project]

## [Localized heading: Decision Rationale]
- **[Localized label: Project type]:** [from DESCRIPTION.md]
- **[Localized label: Tech stack]:** [language, framework]
- **[Localized label: Key factor]:** [primary reason for this choice]

## [Localized heading: Folder Structure]
\`\`\`
[folder structure adapted to the project's tech stack]
[use actual framework conventions — e.g., Next.js app/ dir, Laravel app/ dir, Go cmd/ dir]
\`\`\`

## [Localized heading: Dependency Rules]
[What depends on what. Inner vs outer layers. Module boundaries.]

- ✅ [allowed dependency direction]
- ❌ [forbidden dependency direction]

## [Localized heading: Layer/Module Communication]
[How layers or modules communicate with each other]
- [pattern 1]
- [pattern 2]

## [Localized heading: Key Principles]
1. [Principle 1 — adapted to this project]
2. [Principle 2]
3. [Principle 3]

[If clean project or starter scaffolding, add:]
## [Localized heading: Code Organization Policy]
- **Starter Scaffolding:** Existing entry points/layouts belong to application framing; integrate them organically into presentation/entry adapters without rigid naming.
- **Anti-Junk-Drawer Rule:** Starter files, page components, and framework route handlers must never accumulate domain entities, business logic, or direct database queries. New business features must reside in their own modules, contexts, or slices.
- **New Code:** All subsequent development must strictly follow the rules defined in this document.

[If strict architecture chosen (Option 2), add:]
## [Localized heading: Legacy vs New Code Policy]
- **New Features:** All new code MUST strictly follow the pure target architecture defined in this document.
- **Legacy Quarantine:** Existing non-conforming code is quarantined as legacy; do not automatically refactor unrelated legacy code.
- **Anti-Corruption Layer (ACL):** Isolate interactions between new and legacy code using adapters under `Infrastructure/Adapters/Legacy/` (or module infrastructure). Legacy models or queries must NEVER leak into the Domain layer.
- **Opportunistic Migration:** Migrate legacy code incrementally only when a planned task touches that domain.

[If adapt to reality chosen (Option 1), add:]
## [Localized heading: Code Organization Note]
- **Existing Code:** Documented as-is to reflect current reality without forced rewrites.
- **New Features:** Follow these architectural conventions and allowed dependency directions where practical.
- **Boundary Discipline:** Prevent further degradation by adhering strictly to the allowed dependency directions.

## [Localized heading: Code Examples]

### [Example 1 title]
\`\`\`[language]
[code example in the project's language/framework]
\`\`\`

### [Example 2 title]
\`\`\`[language]
[code example showing dependency rule]
\`\`\`

## [Localized heading: Anti-Patterns]
- ❌ [What NOT to do in this architecture]
- ❌ [Common mistake to avoid]
```

**Rules for generation:**
- Generate headings and prose in resolved `language.artifacts`; keep code identifiers, import paths, and technical keywords in English per `language.technical_terms: keep`.
- For `[Pattern Name]`, use the canonical English display name (e.g. `Explicit Architecture (Vertical Slice By Entity)`), optionally accompanied by localized clarification when `language.artifacts` is non-English.
- Adapt ALL examples to the project's language and framework (don't use TypeScript examples for a Go project)
- Use the project's actual conventions (import paths, naming, etc.)
- Keep it practical — focus on rules that affect day-to-day development
- Base the generated folder structure on the user's decision in Step 1.5 (either adapted to reality or strict pure architecture). Do not automatically merge them without user consent.
- For vertical slice architectures, render the slice folder structure according to the chosen action organization style:
  - If **Feature Subfolders**: render dedicated `[FeatureName]/` subfolders containing Action, Handler, and DTO files (either directly under the entity slice in flat/structured variants, or under `Application/` and `Presentation/` in explicit-vertical).
  - If **Unified Multi-Action**: render unified `[EntityName]Controller` and `[EntityName]Service` files per entity.
- In the Domain layer: place entities directly in `Domain/` (or under `Domain/Entities/` per stack conventions), or in `Domain/[AggregateName]/` for aggregates with child entities. Pure cross-entity business rules belong in `Domain/Services/`, and outbound interfaces belong in `Domain/Ports/`.

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

Only add if the resolved architecture path is not already present.

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
