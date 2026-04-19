# BugHound Mini Model Card (Reflection)

Fill this out after you run BugHound in **both** modes (Heuristic and Gemini).

---

## 1) What is this system?

**Name:** BugHound

**Purpose:** BugHound is a small agentic tool that inspects short Python snippets, detects likely code problems, proposes a fix, and evaluates whether that fix is safe enough to auto-apply.

**Intended users:** Students, developers, and anyone learning how agentic workflows combine heuristic rules with model-based assistance for code review and reliability.

---

## 2) How does it work?

BugHound follows a simple agentic loop:

- **Plan:** The app initializes the workflow and decides it will scan the snippet, generate issues, propose a fix, and assess risk.
- **Analyze:** In heuristic mode, it uses local pattern checks for `print(`, bare `except:`, and `TODO`. In Gemini mode, it sends a structured prompt to the model and expects a JSON array of issue objects.
- **Act:** If issues exist, the agent either uses a heuristic fixer offline or calls Gemini to rewrite the code. The fixer prompt instructs the model to preserve behavior and make minimal changes.
- **Test:** The `assess_risk` function compares original and fixed code for structural changes, removed returns, modified bare except blocks, and now overly long rewrites.
- **Reflect:** Based on the risk score and guardrails, BugHound decides whether the fix appears safe enough for auto-application or if human review is recommended.

Heuristic mode is used offline with no API key, and Gemini mode is used when `GEMINI_API_KEY` is available and selected.

---

## 3) Inputs and outputs

**Inputs:**

- `sample_code/cleanish.py`: a clean function with logging and a simple add function.
- `sample_code/mixed_issues.py`: a function with `TODO`, `print`, and a bare `except:` block.
- `sample_code/print_spam.py`: a simple function containing `print` statements.
- `sample_code/flaky_try_except.py`: a function with a bare `except:` that hides errors.

The inputs were short Python snippets with functions, simple control flow, and one or more heuristic issues.

**Outputs:**

- Detected issues were usually categorized as `Code Quality`, `Reliability`, or `Maintainability`.
- Heuristic mode flagged clear patterns like `print` statements, bare `except:`, and TODO comments.
- Gemini mode could generate more nuanced or additional issue descriptions, but it still had to return parseable JSON.
- Fixes proposed by the agent included replacing `print` with `logging.info`, turning bare `except:` into `except Exception as e:`, and adding handling comments.
- The risk report showed a score, level, whether auto-fix was recommended, and explicit reasons.

---

## 4) Reliability and safety rules

1. **Large rewrite penalty**
   - What it checks: whether fixed code is more than 1.5× longer than the original.
   - Why it matters: large rewrites are more likely to introduce behavior changes or subtle bugs, so this raises caution.
   - False positive: a legitimate refactor that adds good structure or documentation can be penalized even though it is correct.
   - False negative: a small but risky rewrite could still pass if it does not change length enough.

2. **Removed `return` penalty**
   - What it checks: whether the original code had `return` but the fixed code does not.
   - Why it matters: removing a return changes behavior in a widely observable way, so this is a strong safety signal.
   - False positive: if the original `return` was dead code or unreachable, the penalty might still apply incorrectly.
   - False negative: a behavior-changing fix that preserves `return` syntax but changes semantics would not be caught.

Other important rules include penalizing much shorter fixed code and flagging bare `except:` modifications.

---

## 5) Observed failure modes

1. **Model output parsing failure**
   - If the Gemini analyzer returns text that is not valid JSON, BugHound falls back to heuristics. The existing `MockClient` demonstrates this by returning a string instead of JSON.
   - What went wrong: the system expects a strict JSON array and can only recover by using the fallback analyzer.

2. **Over-editing risk**
   - Some fixes can change code structure more than expected, especially when a model rewrites a short snippet with extra logging or added branches.
   - What went wrong: without a guardrail, the risk scorer could still allow auto-fix on a large rewrite, so the agent might appear overconfident.

A third observed issue is that heuristics may miss subtle problems Gemini could catch, while Gemini can produce ambiguous or inconsistent issue objects.

---

## 6) Heuristic vs Gemini comparison

- **Gemini detection:** Gemini mode can generate more expressive issue descriptions and might catch patterns beyond the hard-coded heuristic checks. However, its output must still follow the exact JSON format or BugHound will fallback.
- **Heuristics:** Heuristic mode consistently catches `print` statements, bare `except:`, and `TODO` comments. It is predictable but limited.
- **Fix differences:** Heuristic fixes are simple rewrites using regex substitutions and logging imports. Gemini fixes can be more elaborate, but that also makes them riskier.
- **Risk scorer alignment:** The risk scorer generally agreed that large or structurally changed fixes deserve caution, especially after the new over-editing penalty.

---

## 7) Human-in-the-loop decision

Scenario: if the analysis finds a bare `except:` block and the proposed fix rewrites the code substantially or makes it much longer, BugHound should refuse to auto-fix and require review.

- Trigger: new risk condition for `should_autofix = False` when a large rewrite is detected or when the model output is parseable but the issue list is unusually broad.
- Implementation: `risk_assessor.py` is the best place because it centralizes safety decisions.
- Message: "This change involves a large rewrite or ambiguous behavior. Review is recommended before applying." 

---

## 8) Improvement idea

Add a stricter model-output validation step in the analyzer that checks not only for parseable JSON, but also that each issue object contains `type`, `severity`, and `msg` fields with expected value types. If validation fails, fall back to heuristics and log the mismatch.

This is low complexity, improves reliability by reducing silent misinterpretation, and preserves the existing fallback behavior.

