# Monitoring and Anomaly Detection for AI Agents

> **Threat summary:** AI agents can be compromised, hallucinate dangerous actions, or drift from their intended behaviour without any visible error. Traditional application monitoring (HTTP status codes, latency, error rates) is insufficient. You need agent-specific observability that tracks what the agent *does*, not just whether it is running.

## What a Baseline Looks Like

Before you can detect anomalies, you need to know what normal looks like. For an AI agent, the baseline includes:

| Metric | Normal range (example) | Anomaly signal |
|--------|----------------------|----------------|
| Tokens per request | 500-2000 | Sudden spike to 10,000+ (agent dumping context or system prompt) |
| Tool calls per task | 3-8 | 25+ calls (agent looping or being manipulated) |
| Unique API endpoints called | 2-4 per task type | New endpoint never seen before |
| Task completion time | 5-30 seconds | 5+ minutes (agent stuck or performing unintended actions) |
| Output length | 100-500 tokens | 5000+ tokens (possible data dump) |
| Error rate | <5% | Spike above 20% |

Establish these baselines per agent type and task type over a 2-week observation period before setting alert thresholds.

## Real Scenario

A customer support agent normally:
- Reads 1-2 knowledge base articles per query
- Makes 1 CRM lookup
- Generates a 200-token response
- Completes in 8 seconds

After a prompt injection attack via a customer message, the agent:
- Makes 15 knowledge base lookups (searching for sensitive internal docs)
- Attempts 3 CRM lookups for accounts unrelated to the conversation
- Generates a 4,000-token response (containing internal pricing data)
- Takes 45 seconds to complete

Without behavioral monitoring, this looks like a normal (if slow) interaction. With baseline-aware monitoring, every metric is 3-5x outside normal range and triggers an alert.

## Mitigations

### 1. Structured logging for every agent action

Log every tool call, LLM invocation, and decision point in a structured format:

```python
import structlog
import time
from contextlib import contextmanager

logger = structlog.get_logger()

@contextmanager
def agent_action_span(agent_id: str, task_id: str, action: str):
    start = time.time()
    log = logger.bind(agent_id=agent_id, task_id=task_id, action=action)
    log.info("action_started")
    try:
        yield log
    except Exception as e:
        log.error("action_failed", error=str(e), duration=time.time() - start)
        raise
    else:
        log.info("action_completed", duration=time.time() - start)

# Usage in agent code
with agent_action_span("support-agent-01", "task-abc123", "crm_lookup") as log:
    result = crm.lookup(account_id=account_id)
    log.info("crm_result", records_returned=len(result))
```

### 2. Token volume and API call pattern monitoring

Track cumulative token usage and API calls per task:

```python
class AgentMetrics:
    def __init__(self, agent_id: str, task_id: str):
        self.agent_id = agent_id
        self.task_id = task_id
        self.total_input_tokens = 0
        self.total_output_tokens = 0
        self.tool_calls = []
        self.api_endpoints_called = set()

    def record_llm_call(self, input_tokens: int, output_tokens: int):
        self.total_input_tokens += input_tokens
        self.total_output_tokens += output_tokens

    def record_tool_call(self, tool_name: str, endpoint: str = None):
        self.tool_calls.append(tool_name)
        if endpoint:
            self.api_endpoints_called.add(endpoint)

    def check_anomalies(self, baselines: dict) -> list:
        alerts = []
        if self.total_output_tokens > baselines["max_output_tokens"]:
            alerts.append(f"Output tokens {self.total_output_tokens} exceeds baseline")
        if len(self.tool_calls) > baselines["max_tool_calls"]:
            alerts.append(f"Tool calls {len(self.tool_calls)} exceeds baseline")
        new_endpoints = self.api_endpoints_called - baselines["known_endpoints"]
        if new_endpoints:
            alerts.append(f"Unknown endpoints called: {new_endpoints}")
        return alerts
```

### 3. Alerting configuration

