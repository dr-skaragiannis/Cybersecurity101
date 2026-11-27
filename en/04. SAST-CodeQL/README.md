# 04. SAST with GitHub CodeQL

A **Static Application Security Testing (SAST)** lab. You push a small, intentionally vulnerable Flask application ([plexicus/simplest-sast-vulnerable](https://github.com/plexicus/simplest-sast-vulnerable)) to your own GitHub repository, enable **CodeQL**, and read the findings the way a security engineer would. You then check the libraries for known **CVEs** (SCA), fix the code, and verify that the alerts close.

> 🇬🇷 **Greek version:** [04. SAST-CodeQL/README.md](../../04.%20SAST-CodeQL/README.md)

Unlike the CTF labs, you do **not** attack a machine here. The analysis only reads the source code; it never runs it.

---

## Material

| File | Contents |
| :--- | :--- |
| [Lab 04 - SAST-CodeQL.pdf](../../04.%20SAST-CodeQL/Lab%2004%20-%20SAST-CodeQL.pdf) | The full exercise (51 pages, in Greek): theory (SAST, CWE, CVE, CVSS, GHSA, NVD), enabling CodeQL, analysing an alert, CVEs in dependencies, false negatives, writing up a finding, the 10 README vulnerabilities, 7 hands-on exercises, fixing and re-scanning every vulnerability (Δ0–Δ11), 10 investigative questions. |
| [Lab 04 - SAST-CodeQL - Οδηγός.pdf](../../04.%20SAST-CodeQL/Lab%2004%20-%20SAST-CodeQL%20-%20Οδηγός.pdf) | Step-by-step guide (25 pages, in Greek): setup with screenshots, reading the alert list and an alert page, search filters, 10 search questions, fixing the 2 CVEs and the 3 most severe alerts. |

**Reference repositories:**

- [dr-skaragiannis/simplest-sast-beginners](https://github.com/dr-skaragiannis/simplest-sast-beginners): the original code (commit `77baa1b`) with the guide's scan, the CVE fixes and the 3 code fixes.
- [dr-skaragiannis/simplest-sast-vulnerable-lab](https://github.com/dr-skaragiannis/simplest-sast-vulnerable-lab): the hands-on exercises and the fix for every vulnerability, one commit per fix.

**Requirements:** a (free) github.com account, `git`, a browser. The analysis runs on GitHub's servers. Bandit and `pip-audit` need Python 3.

---

## Key concepts

| Term | Meaning |
| :--- | :--- |
| **SAST** | Analysis of source code without running it. CodeQL turns the code into a database and runs queries against it, one per vulnerability type. |
| **Alert / Rule** | A finding at a specific file and line / the rule that produced it, e.g. `py/sql-injection`. |
| **Source / Sink** | Where user-controlled data enters (`request.args.get`) / the dangerous point where it ends up (`cursor.execute`). |
| **CWE** | MITRE's catalogue of weakness **types**, e.g. CWE-89 (SQL Injection). |
| **CVE / GHSA** | The identifier of one **specific** vulnerability in a specific product / its entry in the GitHub Advisory Database. |
| **Security severity** | A 0–10 score per rule: Critical ≥ 9.0, High 7.0–8.9, Medium 4.0–6.9, Low < 4.0. |
| **SCA** | Checking dependencies (libraries) for known CVEs, e.g. Dependabot, `pip-audit`. |

---

## Lab flow

### 1. Your own repository

```bash
git clone https://github.com/plexicus/simplest-sast-vulnerable.git
cd simplest-sast-vulnerable
git remote set-url origin https://github.com/<username>/simplest-sast-beginners.git
git push -u origin main
```

The new repository must be **Public** (code scanning is free there) and **empty** (no README, .gitignore or license).

![The repository after the push](../../04.%20SAST-CodeQL/images/01-repo.png)

### 2. Enable CodeQL

`Settings → Advanced Security → Code scanning → CodeQL analysis → Set up → Default`. Check that the languages are **JavaScript/TypeScript** and **Python** and the query suite is **Default**, then click **Enable CodeQL**.

### 3. The run

The **Actions** tab shows a **CodeQL Setup** workflow with one analysis job per language. Our scan finished in **1m 6s** (CodeQL 2.27.6, 43 Python rules, 87 JavaScript rules).

![The CodeQL Setup run](../../04.%20SAST-CodeQL/images/02-codeql-run.png)

### 4. The alerts

`Security and quality → Code scanning`. The scan of the original code produces **13 alerts**: 11 Python, 2 JavaScript.

| # | Rule | Location | Score | CWE |
| :--- | :--- | :--- | :--- | :--- |
| 5 | `py/command-line-injection` | `app.py:60` | Critical 9.8 | 78, 88 |
| 1 | `py/template-injection` | `app.py:68` | Critical 9.3 | 74 |
| 6 | `py/sql-injection` | `app.py:52` | High 8.8 | 89 |
| 7–10 | `py/reflective-xss` | `app.py:54, 68, 106, 129` | High 7.8 | 79, 116 |
| 12, 13 | `js/xss-through-dom` | `static/js/app.js:6, 13` | High 7.8 | 79, 116 |
| 11 | `py/path-injection` | `app.py:77` | High 7.5 | 22, 23, 36, 73, 99 |
| 4 | `py/flask-debug` | `app.py:133` | High 7.5 | 215, 489 |
| 2, 3 | `py/stack-trace-exposure` | `app.py:81, 92` | Medium 5.4 | 209, 497 |

**Search filters:** `is:open`, `severity:critical`, `rule:py/reflective-xss`, `language:python`, `tool:CodeQL`. The same filters are available from the *Language*, *Tool*, *Rule* and *Severity* menus.

### 5. Analysing an alert

An alert page shows the rule, the sink line and, via **Show paths**, the data flow. For the SQL injection (#6):

`request.args.get('user', '')` (line 49, **source**) → `username` → `query` (line 51, f-string) → `cursor.execute(query)` (line 52, **sink**)

![The /sqli route: source on line 49, sink on line 52](../../04.%20SAST-CodeQL/images/04-sqli-source-sink.png)

Each rule's documentation (`codeql.github.com/codeql-query-help/<language>/<rule-id>/`) lists its security severity, precision, CWEs and query suites. **Severity** (`error`/`warning`) is a **different** field from security severity: `js/xss-through-dom` is a `warning` yet High 7.8.

![Documentation for py/command-line-injection](../../04.%20SAST-CodeQL/images/03-rule-help.png)

### 6. What CodeQL did not find

Of the 10 vulnerability types in the README, the Default suite does **not** report: insecure deserialization (the data flow is lost at `bytes.fromhex`), hardcoded secrets (the rule has low precision), MD5 (not treated as sensitive data), XXE (the input comes from `sys.argv`, not from HTTP) and insecure randomness. Zero alerts does **not** mean secure code. That is why the exercise compares the results with a second tool, **Bandit** (24 findings: 4 High, 7 Medium, 13 Low).

---

## CVEs in dependencies (SCA)

CodeQL examines your own code. For the libraries in `requirements.txt`:

```bash
pip install pip-audit
pip-audit -r requirements.txt
```

There are **2** real CVEs. `pyyaml 6.0` and `psycopg2-binary 2.9.9` have no advisories, and transitive dependencies are installed at their latest, clean, version.

| CVE / GHSA | Package | Fix | Score | CWE | Commit |
| :--- | :--- | :--- | :--- | :--- | :--- |
| [CVE-2026-41066](https://github.com/advisories/GHSA-vfmq-68hx-4jfw) / GHSA-vfmq-68hx-4jfw | `lxml` | 4.9.3 → **6.1.0** | High 7.5 | 611 (XXE) | [`cd7037d`](https://github.com/dr-skaragiannis/simplest-sast-beginners/commit/cd7037d341f41b287648298a9b1ae5956bb0fa3d) |
| [CVE-2026-27205](https://github.com/advisories/GHSA-68rp-wp8r-4726) / GHSA-68rp-wp8r-4726 | `Flask` | 2.3.3 → **3.1.3** | Low 2.3 | 524 | [`3fb9aff`](https://github.com/dr-skaragiannis/simplest-sast-beginners/commit/3fb9aff70a21a80a7fa803138c84fd6960a2ab62) |

![The lxml advisory](../../04.%20SAST-CodeQL/images/05-ghsa-lxml.png)

> ⚠️ **Upgrading lxml is not enough.** Version 6.1.0 only changes the *default* to `resolve_entities='internal'`. `xml_parser.py`, however, **explicitly** sets `XMLParser(resolve_entities=True)`, so the XXE remains until it is changed to `resolve_entities=False`. After every upgrade, read the advisory's *Patches* section.

---

## Fixing the code: the 3 most severe alerts

One commit per fix. After each push, wait for the new analysis and check the **Closed** tab. Open alerts: **13 → 11 → 9 → 8**.

### F1. SQL Injection → parameterized query (#6, High 8.8)

```diff
-    query = f"SELECT * FROM users WHERE username = '{username}'"
-    cursor.execute(query)
+    query = "SELECT * FROM users WHERE username = ?"
+    cursor.execute(query, (username,))
```

The value is sent **separately** from the SQL and can never change its syntax. Test `/sqli?user=' OR '1'='1`: before, it returned `admin`; now it returns `[]`. Alerts **#6** and **#7** closed (the XSS on line 54 went through the same `query` variable). Commit [`afb8d9c`](https://github.com/dr-skaragiannis/simplest-sast-beginners/commit/afb8d9c88b2114b41a4c2588517cc5101aa277c1).

![F1: the SQL injection fix commit](../../04.%20SAST-CodeQL/images/06-fix-sqli.png)

### F2. Template Injection → input as a template variable (#1, Critical 9.3)

```diff
-    template = f'<h1>Hello, {name}!</h1><p>Welcome to our site.</p>'
-    return render_template_string(template)
+    template = '<h1>Hello, {{ name }}!</h1><p>Welcome to our site.</p>'
+    return render_template_string(template, name=name)
```

The template is now constant and the input is passed as a **value**, with autoescaping. `{{7*7}}` is shown as text instead of `49`, and `<script>` becomes `&lt;script&gt;`. Alerts **#1** and **#8** closed. Commit [`df74be8`](https://github.com/dr-skaragiannis/simplest-sast-beginners/commit/df74be8b24f7d3d170a449c45ef6b02d503242f0).

![F2: the template injection fix commit](../../04.%20SAST-CodeQL/images/07-fix-ssti.png)

### F3. Command Injection → subprocess with an argument list (#5, Critical 9.8)

```diff
 import os
+import subprocess
 ...
-    output = os.popen(f'ls -la {filename}').read()
+    output = subprocess.run(['ls', '-la', '--', filename], capture_output=True, text=True).stdout
```

Without a shell, `filename` is a **single** argument, and `;`, `&`, `|` no longer separate commands. The `--` stops a name that starts with `-` from becoming an `ls` option (CWE-88). Test `/cmd?file=nofile & echo INJECTED`: before, it printed `INJECTED`; now it prints nothing. Alert **#5** closed. Commit [`61071fa`](https://github.com/dr-skaragiannis/simplest-sast-beginners/commit/61071fab956e16341bb2dcc31c855294eae8f3f2).

![F3: the command injection fix commit](../../04.%20SAST-CodeQL/images/08-fix-cmd.png)

> 💡 The fix that adds an `import` was done **last**, because the new line shifts all the code below it. The remaining 8 alerts kept their numbers and only moved down one line.

Fixing **every** vulnerability (Δ0–Δ11, including those CodeQL did not detect) is covered in the exercise PDF, with one commit per fix in [simplest-sast-vulnerable-lab](https://github.com/dr-skaragiannis/simplest-sast-vulnerable-lab/commits/main).

---

## Self-assessment questions

1. How many open alerts did CodeQL report, and how many per language?
2. Which alerts are Critical? Number, rule, location, security severity.
3. Which rule appears most often, and on which lines?
4. Which line of `app.py` has two different alerts?
5. For the SQL injection: where is the source, which variables does the value pass through, and where is the sink?
6. What security severity, precision and CWE does `py/sql-injection` have?
7. For the path injection: which parameter controls the file, and how many CWE tags does the rule have?
8. Which alert has no *Show paths*, and why?
9. Which alerts are Medium, and what is returned to the user?
10. Why is `js/xss-through-dom` a `warning` yet shown as High?

---

## References

- **Target repository:** [plexicus/simplest-sast-vulnerable](https://github.com/plexicus/simplest-sast-vulnerable)
- **CodeQL:** [About code scanning with CodeQL](https://docs.github.com/en/code-security/code-scanning/introduction-to-code-scanning/about-code-scanning-with-codeql), [CodeQL query help](https://codeql.github.com/codeql-query-help/)
- **Catalogues:** [CWE (MITRE)](https://cwe.mitre.org/), [GitHub Advisory Database](https://github.com/advisories), [NVD](https://nvd.nist.gov/), [FIRST CVSS 3.1 Calculator](https://www.first.org/cvss/calculator/3.1)
- **Tools:** [Bandit](https://bandit.readthedocs.io/), [pip-audit](https://github.com/pypa/pip-audit), [Dependabot](https://docs.github.com/en/code-security/dependabot)
- **OWASP:** [Top 10](https://owasp.org/Top10/), [SQL Injection Prevention Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/SQL_Injection_Prevention_Cheat_Sheet.html), [OS Command Injection Defense Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/OS_Command_Injection_Defense_Cheat_Sheet.html)
