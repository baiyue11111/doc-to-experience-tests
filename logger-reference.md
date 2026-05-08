# Experience Test Logger — Reference Implementation

## Python (pytest)

```python
import json
import time
import os
import platform
import traceback
from datetime import datetime, timezone
from pathlib import Path
from contextlib import contextmanager
from dataclasses import dataclass, field, asdict
from typing import Any

@dataclass
class ExperienceLogger:
    test_id: str
    log_dir: Path = field(default_factory=lambda: Path("tests/experience/logs"))
    _step: int = field(default=0, init=False)
    _events: list = field(default_factory=list, init=False)
    _start_time: float = field(default=0, init=False)

    def __post_init__(self):
        self.log_dir.mkdir(parents=True, exist_ok=True)
        self._start_time = time.time()
        self._current_file = self.log_dir / "current.jsonl"

    def _emit(self, phase: str, action: str, **kwargs):
        self._step += 1
        event = {
            "test_id": self.test_id,
            "ts": datetime.now(timezone.utc).isoformat(),
            "phase": phase,
            "step": self._step,
            "action": action,
            **kwargs
        }
        self._events.append(event)
        line = json.dumps(event, ensure_ascii=False, default=str)
        with open(self._current_file, "a") as f:
            f.write(line + "\n")
        return event

    def setup(self, action: str, detail: dict = None):
        return self._emit("setup", action, detail=detail or self._env_info())

    def execution(self, action: str, **kwargs):
        return self._emit("execution", action, **kwargs)

    def assertion(self, action: str, field_name: str, expected: Any, actual: Any):
        passed = expected == actual
        return self._emit("assertion", action,
                         field=field_name, expected=expected, actual=actual, passed=passed)

    def teardown(self, action: str, detail: dict = None):
        return self._emit("teardown", action, detail=detail)

    def diagnostic(self, action: str, data: dict):
        return self._emit("diagnostic", action, data=data)

    def summary(self, status: str):
        total_ms = int((time.time() - self._start_time) * 1000)
        assertions = [e for e in self._events if e["phase"] == "assertion"]
        return self._emit("summary", "final",
                         status=status,
                         total_duration_ms=total_ms,
                         assertions_passed=sum(1 for a in assertions if a.get("passed")),
                         assertions_failed=sum(1 for a in assertions if not a.get("passed")))

    def collect_diagnostics(self, error: Exception = None):
        import psutil
        self.diagnostic("env_snapshot", {
            "cpu_usage_pct": psutil.cpu_percent(),
            "memory_free_mb": psutil.virtual_memory().available // (1024*1024),
            "disk_free_gb": psutil.disk_usage('/').free // (1024**3),
        })
        if error:
            self.diagnostic("error_detail", {
                "error_type": type(error).__name__,
                "message": str(error),
                "stack": traceback.format_exc(),
            })

    def save_history(self):
        ts = datetime.now().strftime("%Y-%m-%dT%H-%M-%S")
        history_dir = self.log_dir / "history"
        history_dir.mkdir(exist_ok=True)
        dest = history_dir / f"{ts}_{self.test_id}.jsonl"
        with open(dest, "w") as f:
            for event in self._events:
                f.write(json.dumps(event, ensure_ascii=False, default=str) + "\n")

    @staticmethod
    def _env_info():
        return {
            "runtime": f"python{platform.python_version()}",
            "os": platform.system().lower(),
            "arch": platform.machine(),
        }

@contextmanager
def experience_test(test_id: str):
    logger = ExperienceLogger(test_id=test_id)
    logger.setup("init_env")
    try:
        yield logger
        logger.summary("pass")
    except Exception as e:
        logger.collect_diagnostics(e)
        logger.summary("fail")
        raise
    finally:
        logger.save_history()
```

### Usage in pytest

```python
from experience_logger import experience_test

def test_create_resource():
    with experience_test("atomic_001_create_resource") as log:
        # Log the request
        log.execution("http_request", input={
            "method": "POST",
            "url": "/api/resources",
            "body": {"name": "test"}
        })

        # Make the actual call
        response = client.post("/api/resources", json={"name": "test"})

        # Log the response
        log.execution("http_response", output={
            "status": response.status_code,
            "body": response.json(),
            "duration_ms": response.elapsed.total_seconds() * 1000
        })

        # Assert with logging
        log.assertion("assert_eq", "status", expected=201, actual=response.status_code)
        assert response.status_code == 201
```

## JavaScript/TypeScript (Vitest/Jest)

