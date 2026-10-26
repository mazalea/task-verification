# Task A reproduction: sanitize-git-repo

**Date:** 2026-10-26
**Benchmark:** Terminal-Bench 2.0, task `sanitize-git-repo`
**Known defect:** Delphik board, issue #84 (verifier crashes when git history is rewritten)
**Environment:** Docker image `alexgshaw/sanitize-git-repo:20251031` (the image Harbor uses), run interactively with `tests/` mounted at `/tests`
**Grader:** `tests/test_outputs.py`

## Goal
Show that the same correct sanitization passes or crashes the grader depending only on whether git history is rewritten.

## Step 0: Reference solution
Command:
```
harbor run -d terminal-bench/terminal-bench-2 -a oracle --include-task-name "terminal-bench/sanitize-git-repo"
```
Result: reward 1.0, 1 trial, 0 exceptions, runtime 46s.
The reference solution only edits files with `sed` and skips `.git`, so it never rewrites history.

## Step 1: Start the task environment
```
docker run -it --rm -v "${PWD}\tests:/tests" alexgshaw/sanitize-git-repo:20251031 bash
pip install pytest gitpython
```
Note: building the Dockerfile locally gave an empty `/app/dclm`, so I used the prebuilt image instead.

## Step 2: Apply the correct fix (history untouched)
```
cp /tests/ray_cluster.yaml ray_processing/ray_cluster.yaml
cp /tests/process.py ray_processing/process.py
cp /tests/rw_v2_fasttext_openhermes_vs_rw_v2_bigram_0.1_arcade100k.json exp_data/datasets/tokenized/
```

## Step 3: Run the grader (history intact)
```
pytest /tests/test_outputs.py -rA
```
Output:
```
==================================================================== PASSES ====================================================================
=========================================================== short test summary info ============================================================
PASSED ../../tests/test_outputs.py::test_removal_of_secret_information
PASSED ../../tests/test_outputs.py::test_correct_replacement_of_secret_information
PASSED ../../tests/test_outputs.py::test_no_other_files_changed
============================================================== 3 passed in 0.78s ===============================================================
```
Result: all 3 tests passed

## Step 4: Remove the secrets from git history
```
git checkout --orphan clean
git add -A
git commit -m "sanitized"
git branch -D main
git branch -m main
git remote remove origin
git reflog expire --expire=now --all
git gc --prune=now
```

## Step 5: Run the grader again
```
pytest /tests/test_outputs.py -rA
```
Output:
```
==================================================================== PASSES ====================================================================
=========================================================== short test summary info ============================================================
PASSED ../../tests/test_outputs.py::test_removal_of_secret_information
PASSED ../../tests/test_outputs.py::test_correct_replacement_of_secret_information
FAILED ../../tests/test_outputs.py::test_no_other_files_changed - ValueError: SHA b'd6987af002b122fef54bc0be402062c76488a4d9' could not be resolved, git returned: b'd6987af002b122fef54bc0be402062c76488a4d9...
========================================================= 1 failed, 2 passed in 0.32s ==========================================================
```
Result: the first two tests pass, and test_no_other_files_changed fails with a ValueError.

## Conclusion
The same fix passes or crashes depending only on whether git history was rewritten.
The instruction says the secrets must not be present in the repository. Rewriting history is the thorough way to do that, but it removes the fixed commit `d6987af...` that `test_no_other_files_changed` looks up, so the grader raises an error instead of grading.

Defect type: grader fails on a valid solution (false negative), caused by a hardcoded commit ID in the test.
Confidence:
- Defect exists: high. Reproduced twice with the same fix (passes with history intact, crashes after rewriting history); the cause is the hardcoded commit ID in test_no_other_files_changed; the reference solution passes in the same environment.
- Real-world impact: medium. I simulated the history rewrite by hand and did not run a real agent that rewrites history. The crash only occurs when old commits are fully removed; a partial cleanup that leaves old references still passes.