# BugHound Mini Model Card (Reflection)

Fill this out after you run BugHound in **both** modes (Heuristic and Gemini).

---

## 1) What is this system?

**Name:** BugHound  
**Purpose:** Analyze a Python snippet, propose a fix, and run reliability checks before suggesting whether the fix should be auto-applied.

**Intended users:** Students learning agentic workflows and AI reliability concepts.

---

## 2) How does it work?

Describe the workflow in your own words (plan → analyze → act → test → reflect).  
Include what is done by heuristics vs what is done by Gemini (if enabled).

**Answer:**

BugHound runs 5 steps on every snippet:

1. **Plan** — logs that it's about to scan the code and find a fix.
2. **Analyze** — looks for issues. No LLM: uses simple heuristics (checks for `print(`, bare `except:`, `TODO`). With Gemini: sends the code and asks for a list of issues back.
3. **Act** — proposes a fix. No LLM: does a heuristic text-replace fix. With Gemini: asks it to rewrite the code.
4. **Test** — `assess_risk()` compares old vs new code and gives a score, a risk level, and reasons.
5. **Reflect** — decides if the fix is safe to auto-apply, or if a human should check it first.

Heuristics run when there's no LLM client (or the mock client). Gemini runs when it's configured and gives back valid output. Either way, the same risk check runs afterward on the result.

---

## 3) Inputs and outputs

**Inputs:**

- What kind of code snippets did you try?
- What was the “shape” of the input (short scripts, functions, try/except blocks, etc.)?

**Outputs:**

- What types of issues were detected?
- What kinds of fixes were proposed?
- What did the risk report show?

**Answer:**

**Inputs:** the 4 sample files in `sample_code/`:

- `print_spam.py` — three `print()` calls
- `flaky_try_except.py` — bare `except:`, no `with` block around a file read
- `mixed_issues.py` — a `TODO`, a `print()`, and a bare `except:` together
- `cleanish.py` — no issues, already uses `logging`

All short, single functions, under 10 lines. No multi-function files or classes tested.

**Outputs:**

- Issues found: "Code Quality" (prints), "Reliability" (bare except), "Maintainability" (TODOs) in heuristic mode. Gemini used its own labels (e.g. "Resource Management") and gave more specific reasons, like noting a file stays open if `read()` fails.
- Fixes proposed: heuristics just text-replace (`print(` → `logging.info(`, bare `except:` → `except Exception as e:`). Gemini rewrote more naturally, e.g. using a `with open(...) as f:` block.
- Risk report: every run gave a score, level, reasons, and `should_autofix`. The clean file scored 100/low. Every file with a real issue scored 30-55 (medium/high) and never got auto-fix approval.

---

## 4) Reliability and safety rules

List at least **two** reliability rules currently used in `assess_risk`. For each:

- What does the rule check?
- Why might that check matter for safety or correctness?
- What is a false positive this rule could cause?
- What is a false negative this rule could miss?

**Answer:**

**Rule 1 — Severity deductions:** each issue subtracts points by severity (High -40, Medium -20, Low -5).

- *Why it matters:* worse problems should push the score down more, so risky fixes get flagged harder than cosmetic ones.
- *False positive:* several unrelated Low issues (e.g. three separate prints) stack up and can push a harmless snippet into "medium risk."
- *False negative:* if the analyzer misses an issue entirely, it costs zero points — the report looks safer than it is.

**Rule 2 — String-literal check:** it checks if a string or docstring that used to contain `print(` or `except:` disappeared from the fixed code's strings — a sign the fixer text-replaced inside a string instead of real code.

- *Why it matters:* the heuristic fixer does blind text replacement, so it can't tell a docstring from real code. This catches that mistake.
- *False positive:* legitimately rewording a docstring (and dropping the word "print(") gets penalized even though nothing broke.
- *False negative:* it only watches for `print(` and `except:`. Any other kind of string corruption slips through.

---

## 5) Observed failure modes

Provide at least **two** examples:

1. A time BugHound missed an issue it should have caught  
2. A time BugHound suggested a fix that felt risky, wrong, or unnecessary  

For each, include the snippet (or describe it) and what went wrong.

**Answer:**

1. **Gemini missed 2 of 3 issues on `print_spam.py`.** The file has three `print()` calls. Gemini's fix only converted the first one to `logger.info(...)` and left the other two prints untouched. It looked finished but wasn't.

2. **Heuristic fix quietly changes behavior.** The heuristic swap turns `print("Hello", name)` into `logging.info("Hello", name)`. That's not the same thing — `logging.info` treats the second value as a format arg, not something to print alongside the first. The output changes silently, and `assess_risk()` doesn't catch it because it only checks code size and string-literal rewrites, not argument meaning.

---

## 6) Heuristic vs Gemini comparison

Compare behavior across the two modes:

- What did Gemini detect that heuristics did not?
- What did heuristics catch consistently?
- How did the proposed fixes differ?
- Did the risk scorer agree with your intuition?

**Answer:**

- **Gemini caught more:** on `flaky_try_except.py` it also flagged that the file never gets closed if `read()` fails — heuristics don't check for that at all.
- **Heuristics caught the basics reliably:** `print(`, bare `except:`, `TODO` got caught every time, with no API call needed.
- **Fixes differed in style:** heuristics do a blanket text swap everywhere; Gemini writes more natural code (like a `with` block) but sometimes only fixes part of the file, as seen above.
- **Did the risk scorer agree with me?** Mostly. Every risky fix scored medium/high and none auto-applied. But it scored the "Gemini only fixed 1 of 3 prints" case about the same as the "solid fix, just policy-blocked" case (~55 vs ~45), even though the first one feels like the worse problem.

---

## 7) Human-in-the-loop decision

Describe one scenario where BugHound should **refuse** to auto-fix and require human review.

- What trigger would you add?
- Where would you implement it (risk_assessor vs agent workflow vs UI)?
- What message should the tool show the user?

**Answer:**

**Scenario:** the fix only partially solves the flagged issues — like the `print_spam.py` case, where Gemini fixed 1 of 3 prints and left the rest.

- **Trigger:** count how many times the flagged pattern (e.g. `print(`) still shows up in the fixed code. If it didn't drop to zero, don't auto-apply.
- **Where to add it:** in `risk_assessor.py`, inside `assess_risk()` — it already has the code and the issue list.
- **Message to user:** "BugHound found N issue(s), but the fix doesn't look like it resolved all of them. Please review before applying."

---

## 8) Improvement idea

Propose one improvement that would make BugHound more reliable *without* making it dramatically more complex.

Examples:

- A better output format and parsing strategy
- A new guardrail rule + test
- A more careful “minimal diff” policy
- Better detection of changes that alter behavior

Write your idea clearly and briefly.

**Answer:**

Add a "fix completeness" check to `assess_risk()`: count how many times a flagged pattern (like `print(`) still appears in the fixed code. If it's not gone, dock points and add the reason "Fix did not address all instances of the flagged issue." This directly fixes the incomplete-Gemini-fix problem from section 5. It's a small addition — a few lines, testable with the existing `print_spam.py` sample — and needs no new dependencies or prompt changes.
