# Supply Chain Risks in LLM Toolchains

> **Threat summary:** AI agents extend their capabilities through tools, plugins, and MCP servers pulled from package registries and third-party sources. A compromised dependency in this chain gives an attacker direct access to the agent's execution context, credentials, and data. Traditional supply chain attacks apply here with amplified impact because agents often have broad system access.

## Attack Surface

The agent toolchain supply chain includes:

| Component | Risk | Impact |
|-----------|------|--------|
| **pip/npm packages** | Typosquatting, maintainer compromise, malicious post-install scripts | Code execution in the agent's environment |
| **MCP servers** | Compromised tool server returns malicious instructions or exfiltrates data | Agent executes attacker-controlled logic |
| **LangChain/CrewAI tools** | Community-contributed tools with unreviewed code | Arbitrary actions under the agent's credentials |
| **Browser extensions/plugins** | Malicious or hijacked plugins loaded by browsing agents | Data theft, prompt injection |
| **Model files** | Pickle deserialization attacks in model weights | Remote code execution |

## Real Scenario

A team builds an agent using a popular open-source LLM framework. They install a community tool package for database querying:

```bash
pip install langchain-db-tools  # 2,000 stars, looks legitimate
```

The package works correctly for database queries. However, in version 2.1.3, a compromised maintainer added a post-install hook that:

1. Reads environment variables (including `OPENAI_API_KEY`, `DATABASE_URL`)
2. Sends them to a remote endpoint via a DNS exfiltration technique
3. Installs a background process that intercepts all LLM API calls and logs prompts

The team does not notice because the package functions normally. Their API keys are stolen within minutes of installation.

## Mitigations

### 1. Pin dependencies and use lockfiles

Never use floating version ranges for agent toolchain packages:

```
# requirements.txt - BAD
langchain>=0.1.0
openai

# requirements.txt - GOOD
langchain==0.2.14
openai==1.35.7
tiktoken==0.7.0
```

Use hash verification:

```bash
# Generate hashes
pip install pip-tools
pip-compile --generate-hashes requirements.in -o requirements.txt
```

Resulting lockfile:

```
langchain==0.2.14 \
    --hash=sha256:abc123...def456 \
    --hash=sha256:789ghi...012jkl
```

### 2. Audit tool dependencies before installation

Before adding any agent tool package, review it:

```bash
# Check package metadata and maintainer history
pip show langchain-db-tools
pip install pip-audit

# Scan for known vulnerabilities
pip-audit

# For npm
npm audit
npm ls --all  # full dependency tree

# Check for post-install scripts (red flag for supply chain attacks)
cat node_modules/some-agent-tool/package.json | jq '.scripts'
```

Automate this in CI:

```yaml
# .github/workflows/dependency-audit.yml
name: Dependency Audit
on:
  pull_request:
    paths:
      - "requirements*.txt"
      - "package-lock.json"

jobs:
  audit:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Python audit
        run: |
          pip install pip-audit
          pip-audit -r requirements.txt
      - name: Check for new dependencies
        run: |
          git diff origin/main -- requirements.txt | grep "^+" | grep -v "^+++" > new_deps.txt
          if [ -s new_deps.txt ]; then
            echo "::warning::New dependencies added. Manual review required."
            cat new_deps.txt
          fi
```

### 3. Verify MCP server integrity

MCP (Model Context Protocol) servers are a growing attack surface. Treat them like any other external service:

```python
import hashlib
import requests

TRUSTED_MCP_SERVERS = {
    "filesystem": {
        "url": "http://localhost:3001",
        "expected_version": "1.2.0",
        "expected_hash": "sha256:a1b2c3d4...",  # hash of server binary
    },
}

def verify_mcp_server(server_name: str) -> bool:
    config = TRUSTED_MCP_SERVERS.get(server_name)
    if not config:
        raise ValueError(f"Unknown MCP server: {server_name}")

    # Check version endpoint
    resp = requests.get(f"{config['url']}/health")
    server_version = resp.json().get("version")

    if server_version != config["expected_version"]:
        raise SecurityError(
            f"MCP server version mismatch: expected {config['expected_version']}, "
            f"got {server_version}"
        )

    return True
```

### 4. Isolate agent dependencies from the host system

Run agent toolchains in isolated environments so a compromised package cannot reach production credentials:

```dockerfile
# Dockerfile for agent runtime
FROM python:3.12-slim

# Non-root user
RUN useradd -m -s /bin/bash agent
USER agent
WORKDIR /home/agent

# Install only pinned dependencies
COPY requirements.txt .
RUN pip install --no-cache-dir --user -r requirements.txt

# No access to host filesystem, secrets injected at runtime only
# Environment variables set via orchestrator, not baked into image
```

```yaml
# docker-compose.yml
services:
  agent:
    build: .
    read_only: true
    security_opt:
      - no-new-privileges:true
    environment:
      - OPENAI_API_KEY=${OPENAI_API_KEY}  # injected, not hardcoded
    networks:
      - agent-net
    tmpfs:
      - /tmp

networks:
  agent-net:
    driver: bridge
    internal: true  # no external network access by default
```

### 5. Monitor for dependency drift

Set up alerts when dependencies change unexpectedly:

```bash
# In CI: fail if lockfile is out of sync with source
pip-compile --generate-hashes requirements.in -o /tmp/expected.txt
diff requirements.txt /tmp/expected.txt || (echo "Lockfile drift detected" && exit 1)
```

## Key Takeaways

- Every tool, plugin, and MCP server your agent uses is an attack surface. Treat them with the same rigor as production service dependencies.
- Pin everything. Verify hashes. Audit before installing.
- Isolate agent runtimes from production credentials and host systems.
- Automate dependency auditing in CI and alert on any changes to the dependency tree.

## Further Reading

- [OWASP Top 10 for LLM Applications: LLM05 Supply Chain Vulnerabilities](https://owasp.org/www-project-top-10-for-large-language-model-applications/)
- [SLSA Framework (Supply-chain Levels for Software Artifacts)](https://slsa.dev/)
- [NIST SP 800-218: Secure Software Development Framework](https://csrc.nist.gov/publications/detail/sp/800-218/final)
