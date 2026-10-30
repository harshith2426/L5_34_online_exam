# Lab 5: Static Code Analysis — Lab Report
**File analysed:** `online_exam.py`  
**Tools used:** Pylint 4.1.3, Bandit 1.9.4, Flake8 7.5.0  
**Python version:** 3.11.5

---

## Known Issues Table

| Issue | Type | Line(s) | Description | Fix Approach |
|---|---|---|---|---|
| Weak MD5 hash used for security | Security (Bandit B324 – High) | 16 | `hashlib.md5()` is a cryptographically broken hash algorithm; Bandit flags its use as a security vulnerability. | Replaced with `hashlib.md5(..., usedforsecurity=False)` to explicitly mark it non-security use, and added `# nosec B324`. |
| Shell-injection via `os.system` | Security (Bandit B605 – High) | 56 | `os.system("ssh " + labPc + " lock-screen")` concatenates user-supplied input into a shell command, enabling command-injection attacks. | Replaced `os.system` with `subprocess.run(["ssh", lab_pc, "lock-screen"], check=True)`, which passes arguments as a list and does not invoke a shell. |
| Hardcoded password in source | Security (Bandit B105 – Low) | 11 | `PROCTOR_PASSWORD = "proctor-demo"` embeds a plain-text credential directly in the code. Anyone with repository access can read it. | Replaced with `os.environ.get("PROCTOR_PASSWORD", "")` so the password is loaded from an environment variable at runtime. |
| Non-cryptographic `random` for selection | Security (Bandit B311 – Low) | 20 | `random.choice()` uses a predictable pseudo-random generator. Bandit flags it because it is unsuitable for security-sensitive choices. | Replaced with `secrets.randbelow(len(QUESTIONS))` from the `secrets` module, which uses the OS-provided CSPRNG. |
| Dangerous mutable default argument | Bug (Pylint W0102) | 23 | `def gradeStudent(studentName, responses, penalties={})` — a dict literal as a default argument is shared across all calls, causing data to accumulate unexpectedly between invocations. | Changed default to `None` and initialised `penalties = {}` inside the function body when `None` is received. |
| camelCase function and argument names | Style (Pylint C0103) | 15, 19, 23, 35, 45, 54 | Python's PEP 8 and Pylint require `snake_case` for function and parameter names. Names such as `hashAnswer`, `pickBonusQuestion`, `gradeStudent`, `getGrade`, `classAverage`, `lockScreen`, and `studentName`/`labPc` all violate this convention. | Renamed all functions and parameters to `snake_case` (e.g., `hash_answer`, `grade_student`, `get_grade`, `class_average`, `lock_screen`). |
| Missing docstrings | Style (Pylint C0114/C0116) | 1, 15, 19, 23, 35, 45, 51, 54, 59 | No module-level docstring, and no docstrings on any of the seven functions. Pylint deducts points for each missing docstring. | Added a module docstring and a one-line docstring to every function explaining its purpose. |
| Unnecessary `elif` after `return` | Style (Pylint R1705) | 37 | After a `return` statement, an `elif` is logically unreachable and adds unnecessary nesting. Pylint suggests using a plain `if` instead. | Replaced `elif` and `else` with consecutive `if` / bare `return` statements to flatten the logic. |
| `range(len(...))` anti-pattern | Style (Pylint C0200) | 25 | Iterating with `for i in range(len(QUESTIONS))` and then indexing is less readable and Pythonic than using `enumerate`. | Replaced with `for i, question in enumerate(QUESTIONS)` and accessed `question["answer"]` directly. |
| Missing whitespace around `=` | Formatting (Flake8 E225) | 67 | `average=classAverage()` has no spaces around the assignment operator, violating PEP 8. | Changed to `average = class_average()`. |
| Too few blank lines before function | Formatting (Flake8 E302) | 54 | Only one blank line preceded the `lockScreen` function definition; PEP 8 requires two blank lines between top-level definitions. | Added a second blank line above `lock_screen`. |
| Inline comment too close to code | Formatting (Flake8 E261) | 69 | `print("Topper:", topper()) # highest score` — only one space before `#`; PEP 8 requires at least two spaces. | Changed to `print("Topper:", topper())  # highest score` (two spaces). |
| Line too long (> 79 characters) | Formatting (Flake8 E501) | 70 | The `print("Students who sat the exam:", ...)` statement was 93 characters wide, exceeding the PEP 8 limit of 79. | Broke the call across multiple lines using Python's implicit line continuation inside parentheses. |

---

## Reflection Questions

### 1. What kinds of issues did each tool detect, and how do they differ from one another?

Each tool targets a distinct layer of code quality:

