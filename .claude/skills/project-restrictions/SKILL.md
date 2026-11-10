---
name: project-restrictions
description: Hard limits for any change to this project. Apply whenever the task would create, modify, delete or move files, run commands, or touch Git or remote systems. Covers the project boundary, forbidden Git operations, and rules for keeping changes small and respecting the existing codebase.
---

# Project restrictions

These are hard limits. They apply to every change, in every dialog. Dialog behaviour (plans, language, teaching) is in the `dialog-rules` skill.

## 1. Project scope

The current project is the ONLY allowed workspace.

Allowed inside the current project directory, after the user approves the plan:

- Create, modify and delete files and directories required by the task.
- Refactor existing code.
- Run project-related commands, tests and build commands.
- Install dependencies required by the approved task.

Forbidden:

- Creating, modifying or deleting files outside the current project.
- Modifying global system configuration.
- Modifying files in the user's home directory.
- Modifying other repositories or projects.
- Running commands that affect external projects or environments.
- Pushing code, creating remote pull requests or modifying remote repositories without explicit approval.

If an operation may affect anything outside the project, STOP and ask for explicit approval.

Reading external documentation and inspecting information is allowed. Changing external resources is not.

## 2. Git and project safety

- Inspect the Git working tree before making changes.
- Preserve existing uncommitted user changes.
- Never discard user changes.
- Never overwrite unrelated work.
- Never run destructive Git commands without explicit approval: `git reset --hard`, `git clean`, force push, history rewriting.
- Do not create commits automatically. The only exception: the user explicitly invokes the `commit` skill (`/commit`). That invocation counts as approval to commit, and the steps of the `commit` skill replace the plan.
- Do not push code automatically.
- Do not merge branches automatically.
- Do not create remote pull requests without approval.

If existing user changes may be affected, stop and explain the risk.

## 3. Change discipline

- Inspect the existing project structure before proposing changes.
- Follow the existing architecture, naming conventions and code style.
- Look for reusable components and existing implementations first.
- Change only what the task requires. Avoid unnecessary refactoring.
- Do not introduce new dependencies without a clear reason.
- Do not redesign the architecture unless the task requires it.
- Do not invent files, classes, methods or project behaviour. Check that they exist.
- Prefer the smallest safe change that solves the problem.