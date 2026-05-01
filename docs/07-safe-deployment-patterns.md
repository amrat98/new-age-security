# Safe Deployment Patterns for AI Agents

> **Threat summary:** Deploying AI agents to production without safety patterns is equivalent to giving an unpredictable autonomous process broad access to your systems with no guardrails. Agents need deployment patterns that account for their non-deterministic nature: least privilege, isolation, secrets hygiene, human gates, and gradual rollout.

## The Core Difference From Traditional Deployments

Traditional services execute deterministic code paths. You can test every branch. AI agents are non-deterministic: the same input may produce different actions on different runs. This means:

- You cannot fully test agent behaviour in staging
- Prompt changes are functionally code changes but are often treated casually
- An agent that behaved safely yesterday may not behave safely today if its context changes

Deployment patterns must account for this uncertainty.

## Real Scenario

A team deploys an internal operations agent that can:
- Query databases for reports
- Send Slack messages to team channels
- Create Jira tickets
- Execute SQL for data corrections

On day one, the agent works correctly. On day 14, a user asks it to "clean up the duplicate records in the orders table." The agent interprets this as a DELETE operation, constructs a SQL query, and executes it. 847 production orders are deleted. There was no approval gate, no dry-run mode, and no way to distinguish this from a normal read query at the deployment level.

## Mitigations

### 1. Principle of least privilege

Every agent gets the minimum permissions required for its task. Default deny, explicit allow:

```python
# Agent capability manifest - declare at deployment time
AGENT_CAPABILITIES = {
    "report-agent": {
        "database": {
            "allowed_operations": ["SELECT"],  # no INSERT, UPDATE, DELETE
            "allowed_tables": ["orders_view", "customers_view"],  # views, not base tables
            "max_rows_returned": 1000,
        },
        "slack": {
            "allowed_channels": ["#reports", "#ops-team"],
            "can_dm": False,
        },
        "filesystem": None,  # no filesystem access
    }
}

def enforce_capabilities(agent_id: str, action: dict) -> bool:
    caps = AGENT_CAPABILITIES[agent_id]
    resource = action["resource_type"]

    if resource not in caps or caps[resource] is None:
        raise PermissionError(f"{agent_id} has no access to {resource}")

    if resource == "database":
        if action["operation"] not in caps[resource]["allowed_operations"]:
            raise PermissionError(
                f"{agent_id} cannot execute {action['operation']}"
            )
    return True
```

### 2. Environment isolation

Run agents in isolated environments with no shared state with production services:

```yaml
# docker-compose.yml for agent deployment
services:
  agent:
    image: company/ops-agent:v1.2.3
    read_only: true
    security_opt:
      - no-new-privileges:true
    cap_drop:
      - ALL
    networks:
      - agent-internal
    deploy:
      resources:
        limits:
          memory: 512M
          cpus: "0.5"
    tmpfs:
      - /tmp:size=50M

  # Agent communicates with production only through a gateway
  agent-gateway:
    image: company/agent-gateway:v1.0.0
    networks:
      - agent-internal
      - production
    environment:
      - ALLOWED_ENDPOINTS=db-readonly.internal,slack-api.internal

networks:
  agent-internal:
    internal: true  # no external access
  production:
    external: true
```

### 3. Secrets management

Never hardcode keys. Never pass secrets in environment variables that the agent's LLM calls can access:

```python
import os
from functools import lru_cache

class AgentSecretsManager:
    """Secrets are fetched at tool execution time, never included in prompts."""

    def __init__(self, vault_addr: str):
        self.vault_addr = vault_addr
        # Auth token is process-level, never exposed to the LLM
        self._auth_token = os.environ["VAULT_TOKEN"]

    def get_secret(self, path: str) -> str:
        """Fetch secret from vault. Called by tool implementations, not by the agent."""
        import hvac
        client = hvac.Client(url=self.vault_addr, token=self._auth_token)
        secret = client.secrets.kv.v2.read_secret_version(path=path)
        return secret["data"]["data"]["value"]

# Tool implementation uses secrets manager directly
# The LLM never sees the API key
def send_slack_message(channel: str, message: str):
    secrets = AgentSecretsManager(os.environ["VAULT_ADDR"])
    token = secrets.get_secret("slack/bot-token")
    # Use token to call Slack API
    ...
```

