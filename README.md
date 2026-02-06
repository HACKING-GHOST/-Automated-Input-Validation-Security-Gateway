# Automated Input Validation & Security Gateway

CLI-first MVP for validating input, scoring risk, and logging decisions.

## Quick Start

```bash
python gateway/main.py --text "SELECT * FROM users"
```

```bash
python gateway/main.py --file ./samples/input.txt
```

## What It Does

1. Accepts input from CLI arguments or files.
2. Normalizes the input (trim + lowercase + whitespace collapse).
3. Validates against built-in rules (SQLi, command injection, XSS, path traversal).
4. Scores risk and decides allow / warn / block.
5. Logs an explainable JSON report.

## Output

Reports are printed to stdout and appended to `logs/audit.jsonl` by default.

## Decisions

- `score >= 51` => `block`
- `score >= 31` => `warn`
- otherwise => `allow`