```typescript
import { writeFileSync, appendFileSync, mkdirSync, existsSync } from 'fs';
import { join } from 'path';
import os from 'os';

interface LogEvent {
  test_id: string;
  ts: string;
  phase: 'setup' | 'execution' | 'assertion' | 'teardown' | 'diagnostic' | 'summary';
  step: number;
  action: string;
  [key: string]: any;
}

class ExperienceLogger {
  private step = 0;
  private events: LogEvent[] = [];
  private startTime: number;
  private logDir: string;
  private currentFile: string;

  constructor(public testId: string, logDir = 'tests/experience/logs') {
    this.logDir = logDir;
    this.currentFile = join(logDir, 'current.jsonl');
    this.startTime = Date.now();
    mkdirSync(logDir, { recursive: true });
  }

  private emit(phase: LogEvent['phase'], action: string, extra: Record<string, any> = {}): LogEvent {
    this.step++;
    const event: LogEvent = {
      test_id: this.testId,
      ts: new Date().toISOString(),
      phase,
      step: this.step,
      action,
      ...extra,
    };
    this.events.push(event);
    appendFileSync(this.currentFile, JSON.stringify(event) + '\n');
    return event;
  }

  setup(action: string, detail?: Record<string, any>) {
    return this.emit('setup', action, { detail: detail ?? this.envInfo() });
  }

  execution(action: string, extra: Record<string, any> = {}) {
    return this.emit('execution', action, extra);
  }

  assertion(action: string, field: string, expected: any, actual: any) {
    const passed = JSON.stringify(expected) === JSON.stringify(actual);
    return this.emit('assertion', action, { field, expected, actual, passed });
  }

  diagnostic(action: string, data: Record<string, any>) {
    return this.emit('diagnostic', action, { data });
  }

  summary(status: 'pass' | 'fail' | 'error') {
    const totalMs = Date.now() - this.startTime;
    const assertions = this.events.filter(e => e.phase === 'assertion');
    return this.emit('summary', 'final', {
      status,
      total_duration_ms: totalMs,
      assertions_passed: assertions.filter(a => a.passed).length,
      assertions_failed: assertions.filter(a => !a.passed).length,
    });
  }

  collectDiagnostics(error?: Error) {
    this.diagnostic('env_snapshot', {
      memory_free_mb: Math.round(os.freemem() / (1024 * 1024)),
      platform: os.platform(),
      uptime_s: os.uptime(),
    });
    if (error) {
      this.diagnostic('error_detail', {
        error_type: error.constructor.name,
        message: error.message,
        stack: error.stack,
      });
    }
  }

  saveHistory() {
    const historyDir = join(this.logDir, 'history');
    mkdirSync(historyDir, { recursive: true });
    const ts = new Date().toISOString().replace(/[:.]/g, '-');
    const dest = join(historyDir, `${ts}_${this.testId}.jsonl`);
    writeFileSync(dest, this.events.map(e => JSON.stringify(e)).join('\n') + '\n');
  }

  private envInfo() {
    return {
      runtime: `node${process.version}`,
      os: os.platform(),
      arch: os.arch(),
    };
  }
}

export function experienceTest(testId: string) {
  const logger = new ExperienceLogger(testId);
  logger.setup('init_env');
  return {
    logger,
    async run(fn: (log: ExperienceLogger) => Promise<void>) {
      try {
        await fn(logger);
        logger.summary('pass');
      } catch (e) {
        logger.collectDiagnostics(e instanceof Error ? e : new Error(String(e)));
        logger.summary('fail');
        throw e;
      } finally {
        logger.saveHistory();
      }
    },
  };
}
```

### Usage in Vitest

```typescript
import { test } from 'vitest';
import { experienceTest } from './experience-logger';

test('create resource', async () => {
  const { run } = experienceTest('atomic_001_create_resource');
  await run(async (log) => {
    log.execution('http_request', { input: { method: 'POST', url: '/api/resources', body: { name: 'test' } } });

    const res = await fetch('/api/resources', { method: 'POST', body: JSON.stringify({ name: 'test' }) });
    const body = await res.json();

    log.execution('http_response', { output: { status: res.status, body, duration_ms: performance.now() } });
    log.assertion('assert_eq', 'status', 201, res.status);
    expect(res.status).toBe(201);
  });
});
```

## Shell Script Logger

```bash
#!/bin/bash
# experience_logger.sh — source this in shell-based experience tests

EXPERIENCE_LOG_DIR="${EXPERIENCE_LOG_DIR:-tests/experience/logs}"
EXPERIENCE_CURRENT="${EXPERIENCE_LOG_DIR}/current.jsonl"
_EXP_STEP=0
_EXP_TEST_ID=""
_EXP_START_TIME=""

exp_init() {
  _EXP_TEST_ID="$1"
  _EXP_START_TIME=$(date +%s%3N)
  mkdir -p "${EXPERIENCE_LOG_DIR}/history"
  _exp_emit "setup" "init_env" "{\"runtime\":\"bash${BASH_VERSION}\",\"os\":\"$(uname -s | tr '[:upper:]' '[:lower:]')\"}"
}

_exp_emit() {
  _EXP_STEP=$((_EXP_STEP + 1))
  local phase="$1" action="$2" extra="$3"
  local ts=$(date -u +%Y-%m-%dT%H:%M:%S.%3NZ)
  local event="{\"test_id\":\"${_EXP_TEST_ID}\",\"ts\":\"${ts}\",\"phase\":\"${phase}\",\"step\":${_EXP_STEP},\"action\":\"${action}\"${extra:+,${extra#\{}}}"
  echo "$event" >> "$EXPERIENCE_CURRENT"
}

exp_exec() { _exp_emit "execution" "$1" "$2"; }
exp_assert() {
  local field="$1" expected="$2" actual="$3"
  local passed="false"
  [ "$expected" = "$actual" ] && passed="true"
  _exp_emit "assertion" "assert_eq" "{\"field\":\"${field}\",\"expected\":\"${expected}\",\"actual\":\"${actual}\",\"passed\":${passed}}"
  [ "$passed" = "true" ]
}

exp_finish() {
  local status="$1"
  local end_time=$(date +%s%3N)
  local duration=$(( end_time - _EXP_START_TIME ))
  _exp_emit "summary" "final" "{\"status\":\"${status}\",\"total_duration_ms\":${duration}}"
  local ts=$(date +%Y-%m-%dT%H-%M-%S)
  cp "$EXPERIENCE_CURRENT" "${EXPERIENCE_LOG_DIR}/history/${ts}_${_EXP_TEST_ID}.jsonl"
}
```
