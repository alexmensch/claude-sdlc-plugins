# Claude Code SDL Plugins — Project Reference

## What this is

A Claude Code plugin marketplace containing skills and agents for structured software development lifecycle workflows. The marketplace is installed by users into their own projects — the files here are skill definitions (Markdown), not application code.

## Repository structure

```
.claude-plugin/
  marketplace.json          # Marketplace manifest — lists all plugins, versions, metadata

plugins/
  feature-definition/       # Plugin: define features before building them
    .claude-plugin/plugin.json
    skills/define-feature/SKILL.md
    agents/
      user-researcher.md      # Web research to validate the user problem (runs in foreground)

  feature-development/      # Plugin: end-to-end feature implementation workflow
    .claude-plugin/plugin.json
    skills/
      new-feature/SKILL.md  # Orchestrating skill — drives the full workflow
      pull-request/SKILL.md  # PR formatting rules
      semver/SKILL.md        # Semantic versioning reference
    agents/
      technical-spec.md      # Produces technical specs from requirements (runs in foreground)
      test-writer.md         # Writes tests from specs (runs in background, isolated worktree)
      coverage-reviewer.md   # Analyses test coverage of new code (runs in foreground)
      code-reviewer.md       # Reviews branch changes and returns findings (runs in foreground)

  roadmap/                  # Plugin: roadmap planning, strategy, and communication
    .claude-plugin/plugin.json
    skills/
      define-strategy/SKILL.md # Product strategy definition and stress-testing
      plan-roadmap/SKILL.md
      communicate-roadmap/SKILL.md
    agents/
      persona-researcher.md   # Web research to validate/challenge persona claims (runs in foreground)
      market-researcher.md    # Web research on competitive landscape (runs in foreground)

  debugging/                # Plugin: code investigation and traceability
    .claude-plugin/plugin.json
    skills/blame/SKILL.md

  documentation/            # Plugin: user-facing documentation generation
    .claude-plugin/plugin.json
    skills/write-changelog/SKILL.md

  legal/                    # Plugin: legal and compliance assessment
    .claude-plugin/plugin.json
    skills/
      search-trademark/SKILL.md    # Trademark and naming conflict research
      assess-compliance/SKILL.md   # Regulatory compliance assessment
    agents/
      trademark-researcher.md      # Web research for trademark/naming conflicts (runs in foreground)
      compliance-researcher.md     # Web research for regulatory requirements (runs in foreground)

  marketing/                # Plugin: marketing website and landing page generation
    .claude-plugin/plugin.json
    skills/
      build-marketing-site/SKILL.md  # Full marketing site from product context
      build-landing-page/SKILL.md    # Topic-specific landing pages (comparisons, migrations, etc.)
      evaluate-seo/SKILL.md          # SEO/GEO evaluation and recommendations
```

## Naming conventions

- **Plugins** are nouns or noun phrases describing a domain: `feature-definition`, `roadmap`, `debugging`
- **Skills** are verb phrases describing an action: `define-feature`, `plan-roadmap`, `blame`
- A plugin name and a skill name must never be the same

## Key conventions

### File formats

- Skills are defined in `SKILL.md` with YAML frontmatter (`name`, `description`, optionally `model`, `background`, `isolation`)
- Agents are plain Markdown files in an `agents/` directory with the same frontmatter structure
- Plugin metadata lives in `.claude-plugin/plugin.json` (name, description, version)
- The marketplace manifest is `.claude-plugin/marketplace.json`

### Agent isolation model

Agents are invoked by orchestrating skills and never interact directly with the user. The orchestrating skill handles all user interaction (presenting outputs, collecting approvals, making changes). Each agent file contains an explicit isolation statement reinforcing this boundary.

**feature-development** (invoked by `new-feature`):
- **technical-spec** — runs in the foreground, returns a spec to the orchestrator
- **test-writer** — runs in the background in an isolated worktree, reports what it created
- **coverage-reviewer** — runs in the foreground, returns a coverage report (or skips if no coverage tooling is detected)
- **code-reviewer** — runs in the foreground, returns a findings report to the orchestrator

**feature-definition** (invoked by `define-feature`):
- **user-researcher** — runs in the foreground, researches whether the user problem exists, how people solve it today, and gaps in existing solutions

**roadmap** (invoked by `plan-roadmap` and `define-strategy`):
- **persona-researcher** — runs in the foreground, conducts web research to validate/challenge persona claims, returns a structured report (invoked by `plan-roadmap`)
- **market-researcher** — runs in the foreground, researches competitive landscape, validates strategic claims about the market (invoked by `define-strategy`)

**legal** (invoked by `search-trademark` and `assess-compliance`):
- **trademark-researcher** — runs in the foreground, searches trademark registries, existing products, and SEO landscape for naming conflicts, returns a structured report (invoked by `search-trademark`)
- **compliance-researcher** — runs in the foreground, researches applicable regulatory frameworks, enforcement patterns, and practical requirements, returns a structured report (invoked by `assess-compliance`)

### Requirements file format

Skills that produce or consume requirements files use this structure:

```markdown
---
guid: <uuid>
date: <YYYY-MM-DD>
feature: <kebab-case-slug>
---

#### Feature: [name]

[Summary]

#### Requirements

| # | Requirement | Description | Acceptance Criteria | Edge Cases / Error Conditions |
```

