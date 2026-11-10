---
name: commit
description: Short pre-commit routine for Java and Spring Boot changes. Use this skill whenever the user asks to commit, make a commit, or commit the current changes. Runs the review-diff checks, checks by reading that the changes are covered by tests, runs the Gradle tests, then commits. Never pushes.
---

# Pre-commit routine

Run the steps in this order. If a step fails, do not commit. Report what failed and wait for the user.

## 0. Preflight

- Run `git status --porcelain` and `git branch --show-current`.
- If there are no changes, stop and say so.
- **Never commit to `main`** (or the repo's default branch). If you are on it, create a feature branch first with `git switch -c <type>/<short-slug>`, where the name describes the change (for example `feature/add-user-export`). If the name is unclear, ask the user. Name the branch in the report. Commits go to a branch so they can go through code review.
- If `CLAUDE.md` is missing from the repository root, print the warning in the "Missing CLAUDE.md" section below at the start of the report.

## 1. Review all changes

- Scope: every change shown by `git status` that is not ignored by `.gitignore`. This includes modified, deleted and untracked new files, not only what is already staged.
- Read each changed file in full where the change matters: the whole function, class or block, not only the diff hunk. Read new untracked files completely.
- Run the `review-diff` checks on this set of changes.
- A confirmed Critical or High finding blocks the commit. Medium, Low and Info findings are reported but do not block.
- Stage the reviewed files by explicit path. Do not stage files the user did not ask for, and do not stage files that are ignored.

## 2. Check test coverage by reading

- Judge coverage by reading the changed code and the tests that exercise it. Check that each new or changed branch, validation rule and error path has a test.
- Name the untested parts in the report. Do not block the commit on this step unless the user says so.
- Say clearly that this is an estimate by reading, not a measured coverage number.

## 3. Run the tests

- Find the test command in this order:
  1. The command documented in `CLAUDE.md`, if it exists.
  2. A command detected from the project files: `gradlew.bat test` (PowerShell) or `./gradlew test` (Git Bash or Unix) for Gradle; `npm test` for Node; `pytest` for Python; `go test ./...` for Go; `cargo test` for Rust; `mvn test` for Maven.
  3. If no command is clear, ask the user. Do not commit without a test run.
- All tests must pass. Any failure blocks the commit. Report the failing test names and output. Do not label any failure as "known".

## 4. Lint reminder

- If the project has no linter or formatter configured, say so in the report and suggest one as a next step. Do not add one during a commit.

## 5. Commit

- Write a short message: a summary line in the imperative mood, and a body only if the change needs an explanation.
- End the message with:
  `Co-Authored-By: Claude Haiku 5.5 <noreply@anthropic.com>`
- Do not amend existing commits unless the user asks. Do not use `--no-verify`.
- **Never run `git push`.** The user pushes by hand.

## Missing CLAUDE.md

If `CLAUDE.md` is not in the repository root, put this at the top of the report, in capitals:

```
WARNING: NO CLAUDE.md FOUND. PLEASE ADD ONE (for example with /init).
WITHOUT IT:
- the test command is guessed from project files and may be wrong;
- project rules and documented design decisions are unknown, so the review may flag intentional choices as defects;
- the build and test setup cannot be recorded for future runs;
- the review cannot separate documented decisions from real problems.
```

## Report

Write the report in the user's language:

- the result of each step;
- the branch name, and whether a new branch was created;
- the commit hash and message, if a commit was made;
- anything that was not checked, including the coverage estimate and the missing-`CLAUDE.md` warning when it applies.
