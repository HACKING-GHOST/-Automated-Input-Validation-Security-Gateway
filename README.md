# Automated Input Validation and Security Gateway

A modular, CLI-first gateway that normalizes input, evaluates rules, scores risk, makes allow/warn/block decisions, logs structured output, and stores history in SQLite.

## Project Structure

```text
input_gateway/
|- main.py         # CLI entrypoint: scan, history, ai-assess
|- normalizer.py   # Input normalization and Unicode cleanup
|- rules.py        # Rule engine, rule tags, overrides, optional allowlist validators
|- scorer.py       # Weighted risk score aggregation
|- decision.py     # allow/warn/block threshold logic
|- logger.py       # JSONL logs and SQLite persistence
|- config.py       # JSON/YAML config loading and validation
|- ai_assessor.py  # Optional AI second-opinion API integration
`- utils.py        # report/error helpers
gateway/
`- main.py         # wrapper entrypoint that delegates to input_gateway.main
```

## How Scoring Works

Each hit includes:
- `severity`: `low` / `medium` / `high`
- `severity_weight`: from config
- `score`: numeric value added to total risk
- `tags`: internal category tags

Default severity weights:
- `low = 0.33`
- `medium = 0.55`
- `high = 1.75`

Decision thresholds:
- `score >= 1.75` => `block`
- `score >= 0.55` => `warn`
- else => `allow`

## CLI Commands

Scan input:
```bash
python -m input_gateway.main scan --text "SELECT * FROM users"
python -m input_gateway.main scan --file ./sample.txt --explain
```

Show decision history:
```bash
python -m input_gateway.main history --limit 20
```

Run AI assessment only:
```bash
python -m input_gateway.main ai-assess --text "example input"
python -m input_gateway.main ai-assess --file ./sample.txt --config-report ./report.json
```

Alternative wrapper entrypoint:
```bash
python -m gateway.main scan --text "hello"
```

## Configuration (JSON or YAML)

```json
{
  "decision_thresholds": {"block": 1.75, "warn": 0.55},
  "severity_weights": {"low": 0.33, "medium": 0.55, "high": 1.75},
  "max_input_chars": 100000,
  "log_path": "logs/audit.jsonl",
  "db_path": "logs/gateway.db",
  "rule_overrides": {
    "SQLI_KEYWORD": {"severity": "high", "description": "SQLi attempt aligned to current intel"}
  },
  "ai": {
    "enabled": false,
    "provider": "openai-compatible",
    "endpoint": "https://api.openai.com/v1/chat/completions",
    "api_key": "",
    "model": "gpt-5.2-chat",
    "timeout_s": 30
  }
}
```

Notes:
- `ai.enabled` defaults to `false`.
- If `ai.enabled` is `true`, `ai.endpoint`, `ai.model`, and `ai.api_key` are required.
- `ai.api_key` can come from config or environment variable `AIVSG_AI_API_KEY` / `OPENAI_API_KEY`.
- Both `rule_overrides` and legacy `mitre_overrides` are accepted and normalized.
- JSON config files with UTF-8 BOM are supported.

## Rule Engine Behavior

- Default scan uses detection rules (`SQLI_KEYWORD`, `COMMAND_INJECTION`, `XSS_PATTERN`, `PATH_TRAVERSAL`) plus heuristic rules (`LENGTH_ANOMALY`, `SPECIAL_CHAR_DENSITY`, `REPETITION_PATTERN`).
- Format/allowlist validators (`EMAIL_FORMAT`, `INTEGER_ONLY`, etc.) are available for targeted usage through `evaluate_rules(..., active_rule_names=[...])`.
- Invalid regex patterns are safely skipped instead of crashing scans.

## Tests

```bash
python -m pytest -q
```

## Reliability and Hardening Notes

- `scan`, `history`, and `ai-assess` return structured fail-safe block errors on runtime failures.
- AI failures do not crash scans; only validated AI responses can escalate decisions.
- Config validation enforces types, ranges, and threshold ordering.
- Decision and scoring logic handle invalid/non-finite inputs safely.
- Logger operations normalize malformed report fields before persistence.
