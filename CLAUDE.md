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

  roadmap/                  # Plugin: roadmap planning and communication
    .claude-plugin/plugin.json
    skills/
      plan-roadmap/SKILL.md
      communicate-roadmap/SKILL.md

  debugging/                # Plugin: code investigation and traceability
    .claude-plugin/plugin.json
    skills/blame/SKILL.md

  documentation/            # Plugin: user-facing documentation generation
    .claude-plugin/plugin.json
    skills/write-changelog/SKILL.md
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

All four agents in `feature-development` are invoked by the `new-feature` orchestrating skill. None of them interact directly with the user:

- **technical-spec** — runs in the foreground, returns a spec to the orchestrator
- **test-writer** — runs in the background in an isolated worktree, reports what it created
- **coverage-reviewer** — runs in the foreground, returns a coverage report (or skips if no coverage tooling is detected)
- **code-reviewer** — runs in the foreground, returns a findings report to the orchestrator

The orchestrating skill handles all user interaction (presenting outputs, collecting approvals, making changes). Each agent file contains an explicit isolation statement reinforcing this boundary.

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

### ROADMAP.md format

The roadmap file at `requirements/ROADMAP.md` is owned by the `plan-roadmap` skill and contains: Overview, Target Users, Planned (release-grouped feature tables with global sequence numbers and GUIDs), and Shipped (features with PR numbers). The `new-feature` skill updates the shipped table automatically after a PR is created.

### PR format

PRs include a `**Requirements:** \`<guid>\`` line in the body when created from a requirements file. This is the link that `blame` and `write-changelog` follow to trace code back to requirements.

### Platform and state assumptions

- **blame** — requires GitHub for PR tracing (steps 4–5). If the remote is not `github.com`, the chain stops at the commit level and the user is informed.
- **write-changelog** — checks whether PRs have been merged before writing entries. Unmerged PRs are flagged to the user.
- **communicate-roadmap** — reads the Target Users section from ROADMAP.md to inform tone and framing. Falls back to the Overview section if no user profiles exist.
- **coverage-reviewer** — only runs if the project has coverage tooling configured. Does not install tools; skips gracefully if none are found.
- **code-reviewer** — detects the default branch name dynamically rather than assuming `master`.

## Versioning

- Each plugin has its own version in `.claude-plugin/plugin.json`
- The marketplace has a top-level version in `.claude-plugin/marketplace.json`
- Bump the marketplace version when adding new plugins or making cross-cutting changes
- Bump individual plugin versions when changing only that plugin's skills or agents

## Cross-plugin dependencies

The skills form a pipeline. Data flows through this chain:

```
define-feature → requirements file → plan-roadmap → ROADMAP.md
                                   → new-feature → technical-spec → test-writer
                                                  → coverage-reviewer
                                                  → code-reviewer
                                                  → pull-request (with GUID)
                                                  → ROADMAP.md (shipped update)
                                   → write-changelog (reads PR + requirements)
                                   → blame (traces code → PR → GUID → requirements)
                                   → communicate-roadmap (reads ROADMAP.md)
```

The requirements GUID is the linking key across all of these.

## When editing skills

- Keep skills under 500 lines where possible
- Explain the *why* behind instructions rather than relying on heavy-handed MUST/NEVER
- Use `AskUserQuestion` for user-facing decisions; agents that run in isolation must not use it
- Test trigger phrases in descriptions — skills undertrigger more often than they overtrigger
- After changes, ensure cross-references between skills still match (path names, format expectations, step numbers)
