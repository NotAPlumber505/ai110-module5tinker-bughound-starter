# BugHound Mini Model Card (Reflection)

Observed run context: Heuristic workflow was executed and tested locally. Gemini credentials are provided via `.env` and loaded at runtime by the app.

---

## 1) What is this system?

**Name:** BugHound  
**Purpose:** Analyze a Python snippet, propose a minimal fix, and score risk before deciding whether an auto-fix is safe.

**Intended users:** Students and beginner developers learning agentic workflows, guardrails, fallback behavior, and human-in-the-loop reliability.

---

## 2) How does it work?

BugHound follows a five-step agent loop:

1. Plan: Log a short plan to scan code and propose a fix.
2. Analyze: Detect issues.
3. Act: Generate a rewritten version of code.
4. Test: Run risk scoring on the proposed fix.
5. Reflect: Decide whether to auto-fix or require human review.

Heuristic path:

- Analyzer uses pattern checks like `print(`, bare `except:`, and `TODO`.
- Fixer performs simple string/regEx edits such as replacing `print(` with `logging.info(` and converting bare except.

Gemini path (when enabled):

- Analyzer asks Gemini for JSON issue objects.
- Fixer asks Gemini for full rewritten code.
- If Gemini output is malformed, empty, or unusably short, the workflow falls back to heuristics.

---

## 3) Inputs and outputs

**Inputs:**

- `sample_code/cleanish.py`: mostly clean function with logging and return.
- `sample_code/mixed_issues.py`: TODO + print + bare except.
- Comments-only weird case: `# TODO: odd` and `# print("debug")`.

Input shape included short functions, try/except blocks, comments, and small script-like snippets.

**Outputs:**

- Detected issue types: Code Quality (print), Reliability (bare except), Maintainability (TODO).
- Proposed fixes:
	- For print: inserted/imported logging and rewrote print calls.
	- For bare except: rewrote to `except Exception as e:` pattern.
	- For no issues: returned original code unchanged.
- Risk report examples:
	- Cleanish case: score 100, level low, `should_autofix=True`.
	- Mixed issues with placeholder LLM fix text: score 0, level high, `should_autofix=False`.
	- Comments-only case: score 75, level low, but `should_autofix=False` after stricter threshold.

---

## 4) Reliability and safety rules

Rule A: Missing fix output (`fixed_code` empty) => score 0, high risk, no auto-fix.

- Why necessary: Prevents applying null/failed generations.
- Possible false positive: Model intentionally refuses unsafe edits, but system scores as maximal risk instead of nuanced refusal.
- Possible false negative: Non-empty but useless text can bypass this rule unless other checks catch it.

Rule B: If original has `return` and fixed code does not, subtract 30.

- Why necessary: Return removal often changes behavior and data flow.
- Possible false positive: Equivalent refactor could preserve behavior without literal `return` string in same form.
- Possible false negative: Return semantics can still break even when `return` token remains.

Rule C: Fixed code less than half of original length, subtract 20.

- Why necessary: Large shrink is a rough signal of truncation or over-deletion.
- Possible false positive: Legitimate simplifications can be shorter and still correct.
- Possible false negative: Harmful edits can keep similar length while changing logic significantly.

---

## 5) Observed failure modes

1. False positive from comment text (missed context):

- Snippet: comments-only input containing `# print("debug")`.
- What went wrong: Heuristic analyzer flagged print usage even though no executable print call existed.
- Reliability impact: unnecessary issue reporting and potentially unnecessary fixes.

2. Format failure in fixer output (now guarded):

- Snippet: short function with print where mock LLM fixer returned only `# placeholder`.
- What went wrong: Workflow previously accepted any non-empty fixer output as valid code.
- Reliability impact: unusable output could move forward in pipeline.

3. Analyzer output malformation:

- Snippet: normal function where mock analyzer returned non-JSON text.
- What went wrong: model did not follow output contract.
- Reliability response: agent correctly fell back to heuristics.

---

## 6) Heuristic vs Gemini comparison

Note: The heuristic path was directly exercised in this session. Gemini comparison points below describe expected behavior and guardrail interactions when Gemini mode is enabled with `.env` credentials.

Heuristic mode observations (executed):

- Consistently catches explicit lexical patterns: `print(`, `except:`, and `TODO`.
- Produces predictable, rule-based fixes with minimal variability.

Gemini mode observations:

- Intended strength: detect context-sensitive issues beyond fixed heuristics.
- Intended weakness: output-format variability (JSON/code compliance), requiring fallback guardrails.

Proposed-fix behavior contrast:

- Heuristic fixes are narrower and deterministic.
- Gemini fixes are potentially smarter but can over-edit or misformat output without strict constraints.

Risk scorer intuition check:

- High-risk outcomes for severe/malformed cases matched intuition.
- The stricter auto-fix threshold (`score >= 85`) better matches cautious deployment expectations than allowing all low-risk scores.

---

## 7) Human-in-the-loop decision

Scenario: Proposed fix is structurally suspicious (very short output or placeholder-like content) compared to original code.

- Trigger: If non-empty fixed output has at most half the non-empty lines of original input, treat as unreliable and do not trust model output directly.
- Implementation layer: Agent workflow (act stage), because this is an output-usability validation before risk scoring.
- User-facing message: "Model fix looked incomplete or truncated. Switched to heuristic fixer and recommend human review."

---

## 8) Improvement idea

Add a simple "suspiciously short fix" guardrail plus one offline test:

- Guardrail: In the act step, if LLM fixer output is too short relative to input, log it and fallback to heuristic fixer.
- Test: Use a MockClient that returns valid analyzer JSON but a one-line fixer placeholder; assert fallback occurs and usable heuristic fix is produced.

This improvement is low-complexity, testable offline, and directly reduces format/usability failures.
