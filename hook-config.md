# Hook Configuration for Continuous Guard

## Claude Code Hook Setup

Add to `.claude/settings.json` in your project:

```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Edit|Write",
        "command": "bash tests/experience/run.sh --changed-only 2>&1 | tail -20"
      }
    ]
  }
}
```

## Test Runner Script Template

```bash
#!/bin/bash
# tests/experience/run.sh
# Experience test runner with structured logging

set -euo pipefail

LOG_DIR="tests/experience/logs"
CURRENT_LOG="${LOG_DIR}/current.jsonl"

# Parse args
MODE="all"
while [[ $# -gt 0 ]]; do
  case $1 in
    --changed-only) MODE="changed"; shift ;;
    --all) MODE="all"; shift ;;
    --log-dir) LOG_DIR="$2"; shift 2 ;;
    *) shift ;;
  esac
done

# Clear current log for fresh run
mkdir -p "${LOG_DIR}/history"
> "$CURRENT_LOG"

# Detect test framework
if [ -f "pytest.ini" ] || [ -f "pyproject.toml" ]; then
  if [ "$MODE" = "changed" ]; then
    CHANGED=$(git diff --name-only HEAD 2>/dev/null | grep -E '\.(py)$' || true)
    if [ -z "$CHANGED" ]; then
      echo '{"status":"skip","reason":"no relevant changes"}' >> "$CURRENT_LOG"
      exit 0
    fi
    pytest tests/experience/ -x --tb=short -q 2>&1
  else
    pytest tests/experience/ --tb=short -q 2>&1
  fi
elif [ -f "vitest.config.ts" ] || [ -f "vitest.config.js" ] || [ -f "jest.config.ts" ]; then
  if [ "$MODE" = "changed" ]; then
    npx vitest run tests/experience/ --changed 2>&1 || npx jest tests/experience/ --changedSince=HEAD 2>&1
  else
    npx vitest run tests/experience/ 2>&1 || npx jest tests/experience/ 2>&1
  fi
else
  # Fallback: run shell-based tests
  for test_file in tests/experience/atomic/*.sh tests/experience/journeys/*.sh; do
    [ -f "$test_file" ] && bash "$test_file"
  done
fi

# Archive current log
if [ -s "$CURRENT_LOG" ]; then
  TS=$(date +%Y-%m-%dT%H-%M-%S)
  cp "$CURRENT_LOG" "${LOG_DIR}/history/${TS}.jsonl"
fi

# Print summary for AI
echo ""
echo "=== Experience Test Summary ==="
if [ -s "$CURRENT_LOG" ]; then
  PASS=$(grep -c '"status":"pass"' "$CURRENT_LOG" 2>/dev/null || echo 0)
  FAIL=$(grep -c '"status":"fail"' "$CURRENT_LOG" 2>/dev/null || echo 0)
  echo "PASS: $PASS | FAIL: $FAIL"
  if [ "$FAIL" -gt 0 ]; then
    echo ""
    echo "Failed tests:"
    grep '"status":"fail"' "$CURRENT_LOG" | jq -r '.test_id' 2>/dev/null || grep '"status":"fail"' "$CURRENT_LOG"
    exit 1
  fi
else
  echo "No test output generated"
fi
```

## AI Monitoring Commands

### Real-time monitoring (for long test suites)
```bash
# Watch all test completions
tail -f tests/experience/logs/current.jsonl | grep --line-buffered '"phase":"summary"'

# Watch failures + errors
tail -f tests/experience/logs/current.jsonl | grep -E --line-buffered '"status":"(fail|error)"'

# Watch everything (verbose)
tail -f tests/experience/logs/current.jsonl | jq --unbuffered '.'
```

### Post-run analysis
```bash
# Count pass/fail
jq -s '[.[] | select(.phase=="summary")] | group_by(.status) | map({(.[0].status): length}) | add' tests/experience/logs/current.jsonl

# Find slowest tests
jq -s '[.[] | select(.phase=="summary")] | sort_by(-.total_duration_ms) | .[0:5] | .[] | {test_id, total_duration_ms}' tests/experience/logs/current.jsonl

# Get all diagnostics from failures
jq 'select(.phase=="diagnostic")' tests/experience/logs/current.jsonl
```
