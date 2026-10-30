# Reflection

Answers to the four reflection questions for Lab 5: Static Code Analysis.

---

## Q1. What kinds of issues did each tool detect, and how do they differ from one another?

Each tool targets a distinct layer of code quality:

- **Bandit** focuses exclusively on **security vulnerabilities**. It found four issues in `online_exam.py`: a hardcoded password, use of the weak MD5 hash algorithm, use of a pseudo-random generator where a cryptographically secure one is preferable, and a shell-injection risk from `os.system`. Bandit cares nothing about formatting or naming — only about whether the code could be exploited.

- **Pylint** performs deep **semantic and style analysis**. It found missing docstrings, naming-convention violations (camelCase instead of snake_case), a dangerous mutable default argument that causes a subtle runtime bug, an unnecessary `elif` after a `return`, and the `range(len(...))` anti-pattern. Pylint quantifies code quality with a numeric score (5.65 → 10.00), making progress measurable.

- **Flake8** is a lightweight **PEP 8 style enforcer**. It flagged four purely cosmetic issues: a missing blank line, missing spaces around an operator, an inline comment placed too close to code, and a line exceeding 79 characters. Flake8 does not understand the program's logic at all — it only checks surface-level formatting.

Together the three tools form a complementary safety net: Bandit catches what could be exploited, Pylint catches what could cause bugs or be hard to maintain, and Flake8 ensures the code is consistently readable.

---

## Q2. What was the most important (or most surprising) issue you found, and why?

The most **important** issue was the **shell-injection vulnerability** (Bandit B605, High severity) on line 56:

```python
os.system("ssh " + labPc + " lock-screen")
```

If `labPc` were ever supplied by an untrusted source (e.g. a web form or student-editable config file), an attacker could pass a value like `"; rm -rf /"` and execute arbitrary commands on the server. This is a classic OS command injection vulnerability (CWE-78). The fix — using `subprocess.run` with a list — completely eliminates the injection surface because the shell is never invoked.

The most **surprising** issue was the **dangerous mutable default argument** (`penalties={}`). On the surface the code looks correct, but Python evaluates default argument values once at function definition time, not on every call. This means the same `{}` object is shared across all calls to `gradeStudent`, so penalty counts from earlier students silently leak into later ones — a subtle data-corruption bug that would be almost invisible in testing without knowing the pattern.

---

## Q3. How does static analysis contribute to code quality and long-term maintainability?

Static analysis provides an **automated, objective, and repeatable** code review at virtually zero cost. Its contributions include:

1. **Early defect detection** — Issues are caught before the code is run, deployed, or reviewed by a human, when they are cheapest to fix.
2. **Consistency enforcement** — Style rules (PEP 8, snake_case naming, docstrings) are applied uniformly across the entire codebase regardless of who wrote a given function, making the code easier for any team member to read and modify.
3. **Security hygiene** — Bandit flags entire categories of known-bad patterns (hardcoded secrets, unsafe calls, weak cryptography) automatically, acting as a first line of defence that does not rely on developers memorising a security checklist.
4. **Measurable quality gates** — A Pylint score and clean tool reports can be enforced in CI/CD pipelines, preventing regressions from being merged silently.
5. **Built-in learning** — Seeing Pylint explain *why* `range(len(...))` is discouraged, or Bandit explain CWE-78, teaches better practices on the job with minimal friction.

---

## Q4. What could go wrong if these issues were left unfixed?

| Issue left unfixed | Potential consequence |
|---|---|
| `os.system("ssh " + labPc + ...)` | Remote code execution / OS command injection if `lab_pc` is ever user-controlled. |
| Hardcoded `PROCTOR_PASSWORD` | Any developer with repository read access (or anyone who finds the repo online) knows the exam-system password, enabling impersonation of a proctor or manipulation of results. |
| MD5 for answer comparison | MD5 collisions are trivially computed; a student could craft a wrong answer that hashes identically to the correct one, gaining undeserved marks. |
| Mutable default `penalties={}` | Penalty counts accumulate across calls in the same process; a student graded first unfairly inflates the penalty totals of students graded later, corrupting records. |
| camelCase names / no docstrings | Future maintainers spend extra time deciphering the API; inconsistency compounds as the codebase grows, increasing integration bug risk. |
| Flake8 formatting violations | Individually minor, but consistently messy code erodes readability and generates noise in code diffs; automated formatting checks in CI will fail. |

In a real online examination system — where grades, academic integrity, and student data are at stake — any one of these vulnerabilities or bugs could have serious academic, legal, or reputational consequences.