The `guid` is used to link requirements files to PRs (via the PR body) and to the roadmap.

### Product directory structure

The `product/` directory holds all planning artifacts, organised into subdirectories that mirror the roadmap:

```
product/
  STRATEGY.md                   # Product strategy (owned by define-strategy)
  ROADMAP.md                    # Roadmap (owned by plan-roadmap)
  EXTERNAL-ROADMAP.md           # External-facing roadmap (owned by communicate-roadmap)
  unassigned-feature.md         # Not yet assigned to a release
  release-name/                 # Kebab-cased release name
    feature-in-release.md
  shipped/                      # Completed features
    completed-feature.md
```

- `define-feature` writes new requirements files to the root (unassigned).
- `define-strategy` writes `STRATEGY.md` to the root.
- `plan-roadmap` reads `STRATEGY.md` to inform sequencing, moves files into release subfolders (and `shipped/`) after the user approves the roadmap structure, then auto-commits the moves.
- `new-feature` moves a file to `shipped/` when marking it as shipped, then auto-commits.
- `blame` and `write-changelog` search recursively, so they work with any directory layout.

### ROADMAP.md and STRATEGY.md formats

The roadmap file at `product/ROADMAP.md` is owned by the `plan-roadmap` skill and contains: Overview, Target Users, Planned (release-grouped feature tables with global sequence numbers and GUIDs), and Shipped (features with PR numbers). Feature links use subdirectory-relative paths (e.g. `./release-name/feature.md`, `./shipped/feature.md`). The `new-feature` skill updates the shipped table automatically after a PR is created.

The strategy file at `product/STRATEGY.md` is owned by the `define-strategy` skill and contains: Strategic Thesis, Differentiation, Market Context, and Risks and Open Questions. If it exists, `plan-roadmap` reads it to inform sequencing decisions.

### PR format

PRs include a `**Requirements:** \`<guid>\`` line in the body when created from a requirements file. This is the link that `blame` and `write-changelog` follow to trace code back to requirements.

### Platform and state assumptions

- **blame** — requires GitHub for PR tracing (steps 4–5). If the remote is not `github.com`, the chain stops at the commit level and the user is informed.
- **write-changelog** — checks whether PRs have been merged before writing entries. Unmerged PRs are flagged to the user.
- **communicate-roadmap** — reads the Target Users section from ROADMAP.md to inform tone and framing. Falls back to the Overview section if no user profiles exist.
- **coverage-reviewer** — only runs if the project has coverage tooling configured. Does not install tools; skips gracefully if none are found.
- **code-reviewer** — detects the default branch name dynamically rather than assuming `master`.
- **search-trademark** — uses web research to check trademark registries and existing products. Results are informational, not legal advice.
- **assess-compliance** — scans the codebase for data-handling patterns and researches applicable regulations. Produces a unified compliance report. Skips HIPAA by design. Output location is determined per-project.
- **build-marketing-site** — auto-detects the project's framework for page generation. Falls back to static HTML if no framework is found. Output location is determined per-project.
- **build-landing-page** — matches the style of any existing marketing site. Uses web research for comparison and migration pages.
- **evaluate-seo** — can be invoked independently or by `build-marketing-site` and `build-landing-page`. Uses web research to assess competitive search landscape.

## Versioning

- Each plugin has its own version in `.claude-plugin/plugin.json`
- The marketplace has a top-level version in `.claude-plugin/marketplace.json`
- Bump the marketplace version when adding new plugins or making cross-cutting changes
- Bump individual plugin versions when changing only that plugin's skills or agents

## Cross-plugin dependencies

The skills form a pipeline. Data flows through this chain:

```
define-strategy → STRATEGY.md ──────→ plan-roadmap (reads strategy to inform sequencing)
define-feature  → requirements file → plan-roadmap → ROADMAP.md
                                    → new-feature → technical-spec → test-writer
                                                   → coverage-reviewer
                                                   → code-reviewer
                                                   → pull-request (with GUID)
                                                   → ROADMAP.md (shipped update)
                                    → write-changelog (reads PR + requirements)
                                    → blame (traces code → PR → GUID → requirements)
                                    → communicate-roadmap (reads ROADMAP.md)

search-trademark → trademark-researcher → naming risk report
assess-compliance → compliance-researcher → COMPLIANCE.md
                    (reads codebase, STRATEGY.md, ROADMAP.md, requirements files)

build-marketing-site → evaluate-seo (optional)
                       (reads STRATEGY.md, ROADMAP.md, README.md, codebase)
build-landing-page   → evaluate-seo (optional)
                       (reads STRATEGY.md, ROADMAP.md, existing marketing site)
evaluate-seo         (standalone or invoked by marketing skills)
```

The requirements GUID is the linking key across the feature pipeline. The legal and marketing plugins read from the same product artifacts (STRATEGY.md, ROADMAP.md, requirements files) but do not write to them.

## When editing skills

- Keep skills under 500 lines where possible
- Explain the *why* behind instructions rather than relying on heavy-handed MUST/NEVER
- Use `AskUserQuestion` for user-facing decisions; agents that run in isolation must not use it
- Test trigger phrases in descriptions — skills undertrigger more often than they overtrigger
- After changes, ensure cross-references between skills still match (path names, format expectations, step numbers)
