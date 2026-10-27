# Task A: agent audit of sanitize-git-repo

**Date:** 2026-10-26
**Task:** Terminal-Bench 2.0, `sanitize-git-repo` (known defect on the Delphik board, issue #84)
**Agent:** Codex CLI, model GPT-6-Luna, signed in with a free ChatGPT account
**Setup:** run in the task folder, read-only audit. I approved only commands that list or read files.
**Blind test:** the prompt did not mention the defect I had already found.

## Prompt I gave the agent
```text
You are auditing a benchmark task for defects. Do not edit any files.
Read instruction.md, the tests in tests/, the reference solution in solution/,
and environment/Dockerfile and setup.sh. Check:
1. Does every test assertion follow from the instruction?
2. Could a valid alternative solution fail the tests?
3. Could a wrong or partial solution pass?
4. Is the answer leaked in the environment?
5. Does the reference solution pass the tests?
6. Is anything fragile (versions, network, timing)?
Output: verdict (defect/valid), defect type, evidence with file and line,
confidence. Quote the exact line for every claim.
You may run read-only commands to list and read files. Do not modify, delete, install or download anything.
```

## Agent's answer
```text
Verdict: defect
Defect type: Incomplete test coverage; environment depends on network access.
1. Do all assertions follow from the instruction?
The assertions for the three listed files follow the request to remove secrets, use consistent placeholders, and avoid unrelated changes. But the tests do not check the entire repository for secrets: instruction.md:11 (instruction.md) says, “Please ensure that the sensitive values are not present in the repository after the sanitization,” while tests/test_outputs.py:14–19 (tests/test_outputs.py) lists only three paths and tests/test_outputs.py:23–31 (tests/test_outputs.py) checks secrets only in those paths.
2–3. Could a valid alternative fail, or a wrong/partial solution pass?
A solution that correctly sanitizes all secrets but modifies an uncontaminated file would fail the test’s exact changed-path restriction: tests/test_outputs.py:60–71 (tests/test_outputs.py) says “no files other than CONTAMINATED_PATHS” may change. That restriction is consistent with instruction.md:12–13 (instruction.md), which says, “not delete or modify any files that are not contaminated with sensitive information.”
A partial solution that sanitizes only the three tested files could pass even if secrets remained elsewhere in the repository. The test’s removal loop iterates only over `CONTAMINATED_PATHS` (tests/test_outputs.py:22–31 (tests/test_outputs.py)). The checked-in solution itself searches repository-wide, suggesting broader scanning is intended: solution/solve.sh:5–13 (solution/solve.sh) uses `grep -r` and `find . -type f`.
4. Is the answer leaked in the environment?
The test source contains the exact values it checks: tests/test_outputs.py:8–12 (tests/test_outputs.py) defines `FAKE_AWS_ACCESS_KEY_ID`, `FAKE_AWS_SECRET_ACCESS_KEY`, `FAKE_GITHUB_TOKEN`, and two Hugging Face tokens. The reference solution also hardcodes secret-specific replacement commands (solution/solve.sh:8–9 (solution/solve.sh)). These establish a leak in the task files; I can’t establish from the inspected environment files alone whether those files are visible to the agent at runtime.
5. Does the reference solution pass?
I did not run tests, per the read-only constraint. By inspection, the solution replaces the tested secret patterns with the expected placeholders (solution/solve.sh:6–13 (solution/solve.sh)); the test compares the contaminated files against sanitized fixtures (tests/test_outputs.py:41–57 (tests/test_outputs.py)). I can’t confirm the Git-diff assertion without executing it against the built repository.
6. Fragility
The environment setup clones a GitHub repository and verifies a specific commit (environment/setup.sh:3–5 (environment/setup.sh)). The test script also runs `apt-get update`, installs curl, and downloads an installer (tests/test.sh:3–8 (tests/test.sh)). These steps depend on external network availability.
Confidence: High on the coverage defect and network dependency; moderate on reference-solution success because tests were not run.
```

## My check of each claim
| Agent claim | What I checked | Verdict |
|---|---|---|
| Overall verdict: "incomplete test coverage" | Compared with my own reproduction. The real defect is that the grader crashes when git history is rewritten (`repo.commit("d6987af...")` in test 3). The agent never mentions it. | Missed the real defect |
| Tests only check 3 files, so a partial solution passes while secrets remain elsewhere (instruction.md:11, test_outputs.py:14-31) | Ran `git grep` for all 5 fake keys on the whole repo. The secrets exist only in the 3 tested files. | Wrong in practice |
| Changing an uncontaminated file fails (test_outputs.py:60-71, instruction.md:12-13) | The agent itself says this is consistent with the instruction. | True, but not a defect |
| "Answer leaked": the keys appear in the test file and solve.sh | Those files are not visible to the agent while it works, only during grading. The agent also says it can't tell. The keys are in the repo files by design. | Misleading |
| Reference solution passes (not run) | I ran it with Harbor: reward 1.0, 0 exceptions, 46s. | True (moderate confidence was right) |
| Fragile: needs network (setup.sh:3-5, test.sh:3-8) | Read test.sh: it runs `apt-get`, installs curl, downloads the `uv` installer. | True, but minor |

The line numbers the agent quoted were accurate.

## Extra check I ran (fresh container, HEAD of /app/dclm)
Command:
```
git grep -il -e <the 5 fake keys>
```
Output: `exp_data/datasets/tokenized/rw_v2_fasttext_openhermes_vs_rw_v2_bigram_0.1_arcade100k.json`, `ray_processing/process.py`, `ray_processing/ray_cluster.yaml`
-> The secrets are only in the 3 files the test checks, so the "partial solution passes" claim is wrong.

Command:
```
git log --oneline -S"AKIA1234567890123456"
```
Output: `52dae12`, `8e04bee`
-> The AWS key is also in git history, and no test checks history.

## Answers to my 3 questions
1. **Did the agent find the crash on a history rewrite?** No.
2. **What did it miss, or find that I didn't?** It missed the crash and the fact that the test ignores git history. It added one true point I had not noted: the grader needs internet access to run.
3. **What sounded sure but was wrong?** It said "Confidence: High" on the coverage defect, which does not hold, because the secrets exist only in the 3 tested files. It also listed the test files as a "leak" even though the agent can't see them while working.

## Takeaway
The agent read the exact lines that contain the defect but never considered that the fixed commit could disappear, and it did not run anything. A reviewer who runs the scenario finds what a read-only agent misses.