---
name: review-diff
description: Fast pre-merge or pre-commit check of local code changes (uncommitted work, a branch or a single commit) in any project. Use when the user asks to review diff or check changes before a merge or commit. Focuses on newly introduced bugs, regressions, security flaws and, where relevant, transaction and data-access problems. For a GitHub pull request review, use the code-review skill instead.
---

# Fast review of changes

Review only the changes: find new bugs, regressions and security holes introduced by this diff. Never modify code.

Language-specific checks are in `java-checks.md` next to this file. Open it only when the changed files are Java, and only the sections that match the changed code.

## Golden rule of context

Never draw a conclusion from a few lines of diff alone. If a function or method changed, read the whole function. If a change touches a transaction or a shared resource, read the whole class and its callers. If a data type or DTO changed, find where it is used.

## Workflow

1. **Determine the scope.**
   - If the user names a branch or PR, compare it with the base branch: `git diff <base>...<branch>`. Find the base from `git symbolic-ref refs/remotes/origin/HEAD`, or use `main` or `master`.
   - If the user names a commit, review that commit.
   - If the user names nothing, review all uncommitted work: `git diff` (unstaged), `git diff --cached` (staged), and the untracked files listed by `git status --porcelain`. If the current branch is a feature branch with commits, also review `git diff <base>...HEAD`.
   - Never use `git diff <base>...HEAD` while on the base branch itself. It is empty there and gives a false "no changes" result.
2. **List the changed files.** Ignore files matched by `.gitignore`. Skip test files unless the user asks about them; for test changes, only check whether the change is covered. Read every untracked new file in full, because `git diff` does not show it.
3. **For each changed function or method**, read the whole unit, then its callers if they affect the result.
4. **Look first for:**
   - new bugs and regressions: behaviour changed in a way other code does not expect;
   - security: unvalidated input reaching a query, command, file path or URL; secrets in code, logs or responses; extra data exposed in responses;
   - data consistency: partial writes without a transaction, race conditions, missing uniqueness constraints;
   - resource handling: missing timeouts on external calls, unbounded queries, unclosed resources;
   - language-specific points from `java-checks.md` when the changed code is Java.
5. **Verify each finding** using the golden rule. If you cannot confirm it, mark it **unverified** and say how to verify it.

## Report

Keep it short. Write the report in the user's language .

```
## Verdict
One sentence with the reason.

## Findings (most severe first)
- [Severity] `path/file:line` — what is wrong, why it matters, how to fix. Status: confirmed / unverified.

## Checked
Changed units read in full, and the checks that were applied.
```

If there are no findings, say so explicitly ("No new findings") and list what was checked. Never add findings to make the report look fuller.

## Rules

- Do not change code without a request. Suggest the fix in text.
- Do not run the test suite without a request. To test a hypothesis you may run one test, but say what you ran and show the result.
- Do not confuse style with bugs. Formatting is out of scope.
- For architectural choices, offer at most two or three options with one line of pros and cons each, and recommend one.
