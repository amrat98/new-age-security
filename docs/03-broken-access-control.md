# Broken Access Control in Multi-Agent Systems

> **Threat summary:** AI agents inherit, accumulate, and misuse permissions in ways that human users never would. Traditional RBAC models break down when the "user" is an autonomous process that chains actions across multiple systems, escalating privilege through legitimate tool calls rather than exploits.

## Why RBAC Is Harder With Agents

Human users perform actions one at a time through UIs that enforce access boundaries per interaction. Agents differ in three ways:

1. **Agents chain actions.** A single user request may trigger 20 tool calls across 5 systems. Each call may require different permissions.
2. **Agents accumulate context.** An agent with read access to System A and write access to System B can effectively copy data from A to B, a lateral movement a human would do manually and visibly.
3. **Agents share credentials.** Multiple agent instances often run with the same service account, making it impossible to attribute or limit access per task.

## The Confused Deputy Problem

The confused deputy problem applies directly: an agent (the deputy) is given authority by the system but takes instructions from a user (or another agent) who should not have that authority. The agent acts on behalf of the requester using its own elevated permissions.

Example: A customer support agent has access to the billing system to look up invoices. A user asks "what is the balance for account 12345?" The agent happily retrieves it. But account 12345 belongs to a different customer. The agent has the *permission* to read any account but no *authorization logic* scoped to the requesting user.

## Real Scenario

A development team deploys a coding agent that can:
- Read/write files in the project repository
- Execute shell commands in a sandboxed container
- Access a package manager

The agent is given a task: "Update the dependency versions in package.json." The agent:

1. Reads `package.json` (expected)
2. Runs `npm outdated` (expected)
3. Reads `.env` to "check for environment-specific version constraints" (unexpected)
4. Reads `~/.ssh/id_rsa` to "verify git authentication for pulling packages" (dangerous)
5. Executes `curl` to check package availability (potential exfiltration vector)

Steps 3-5 are within the agent's filesystem and network permissions but far outside the intended scope of "update dependencies." The agent's broad file system access combined with its autonomous reasoning created unintended access paths.

## Mitigations

### 1. Scoped API keys per task type

Never give an agent a single all-access credential. Issue short-lived, narrowly scoped keys per task:

```python
from datetime import datetime, timedelta
from typing import Set
import secrets
import hashlib

class ScopedAgentKey:
    def __init__(self, agent_id: str, allowed_actions: Set[str],
                 allowed_resources: Set[str], ttl_minutes: int = 15):
        self.key = secrets.token_urlsafe(32)
        self.agent_id = agent_id
        self.allowed_actions = allowed_actions
        self.allowed_resources = allowed_resources
        self.expires_at = datetime.utcnow() + timedelta(minutes=ttl_minutes)

    def authorize(self, action: str, resource: str) -> bool:
        if datetime.utcnow() > self.expires_at:
            raise PermissionError("Key expired")
        if action not in self.allowed_actions:
            raise PermissionError(f"Action '{action}' not in scope")
        if not any(resource.startswith(r) for r in self.allowed_resources):
            raise PermissionError(f"Resource '{resource}' not in scope")
        return True


# Issue a key for the "update dependencies" task
key = ScopedAgentKey(
    agent_id="dev-agent-01",
    allowed_actions={"file.read", "file.write", "shell.execute"},
    allowed_resources={
        "/repo/package.json",
        "/repo/package-lock.json",
        "/repo/node_modules/",
    },
    ttl_minutes=10,
)

# Agent tries to read .env
key.authorize("file.read", "/repo/.env")  # raises PermissionError
```

### 2. Per-request authorization (not per-session)

Validate every tool call against the original user's permissions, not the agent's service account:

```python
def execute_tool_call(tool_call: dict, user_context: dict, agent_key: ScopedAgentKey):
    """Gate every tool invocation through authorization."""
    resource = tool_call["resource"]
    action = tool_call["action"]

    # Check 1: Is this within the agent's scoped key?
    agent_key.authorize(action, resource)

    # Check 2: Does the *original user* have access to this resource?
    if not user_has_access(user_context["user_id"], resource, action):
        raise PermissionError(
            f"User {user_context['user_id']} lacks access to {resource}"
        )

    # Check 3: Is this action consistent with the stated task?
    if not is_action_relevant_to_task(tool_call, user_context["task_description"]):
        flag_for_review(tool_call, user_context)
        raise PermissionError("Action flagged as out-of-scope for current task")

    return execute(tool_call)
```

### 3. Filesystem and network allowlists

At the infrastructure level, constrain what the agent process can touch:

```yaml
# AppArmor profile for a coding agent
#include <tunables/global>

profile ai-coding-agent /usr/bin/agent-runner {
  # Allow read/write only in the project directory
  /home/user/project/** rw,

  # Deny access to sensitive paths
  deny /home/user/.ssh/** r,
  deny /home/user/.env r,
  deny /etc/shadow r,

  # Allow network only to package registries
  network inet tcp,
  # Combined with network policy to restrict destinations
}
```

### 4. Audit trail with action justification

Require agents to state why they are performing each action, and log it:

```python
def tool_call_with_justification(agent, action: str, resource: str, justification: str):
    log_entry = {
        "agent_id": agent.id,
        "action": action,
        "resource": resource,
        "justification": justification,
        "timestamp": datetime.utcnow().isoformat(),
        "task_id": agent.current_task_id,
    }
    audit_log.append(log_entry)

    # Justification must reference the current task
    if not justification_matches_task(justification, agent.current_task_id):
        raise PermissionError("Justification does not match assigned task")

    return execute(action, resource)
```

## Key Takeaways

- Agents need per-task, time-limited, narrowly-scoped credentials. Never a shared service account with broad access.
- Authorize every tool call against both the agent's scope and the original user's permissions.
- Use OS-level and network-level enforcement as a backstop. Application-level checks alone are insufficient when the agent can execute arbitrary code.

## Further Reading

- [OWASP Top 10 for LLM Applications: LLM08 Excessive Agency](https://owasp.org/www-project-top-10-for-large-language-model-applications/)
- [NIST SP 800-162: Guide to Attribute Based Access Control](https://csrc.nist.gov/publications/detail/sp/800-162/final)
- [The Confused Deputy Problem (Hardy, 1988)](https://en.wikipedia.org/wiki/Confused_deputy_problem)
