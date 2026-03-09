---
name: new-feature
description: Follow a standard workflow for feature implementation in a project that ensures that a technical spec drives both test writing and implementation in parallel, followed by a test coverage review and code review that are always performed before a pull request is submitted to the default branch of the project's code base. Use when the user says "new feature", "build a feature", "implement a feature", "implement this", or provides a requirements file to build from.
---

When implementing a new feature or new functionality in an existing codebase, you must adhere to the steps that follow with the goal of creating high quality, efficient, and clear code at all times. Follow these steps in strict order:

1. Before starting work, always create a new branch. Make sure that the default branch is at the latest version and make sure the new branch is created from the default branch. Never work in the default branch for the project, which is usually `master`, but may also be `main` in some cases. I do not care what the branch is called, you can decide.
2. Determine how the requirements are being provided:

   **If the user has provided a path to a requirements file** (e.g. a file in `requirements/`): read the file. Extract the `guid` and `date` fields from its frontmatter if present — store the `guid` for use in the roadmap update (step 6), for updating the file after code review (step 5), and for the PR (step 11). Treat the requirements table in the file as the confirmed requirements — proceed directly to summarising what you are about to pass to the `technical-spec` agent and use `AskUserQuestion` to ask the user to confirm, then invoke the agent. Track the file path for use in step 5.

   **If the user has described the feature in chat** (no file provided): require descriptive precision. If there is ambiguity in what the user is requesting, without being pedantic, ask for clarification. Once you have a clear understanding of the requirements, summarise what you are about to pass to the technical-spec agent and use `AskUserQuestion` to ask the user to confirm before you proceed.

   In both cases, only after the user confirms should you invoke the technical-spec agent. The technical-spec agent will deeply analyse the codebase and produce a structured technical specification with a requirements table. Once you receive the spec back:

   1. Present the complete specification to the user and use `AskUserQuestion` to ask them to review it:

      > _"Please review this technical specification. Select **Approved** to proceed, or select **Changes needed** and describe what needs to change."_

      Provide at least two options: one for approval and one to request changes. If the user requests changes, re-invoke the technical-spec agent passing all three of: the original requirements, the existing technical specification, and the user's requested changes. This allows the agent to revise the spec directly without repeating its codebase analysis. Repeat this review step until approved.

   2. If the original requirements were provided as a file (step 2), append the approved technical specification as a new `## Technical Specification` section at the end of that file. This serves as a permanent record of the agreed spec alongside the original requirements.

3. Once you have the approved technical spec, launch two agents in parallel: hand the full technical spec to the implementer agent (which runs in the background in an isolated worktree, and must be invoked with Write and Edit tool permissions), and hand the full technical spec to the test-writer agent (which also runs in the background in an isolated worktree, and must also be invoked with Write and Edit tool permissions). Both agents work from the requirements table in the technical spec. Wait for both agents to complete before proceeding to step 4.
4. Once both agents have finished, merge their work into the current branch. Neither agent performs any git operations — they write files and report what they created, but leave their worktrees uncommitted. When each agent's Task completes, its result will include the worktree path and branch name.

   **Merge the implementer's work first:**

   1. Commit the implementer's work inside its worktree:
      ```bash
      git -C "<implementer_worktree_path>" add -A && git -C "<implementer_worktree_path>" commit -m "Implement [feature name]"
      ```
   2. Merge the implementer's branch into your current feature branch:
      ```bash
      git merge <implementer_worktree_branch> --no-edit
      ```
      If this merge produces a conflict, stop and ask the user how to proceed.
   3. Remove the worktree and its branch:
      ```bash
      git worktree remove <implementer_worktree_path> --force
      git branch -D <implementer_worktree_branch>
      ```
   4. Verify the implementation files are present: run `git log --oneline -3` to confirm the merge commit is visible.

   **Then merge the test-writer's work:**

   1. Commit the test-writer's work inside its worktree:
      ```bash
      git -C "<test_worktree_path>" add -A && git -C "<test_worktree_path>" commit -m "Add tests for [feature name]"
      ```
   2. Merge the test-writer's branch into your current feature branch:
      ```bash
      git merge <test_worktree_branch> --no-edit
      ```
      If this merge produces a conflict, stop and ask the user how to proceed.
   3. Remove the worktree and its branch:
      ```bash
      git worktree remove <test_worktree_path> --force
      git branch -D <test_worktree_branch>
      ```
   4. Verify the test files are present: run `git log --oneline -3` to confirm the merge commit is visible.

   Implementation is merged first because test files typically reference implementation code, making the second merge cleaner.

   **Before running tests**, compare the test-writer's expectations against the implementer's code and identify any meaningful design discrepancies. These are cases where the two agents made different but valid design decisions — different naming, different parameter shapes, different return structures, different abstractions. This is normal and expected when two agents work from the same spec in parallel.

   If discrepancies exist, do not silently resolve them. Use `AskUserQuestion` to present each one clearly to the user:
   - Describe what the test expects
   - Describe what the implementation does
   - Explain the trade-off or difference between the two approaches
   - Ask which direction the user wants to go

   The user's answer determines what changes — either the implementation is adjusted to match the tests, or (with the user's explicit approval) the tests are updated to match the implementation. In either case, do not proceed until the user has confirmed the resolution for every discrepancy.

   Then run all tests for the whole project. In no circumstances will you change any code in the tests without confirming with the user first. The strong principle here is that tests are written to cover desired functionality, not just to pass. Never change tests just to make them pass — you must determine whether there is a genuine bug in the implementation first. If you think there is a bug in a test, confirm with the user first. This step is completed when all tests pass.