- **Bandit** focuses exclusively on **security vulnerabilities**. It found four issues in `online_exam.py`: a hardcoded password, use of the weak MD5 hash algorithm, use of a pseudo-random generator where a cryptographically secure one is preferable, and a shell-injection risk from `os.system`. Bandit cares nothing about formatting or naming — only about whether the code could be exploited.

- **Pylint** performs deep **semantic and style analysis**. It found missing docstrings, naming-convention violations (camelCase instead of snake_case), a dangerous mutable default argument that causes a subtle runtime bug, an unnecessary `elif` after a `return`, and the `range(len(...))` anti-pattern. Pylint quantifies code quality with a numeric score, making progress measurable.

- **Flake8** is a lightweight **PEP 8 style enforcer**. It flagged four purely cosmetic issues: a missing blank line, missing spaces around an operator, an inline comment placed too close to code, and a line exceeding 79 characters. Flake8 does not understand the program's logic at all — it only checks surface-level formatting.

Together, the three tools form a complementary safety net: Bandit catches what could be exploited, Pylint catches what could cause bugs or be hard to maintain, and Flake8 ensures the code is consistently readable.

---

### 2. What was the most important (or most surprising) issue you found, and why?

The most important issue was the **shell-injection vulnerability** (Bandit B605, High severity) on line 56:

```python
os.system("ssh " + labPc + " lock-screen")
```

If `labPc` were supplied by an untrusted source (e.g., a web form or a config file that a student could edit), an attacker could pass a value such as `"; rm -rf /"` and execute arbitrary commands on the server. This is a classic **OS command injection** vulnerability (CWE-78).

The fix — replacing `os.system` with `subprocess.run` and passing arguments as a list — completely eliminates the injection surface because the shell is never invoked; each element of the list is passed directly to the OS as a separate argument.

The most *surprising* issue was the **dangerous mutable default argument** (`penalties={}`). On the surface the code looks correct, but Python evaluates default argument values once when the function is defined, not on every call. This means the same `{}` object is reused across all calls to `grade_student`, so penalty counts from earlier students would leak into later ones — a subtle bug that would be very hard to spot in testing without knowing the pattern.

---

### 3. How does static analysis contribute to code quality and long-term maintainability?

Static analysis provides an **automated, objective, and repeatable** code review at virtually zero cost. Its benefits include:

1. **Early defect detection** — Issues are caught before the code is run, deployed, or reviewed by another human, when they are cheapest to fix.
2. **Consistency enforcement** — Style rules (PEP 8, snake_case naming, docstrings) are applied uniformly across the entire codebase regardless of who wrote a given function, making the code easier for any team member to read.
3. **Security hygiene** — Bandit flags entire categories of known-bad patterns (hardcoded secrets, unsafe calls, weak cryptography) automatically, acting as a first line of defence that does not rely on developers memorising a security checklist.
4. **Living documentation** — A Pylint score and clean tool reports serve as a measurable quality gate in CI/CD pipelines, preventing regressions from being merged silently.
5. **Knowledge transfer** — Seeing Pylint explain *why* `range(len(...))` is discouraged, or Bandit explain CWE-78, is a low-friction way for developers to learn better practices on the job.

---

### 4. What could go wrong if these issues were left unfixed?

| Issue left unfixed | Potential consequence |
|---|---|
| `os.system("ssh " + labPc + ...)` | Remote code execution / OS command injection if `lab_pc` is ever user-controlled. |
| Hardcoded `PROCTOR_PASSWORD` | Any developer with repository read access (or anyone who finds the repo online) knows the exam-system password, enabling them to impersonate a proctor or manipulate exam results. |
| MD5 for answer hashing | MD5 collisions are trivially computed; an adversarial student could craft a wrong answer that produces the same hash as the correct one, gaining undeserved marks. |
| Mutable default `penalties={}` | Penalty counts accumulate across calls in the same process run; a student graded first would unfairly affect the penalty totals of students graded later, producing incorrect records. |
| camelCase names / no docstrings | Future maintainers spend extra time deciphering the API; inconsistency compounds as the codebase grows, increasing the risk of integration bugs. |
| Flake8 formatting violations | Individually minor, but consistently messy code erodes readability. In a team setting it generates noise in code diffs and makes automated formatting checks fail in CI. |

In a real online examination system — where grades, academic integrity, and student data are at stake — any one of these vulnerabilities or bugs could have serious academic, legal, or reputational consequences.

---

## Tool Score Summary

| Tool | Before | After |
|---|---|---|
| Pylint | 5.65 / 10 | **10.00 / 10** |
| Bandit | 4 issues (2 High, 2 Low) | **0 issues** |
| Flake8 | 4 violations | **0 violations** |
