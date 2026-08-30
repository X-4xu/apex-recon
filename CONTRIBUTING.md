# Contributing to APEX

Thank you for your interest in contributing to **APEX**! We welcome bug reports, improvements, and feature pull requests from the security research community.

---

## Code Philosophy & Standards

1. **Single-File Distribution Integrity:**
   `apex.sh` is designed to be entirely self-contained. Any helper functions, regex patterns, or analyzers must reside cleanly within the single file without introducing hard runtime dependencies.

2. **Defensive Bash Architecture:**
   * Always maintain `set -uo pipefail`.
   * Never use unbounded variables or unquoted expansions.
   * Avoid silent error suppression (do not use `|| true` on core execution paths; use `apex_exec` which logs proper return codes).
   * Ensure local variables are declared with `local` inside functions to prevent variable leakage.

3. **Evidence-First Standard:**
   * Any new finding module MUST store verifiable evidence in `db/evidence.ndjson` using `record_evidence`.
   * Findings without content signatures or cryptographic proof must be capped at `NEEDS_REVIEW` to preserve zero-false-positive guarantees.

4. **Secret Redaction:**
   * Never output raw credentials, API keys, or JWT tokens to stdout or logs. All sensitive strings must flow through `redact_stream`.

---

## Submitting a Pull Request

1. Fork the repository and create a descriptive branch:
   ```bash
   git checkout -b feature/new-signature-module
   ```
2. Validate syntax and run ShellCheck:
   ```bash
   shellcheck apex.sh
   bash apex.sh doctor
   ```
3. Test dry-run and scope parsing:
   ```bash
   bash apex.sh scan example.com --authorized=true --dry_run=true
   ```
4. Submit a Pull Request clearly describing the problem solved and the evidence validation logic implemented.

---

## Reporting Vulnerabilities

If you discover a security issue or flaw in APEX itself (e.g. scope evasion or unintended command injection), please open a private GitHub security advisory or contact the maintainer directly.