5. Before the code review, check whether the project has code coverage tooling. Invoke the coverage-reviewer agent — it runs in the foreground and returns a structured report. The agent will detect coverage tooling automatically; if none is found, it returns a skip message and you move on to step 6.

   If the agent returns coverage findings:

   1. Present the full coverage report to the user using `AskUserQuestion`. Ask the user which uncovered areas (if any) they want tests added for, and which they accept as-is.
   2. For any areas the user wants covered: write the additional tests yourself, following the existing test patterns in the codebase. Run the full test suite to confirm nothing is broken.
   3. Create a **single commit** containing all coverage-related test additions.

   If coverage is already complete or the user accepts all gaps, proceed without changes.
6. Next, invoke the code-reviewer agent. It will analyse the branch and return a structured findings report — it does not interact with the user or make any changes. Once you receive the report:

   1. Present the complete findings to the user using `AskUserQuestion`. Show the violations table exactly as returned. Ask the user to indicate which violations they want fixed and which (if any) they want to dismiss. If there are no violations, inform the user and proceed.
   2. Make all approved changes yourself. Do not make changes the user has not approved.
   3. Run the full test suite to confirm nothing is broken. If tests fail after your changes, investigate and fix the issue before proceeding.
   4. Create a **single commit** containing all code review fixes.

   After this step completes: if the original requirements were provided as a file on disk (step 2), and any changes were made, append an "Out-of-spec changes" section to the requirements file documenting those changes. Use the `define-feature` skill to produce the correctly-formatted table rows if it is available; otherwise write the rows directly. Use the same table format as the requirements table. This section is an audit record of changes that were made outside the original specification.

   **Context checkpoint:** The remaining steps (7–13) are mechanical and only require: the feature name, requirements GUID, requirements file path, feature branch name, and whether a ROADMAP.md update is applicable. The technical specification, implementation details, and code review findings are no longer needed. If the session context is approaching its limit, this is a safe point to compact.
7. If the original requirements were provided as a file (step 2) and `./requirements/ROADMAP.md` exists, update the roadmap to record that this feature has shipped. The ROADMAP.md format is owned by the `plan-roadmap` skill — follow its structure exactly when making edits:
    1. Find the feature's row in the planned table by matching the GUID extracted in step 2.
    2. Remove that row from its release section. If removing it leaves the release section with no remaining rows, remove the entire release section (heading, description, and empty table).
    3. Renumber the sequence column (`#`) of the remaining planned rows so they remain contiguous starting from 1.
    4. Add the feature to the Shipped table without a PR number — leave the PR column empty for now. If no Shipped table exists yet, create one following the structure defined in the `plan-roadmap` skill.
    5. Commit the ROADMAP.md change with the message: `Update roadmap: mark [feature-slug] as shipped`.
8. Run the linting tool that's configured for the project.
9. Run either the build command or mock deploy command for the project to ensure there are no build errors. Ensure that you do not build or deploy the project to production, you are only ensuring the project builds, deploys, or compiles correctly.
10. Update README.md and CLAUDE.md in the project root for changes that are functionally noticeable to a user or developer of this codebase. Bug fixes, refactors, internal renaming, and test changes do not require documentation updates unless they change something observable from the outside.
11. If the project uses semver and this change is being released, use the `semver` skill to determine the correct version bump, then update the relevant files. Usually package.json and manifest.json contain semver versions for the project, but check the project CLAUDE.md documentation if in doubt, and add this information to CLAUDE.md if it is not already there. If you are not certain that the version should be updated, ask the user.
12. Ask the user to commit all changes and create a PR. When creating the PR, use the `pull-request` skill to format it correctly. If a requirements GUID was extracted in step 2, pass it to the PR so it can be included as a reference.
13. After the PR is created and you have its number: if the roadmap was updated in step 7, add the PR number to the shipped entry in ROADMAP.md. Find the row for this feature in the Shipped table (match by feature name or GUID) and fill in the PR column with `#[PR number]`. Commit with the message: `Update roadmap: add PR reference for [feature-slug] (#PR)` and push to the PR branch.

