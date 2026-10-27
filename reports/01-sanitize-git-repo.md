# Report 01: sanitize-git-repo (Terminal-Bench 2.0)

**Date:** 2026-10-27
**Benchmark / task:** Terminal-Bench 2.0, `sanitize-git-repo`
**Environment:** Windows, Docker Desktop, Harbor v0.24.0, task image `alexgshaw/sanitize-git-repo:20251031`. Agent used for audit: Codex CLI, GPT-6-Luna.
**Related board entry:** Delphik defect hub, issue #84 (status on the board: fix in progress)

## Verdict
Defect confirmed. The grader fails on a valid solution (false negative).

## Defect type
Test overspecified. The test depends on a hardcoded commit ID that disappears when git history is correctly cleaned.

## Summary
The task asks the AI to remove secret API keys from a repository and replace them with placeholders. The instruction says the secret values must not be present "in the repository". The thorough way to do this is to rewrite git history. If the AI does that, the grader crashes with a `ValueError` instead of grading, because one test looks up a fixed old commit that no longer exists.

## Evidence
1. **Instruction** (`instruction.md`, line 11): "Please ensure that the sensitive values are not present in the repository after the sanitization."
2. **Test** (`tests/test_outputs.py`, lines 60 to 71): `test_no_other_files_changed` calls `repo.commit("d6987af002b122fef54bc0be402062c76488a4d9")` and then `commit.diff(None)`. If that commit does not exist, the call raises an error.
3. **The secrets are also in history.** In a fresh container, `git log --oneline -S"AKIA1234567890123456"` returns two commits: `52dae12` and `8e04bee`. Both are older than `d6987af`.
4. **Consequence.** In git, a commit ID depends on the commits before it. Rewriting either leaking commit changes the ID of `d6987af`, so any proper history cleanup removes the commit the test looks for. I confirmed with `git merge-base --is-ancestor` that both 8e04bee and 52dae12 are ancestors of d6987af.
5. **Reference solution** (`solution/solve.sh`): it edits files with `sed` and skips `.git`, so it never touches history. It passes (reward 1.0, 0 exceptions, 46 s), but it leaves the secrets in history.

## Steps to reproduce
Full commands and output are in `notes/taskA-reproduction.md`. In short:
1. Start the container: `docker run -it --rm -v "${PWD}\tests:/tests" alexgshaw/sanitize-git-repo:20251031 bash`
2. Copy the three sanitized files from `/tests` into the repo and run `pytest /tests/test_outputs.py -rA`. Result: all 3 tests pass.
3. Rewrite history (orphan commit, delete `main`, remove the `origin` remote, expire the reflog, `git gc --prune=now`) and run the tests again. Result: test 3 fails with "ValueError: SHA b'd6987af002b122fef54bc0be402062c76488a4d9' could not be resolved, git returned: b'd6987af002b122fef54bc0be402062c76488a4d9...(truncated)".

## Root cause
`test_no_other_files_changed` hardcodes one commit ID from the original history. A fix that follows the instruction thoroughly removes that commit from the repository.

## Impact
- A correct, thorough solution is rejected (false negative).
- A solution that edits only the three files passes, even though the secrets are still recoverable from git history.
- If the cleanup leaves old references (for example the `origin/main` remote branch), the old commit still exists and the test passes. The crash happens only when the cleanup is complete.

## Proposed fix
Do not depend on one hardcoded commit. For example, take a snapshot of the file list before the AI starts and compare against that, or compare file contents against the expected sanitized versions. The upstream fix is tracked as pull request #86, which I have not reviewed.

## Where the audit agent was right or wrong
I ran Codex CLI as a blind audit. It missed this defect. It claimed secrets might remain in other files, but a repo-wide `git grep` shows they exist only in the three tested files. It was right that the grader needs network access, and its line numbers were accurate. Details: `notes/taskA-agent-audit.md`.

## Limitations
- The instruction says the secrets must not be "present in the repository" but does not say whether git history counts. If it means only the current files, rewriting history is optional. Even so, the instruction never forbids it, and a history rewrite does not modify any uncontaminated file in the working tree. A grader should not crash on it. The task should also state which meaning it intends.
- I rewrote history by hand to reproduce the crash. I did not run an AI agent that does it on its own.

## Confidence
- **Defect exists: high.** I reproduced it twice (passes with history intact, crashes after rewriting it), and the cause is one identifiable line in the test.
- **Real-world impact: medium.** I rewrote history by hand and did not run an agent that does it. The crash also depends on the cleanup being complete.