Key principle: the LLM decides *what* to do. The tool implementation handles *how*, including authentication. The model never sees credentials.

### 4. Human approval gates for destructive actions

Classify actions by risk and require human confirmation for anything irreversible:

```python
DESTRUCTIVE_ACTIONS = {
    "database.DELETE",
    "database.UPDATE",
    "database.DROP",
    "email.send_external",
    "jira.delete_ticket",
    "slack.send_to_public_channel",
    "filesystem.delete",
}

async def gated_execution(action: str, params: dict, approval_channel):
    if action in DESTRUCTIVE_ACTIONS:
        # Show the human exactly what will happen
        preview = generate_action_preview(action, params)
        approved = await approval_channel.request(
            title=f"Agent requests: {action}",
            details=preview,
            timeout_seconds=300,  # auto-deny after 5 minutes
        )
        if not approved:
            return {"status": "denied", "action": action}

    return execute_action(action, params)
```

For SQL specifically, use dry-run mode:

```python
def safe_sql_execution(query: str, connection):
    # Wrap in transaction, show results, require approval before commit
    connection.execute("BEGIN")
    result = connection.execute(query)
    rows_affected = result.rowcount

    if rows_affected > 10:  # threshold for bulk operations
        connection.execute("ROLLBACK")
        raise ApprovalRequired(
            f"Query would affect {rows_affected} rows. Requires human approval."
        )

    connection.execute("COMMIT")
    return result
```

### 5. Canary deployments for prompt changes

Treat prompt changes like code deployments. Roll them out gradually:

```yaml
# Agent prompt deployment config
prompt_deployment:
  strategy: canary
  stages:
    - name: canary
      traffic_percent: 5
      duration: 1h
      success_criteria:
        error_rate: "<2%"
        anomaly_alerts: 0
        avg_task_duration: "<30s"
    - name: partial
      traffic_percent: 25
      duration: 4h
      success_criteria:
        error_rate: "<2%"
        anomaly_alerts: 0
    - name: full
      traffic_percent: 100

  rollback:
    automatic: true
    triggers:
      - error_rate > 5%
      - any critical anomaly alert
      - task_duration_p95 > 60s
```

Implementation with feature flags:

```python
import hashlib

def get_prompt_version(task_id: str, canary_percent: int = 5) -> str:
    # Deterministic assignment based on task_id
    bucket = int(hashlib.md5(task_id.encode()).hexdigest(), 16) % 100
    if bucket < canary_percent:
        return "v2-canary"  # new prompt
    return "v1-stable"      # current prompt

def load_system_prompt(version: str) -> str:
    prompts = {
        "v1-stable": "prompts/system_v1.txt",
        "v2-canary": "prompts/system_v2.txt",
    }
    with open(prompts[version]) as f:
        return f.read()
```

## Key Takeaways

- Agents are non-deterministic. Deploy them with the same caution you would give a new hire with system access: limited permissions, supervised actions, gradual trust building.
- Secrets never flow through the LLM. Tool implementations fetch credentials independently.
- Destructive actions always require human approval. Auto-deny if no response within timeout.
- Prompt changes are production deployments. Canary them.

## Further Reading

- [OWASP Top 10 for LLM Applications: LLM08 Excessive Agency](https://owasp.org/www-project-top-10-for-large-language-model-applications/)
- [NIST SP 800-53: Security and Privacy Controls (AC-6 Least Privilege)](https://csrc.nist.gov/publications/detail/sp/800-53/rev-5/final)
- [Google SAIF: Secure AI Framework](https://safety.google/cybersecurity-advancements/saif/)