## Handling workflow deviations

This workflow is designed to run in a specific sequence with specific agents handling specific roles. **You must never improvise a solution to a deviation.** If anything deviates from the expected workflow — including but not limited to an agent terminating early, an agent reporting a blocker, a step failing unexpectedly, or any situation not explicitly covered by these instructions — you must stop and use `AskUserQuestion` to inform the user of exactly what happened and ask how they would like to proceed.

Specific rules:

- If the implementer agent terminates early or reports a blocker, **do not write the implementation code yourself**. Stop, report what happened, and ask the user how to proceed.
- If the test-writer agent terminates early or reports a blocker, **do not write the tests yourself**. Stop, report what happened, and ask the user how to proceed.
- If any other agent fails, is unavailable, or does not complete its role, **do not take over that role**. Stop, report what happened, and ask the user how to proceed.
- If you are ever uncertain whether a situation is covered by these instructions, ask before acting.

The workflow exists for good reasons. Deviation without user approval undermines those reasons.

## Agent dependencies and handovers

### Dependencies

If the agents described above are not available, stop the session and ask the user to correct this. The intent is that they are located in the user's configuration, not the project.

### Handovers and sequences

The data handovers and sequencing listed in the steps above are:

1. User input (file or chat) -> you clarify or read file -> requirements passed to technical-spec agent (foreground); GUID stored if present in file
2. technical-spec agent analyses codebase, produces spec, confirms with user in its own session -> approved technical spec returned to you
3. You hand the approved spec to BOTH implementer (background, isolated worktree) AND test-writer (background, isolated worktree) — these run in parallel
4. Both agents write files and report what they created -> you commit implementer's work in its worktree, merge into the feature branch, remove its worktree -> then commit test-writer's work in its worktree, merge into the feature branch, remove its worktree -> compare test expectations against implementation and present any design discrepancies to the user for resolution -> run all tests
5. coverage-reviewer checks for coverage tooling -> if found, runs coverage and returns report -> orchestrator presents findings to user, writes any requested tests, commits -> if no coverage tooling, step is skipped
6. code-reviewer reviews and returns findings report -> orchestrator presents findings to user, makes approved fixes, runs tests, commits -> if requirements came from a file and changes were made, append out-of-spec section to requirements file
7. If requirements came from a file and ROADMAP.md exists: feature moved from planned to shipped table (without PR number), sequence renumbered, committed
8. Remaining steps (lint, build, docs, semver, PR with GUID) completed in order
9. If roadmap was updated in step 7: add PR number to shipped entry, commit, push to PR branch

## Commit granularity

Make commits at each stage of the process above. I expect a single commit for the implementation (created when committing the implementer agent's work in its worktree), a single commit for the test implementation (created when committing the test-writer agent's work in its worktree), a single coverage tests commit (if coverage gaps were addressed), a single code review commit, a roadmap update commit (if applicable), a single commit for all remaining steps (lint fixes, docs, semver), and a roadmap PR reference commit after PR creation (if applicable).

## Avoiding repetition

For all tasks above that ask you to run a particular tool, for example, linting, build, mock deployment, etc. look for this information in CLAUDE.md so that you do not have to scan the codebase every time you need project tooling information. If the information is not already in CLAUDE.md, add it. If the information in CLAUDE.md is incorrect, update it with the correct information.

The principle here is that you should minimise repeating the same task, which is slow, takes up more session context, and eats up account usage.

## Code quality principles

These principles apply whenever you write or modify code in this project.

- **Less code is better.** If a change can be made with fewer lines, prefer that. Do not add code speculatively.
- **Keep it DRY.** Never define the same string, value, or logic in more than one place. Use constants and shared utilities.
- **Reuse before creating.** Before writing a new function, class, or module, search the codebase for something that already does what you need. Extend or adapt existing code rather than writing parallel code paths.
- **No unnecessary dependencies.** Do not add a new package or library unless there is genuinely no reasonable way to solve the problem with what is already available.
- **Solve the root problem.** Do not patch symptoms. If something is broken upstream, fix it there rather than working around it downstream.
- **No speculative abstraction.** Do not create new utility functions, base classes, or abstractions unless they are immediately needed by the current task.
