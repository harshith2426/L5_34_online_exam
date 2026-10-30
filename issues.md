# Known Issues Table

Issues identified by Pylint, Bandit, and Flake8 in `online_exam.py`, ordered by severity (High → Low).

| Issue | Type | Line(s) | Description | Fix Approach |
|---|---|---|---|---|
| Shell injection via `os.system` | Security – Bandit B605 (High) | 56 | `os.system("ssh " + labPc + " lock-screen")` concatenates user input into a shell command, enabling OS command injection (CWE-78). | Replaced with `subprocess.run(["ssh", lab_pc, "lock-screen"], check=True)` — passes arguments as a list, never invokes a shell. |
| Weak MD5 hash used | Security – Bandit B324 (High) | 16 | `hashlib.md5()` is a cryptographically broken algorithm; Bandit flags it as a security risk. | Added `usedforsecurity=False` parameter and `# nosec B324` to explicitly declare non-security use. |
| Hardcoded password in source | Security – Bandit B105 (Low) | 11 | `PROCTOR_PASSWORD = "proctor-demo"` embeds a plain-text credential in code visible to anyone with repository access. | Replaced with `os.environ.get("PROCTOR_PASSWORD", "")` so the secret is loaded from an environment variable at runtime. |
| Non-cryptographic `random` used | Security – Bandit B311 (Low) | 20 | `random.choice()` uses a predictable pseudo-random generator, unsuitable for security-sensitive selections. | Replaced with `secrets.randbelow(len(QUESTIONS))` from the `secrets` module (OS-level CSPRNG). |
| Dangerous mutable default argument | Bug – Pylint W0102 | 23 | `def gradeStudent(studentName, responses, penalties={})` — the dict literal is shared across all calls, so penalty counts accumulate between students. | Changed default to `None`; added `if penalties is None: penalties = {}` inside the function body. |
| camelCase function/argument names | Style – Pylint C0103 | 15, 19, 23, 35, 45, 54 | `hashAnswer`, `pickBonusQuestion`, `gradeStudent`, `getGrade`, `classAverage`, `lockScreen`, `studentName`, `labPc` all violate PEP 8 snake_case convention. | Renamed all functions and parameters to snake_case (e.g. `hash_answer`, `grade_student`, `get_grade`). |
| Missing module and function docstrings | Style – Pylint C0114/C0116 | 1, 15, 19, 23, 35, 45, 51, 54, 59 | No module docstring; no docstrings on any of the seven functions, reducing readability and Pylint score. | Added a module-level docstring and a one-line docstring to every function. |
| Unnecessary `elif` after `return` | Style – Pylint R1705 | 37 | After a `return`, the subsequent `elif` is logically unreachable and adds unnecessary nesting. | Replaced with consecutive `if` / bare `return` to flatten the branching logic. |
| `range(len(...))` anti-pattern | Style – Pylint C0200 | 25 | `for i in range(len(QUESTIONS))` is less Pythonic than using `enumerate`. | Changed to `for i, question in enumerate(QUESTIONS)`. |
| Missing whitespace around `=` | Formatting – Flake8 E225 | 67 | `average=classAverage()` has no spaces around the `=` operator. | Changed to `average = class_average()`. |
| Too few blank lines before function | Formatting – Flake8 E302 | 54 | Only one blank line before `lockScreen`; PEP 8 requires two blank lines between top-level definitions. | Added a second blank line above `lock_screen`. |
| Inline comment too close to code | Formatting – Flake8 E261 | 69 | `print("Topper:", topper()) # highest score` — only one space before `#`; PEP 8 requires at least two. | Changed to two spaces before `#`. |
| Line too long (> 79 characters) | Formatting – Flake8 E501 | 70 | `print("Students who sat the exam:", ...)` was 93 characters wide. | Split across multiple lines using implicit continuation inside parentheses. |