Define thresholds and routing in your monitoring system. Example configuration for a Prometheus + Alertmanager style setup:

```yaml
# agent-alerts.yml
groups:
  - name: agent_anomalies
    rules:
      - alert: AgentHighTokenOutput
        expr: agent_output_tokens_total{task_type="support"} > 3000
        for: 0m
        labels:
          severity: warning
        annotations:
          summary: "Agent {{ $labels.agent_id }} produced excessive output tokens"
          description: "Output tokens: {{ $value }}. Baseline max: 3000. Possible data exfiltration."

      - alert: AgentExcessiveToolCalls
        expr: agent_tool_calls_total > 15
        for: 0m
        labels:
          severity: critical
        annotations:
          summary: "Agent {{ $labels.agent_id }} making excessive tool calls"
          description: "Tool calls: {{ $value }}. Normal range: 3-8. Agent may be compromised or looping."

      - alert: AgentUnknownEndpoint
        expr: agent_unknown_endpoint_calls > 0
        for: 0m
        labels:
          severity: critical
        annotations:
          summary: "Agent {{ $labels.agent_id }} called an unknown API endpoint"
          description: "Endpoint: {{ $labels.endpoint }}. This was not in the approved endpoint list."

      - alert: AgentSlowTask
        expr: agent_task_duration_seconds > 120
        for: 0m
        labels:
          severity: warning
        annotations:
          summary: "Agent {{ $labels.agent_id }} task running unusually long"
          description: "Duration: {{ $value }}s. Normal range: 5-30s."

      - alert: AgentHighErrorRate
        expr: rate(agent_errors_total[5m]) / rate(agent_tasks_total[5m]) > 0.2
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Agent error rate above 20%"
```

### 4. Real-time action stream with kill switch

Implement a mechanism to halt an agent immediately when anomalies are detected:

```python
import asyncio
from enum import Enum

class AgentStatus(Enum):
    RUNNING = "running"
    PAUSED = "paused"
    KILLED = "killed"

class AgentSupervisor:
    def __init__(self):
        self.agent_status = {}

    def kill(self, agent_id: str, reason: str):
        self.agent_status[agent_id] = AgentStatus.KILLED
        logger.critical("agent_killed", agent_id=agent_id, reason=reason)
        # Revoke agent's API keys immediately
        revoke_agent_credentials(agent_id)

    def check_before_action(self, agent_id: str):
        status = self.agent_status.get(agent_id, AgentStatus.RUNNING)
        if status == AgentStatus.KILLED:
            raise SystemExit(f"Agent {agent_id} has been terminated")
        if status == AgentStatus.PAUSED:
            raise PausedError(f"Agent {agent_id} is paused pending review")

supervisor = AgentSupervisor()

# In alert handler:
# supervisor.kill("support-agent-01", "Unknown endpoint called")
```

### 5. Periodic baseline recalculation

Baselines drift as agents are updated. Recalculate weekly:

```bash
# Cron job: recalculate baselines from last 14 days of metrics
0 2 * * 1 /usr/local/bin/recalculate-agent-baselines \
  --lookback-days=14 \
  --output=/etc/agent-monitor/baselines.json \
  --percentile=95
```

## Key Takeaways

- Monitor what agents *do* (tool calls, tokens, endpoints), not just whether they are up.
- Establish per-agent, per-task baselines over a 2-week period before alerting.
- Implement a kill switch that revokes credentials immediately on critical anomalies.
- Log every action in structured format. You cannot detect what you do not record.

## Further Reading

- [OWASP Top 10 for LLM Applications: LLM09 Overreliance](https://owasp.org/www-project-top-10-for-large-language-model-applications/)
- [NIST AI 100-1: Artificial Intelligence Risk Management Framework](https://www.nist.gov/artificial-intelligence/executive-order-safe-secure-and-trustworthy-artificial-intelligence)
- [OpenTelemetry Semantic Conventions for GenAI](https://opentelemetry.io/docs/specs/semconv/gen-ai/)
