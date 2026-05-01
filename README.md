# AI Agent Security

A practical, vendor-neutral knowledge base on security threats and defenses for teams building with AI agents and LLM-powered workflows.

## Mission

This repository exists to give developers and technical founders concrete, example-driven guidance on securing AI agent systems. Every article covers real attack scenarios, actionable mitigations, and code or config examples you can apply today. No marketing. No vague advice. Just engineering-grade security content.

## Table of Contents

| # | Topic | File |
|---|-------|------|
| 1 | Prompt Injection Attacks on AI Agents | [docs/01-prompt-injection.md](docs/01-prompt-injection.md) |
| 2 | Data Leakage via Agents | [docs/02-data-leakage-via-agents.md](docs/02-data-leakage-via-agents.md) |
| 3 | Broken Access Control in Multi-Agent Systems | [docs/03-broken-access-control.md](docs/03-broken-access-control.md) |
| 4 | Supply Chain Risks in LLM Toolchains | [docs/04-supply-chain-risks.md](docs/04-supply-chain-risks.md) |
| 5 | Trust Boundaries in Multi-Agent Architectures | [docs/05-agent-to-agent-trust.md](docs/05-agent-to-agent-trust.md) |
| 6 | Monitoring and Anomaly Detection for Agents | [docs/06-monitoring-and-anomaly-detection.md](docs/06-monitoring-and-anomaly-detection.md) |
| 7 | Safe Deployment Patterns for AI Agents | [docs/07-safe-deployment-patterns.md](docs/07-safe-deployment-patterns.md) |

## Who This Is For

- **Developers** integrating LLMs and AI agents into production systems
- **Technical founders** shipping agent-powered products who need a security baseline
- **Security engineers** evaluating the threat surface of agentic AI systems
- **Platform teams** building internal tooling around LLM APIs and agent orchestrators

Prior knowledge of LLMs and basic application security concepts is assumed.

## How to Contribute

We accept pull requests for:

- **New threat articles** covering agent security topics not yet in the repo
- **Corrections** to existing content (factual errors, outdated mitigations)
- **Real-world examples** of agent security failures (anonymized where necessary)
- **Code examples** demonstrating mitigations in additional languages or frameworks

To contribute:

1. Fork the repository
2. Create a branch: `git checkout -b topic/your-article-name`
3. Follow the existing article structure: threat summary box, real scenario, numbered mitigations, further reading
4. Submit a PR with a clear description of what the article covers and why it matters

Please keep the tone direct and technical. No marketing language.

## License

This project is licensed under the [MIT License](LICENSE).
