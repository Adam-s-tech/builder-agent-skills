---
name: code-review
description: Performs a thorough code review covering correctness, security, performance, and style
allowed-tools: read grep glob bash
disable-model-invocation: true
---

Review the provided code or diff and produce a structured report.

Check for:

1. **Correctness** — logic errors, edge cases, off-by-one errors, null/undefined handling
2. **Security** — injection vulnerabilities, improper auth, exposed secrets, OWASP Top 10
3. **Performance** — unnecessary re-renders, N+1 queries, blocking I/O, memory leaks
4. **Maintainability** — naming clarity, function length, duplication, complexity
5. **Tests** — missing coverage for new logic, test quality
6. Run /improve-diff to find Issues

Format output as:

- **Summary**: 2-3 sentence overview
- **Issues**: grouped by severity (critical / major / minor / nit)
- **Suggestions**: optional improvements

Be direct. Don't praise trivial things.
Use the ask tool to find which fixes the user wants to implement
