# Data Leakage via AI Agents

> **Threat summary:** AI agents routinely pass sensitive data to LLM APIs, logging systems, and third-party tools during normal operation. Without explicit data minimization controls, agents become unintentional exfiltration vectors for PII, credentials, and proprietary business data.

## The Exfiltration Surface

Every time an agent constructs a prompt, it potentially sends data to an external system. The exfiltration surface includes:

| Vector | What leaks | Who sees it |
|--------|-----------|-------------|
| **LLM API calls** | Full prompt text including context, user data, and retrieved documents | The LLM provider, their logging infrastructure, and potentially their training pipeline |
| **Agent logs** | Complete prompt/response pairs, tool call arguments, intermediate reasoning | Anyone with log access: devs, SRE, SIEM systems, log aggregation vendors |
| **Third-party tool calls** | Data passed as function arguments to plugins, MCP servers, and integrations | The tool provider and their downstream dependencies |
| **Error reporting** | Stack traces that include variable contents from prompt construction | Error tracking services (Sentry, Datadog, etc.) |

The core problem: agents are designed to be helpful, which means they aggressively gather context. Without guardrails, "gather context" becomes "exfiltrate everything accessible."

## Real Scenario

A sales team deploys an AI agent that drafts follow-up emails. The agent:

1. Queries the CRM for the contact's deal history, revenue figures, and internal notes
2. Constructs a prompt that includes: contact name, email, company revenue, deal size, internal notes like "this client is price-sensitive, willing to offer 40% discount"
3. Sends this prompt to a third-party LLM API

The LLM provider logs all API requests for abuse monitoring. A breach of the provider's logging infrastructure exposes:
- Customer PII (names, emails)
- Proprietary pricing strategy
- Internal negotiation notes

The sales team never intended to share this data externally. The agent did it as part of normal operation.

## The Data Minimization Principle for Agents

The fix is straightforward in principle: **send only the data the model needs to complete the specific task, and nothing more.**

In practice, this requires:

1. Defining what data fields are needed per task type
2. Scrubbing PII and sensitive data before prompt construction
3. Using references (IDs) instead of raw values where possible
4. Auditing what actually gets sent to the LLM API

## Mitigations

### 1. PII scrubbing before prompt construction

Intercept data between your data source and the prompt builder. Replace sensitive values with placeholders, then re-hydrate in the final output if needed.

```python
import re
from typing import Dict, Tuple

# Patterns for common PII types
PII_PATTERNS = {
    "email": r"[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}",
    "phone": r"\b\d{3}[-.]?\d{3}[-.]?\d{4}\b",
    "ssn": r"\b\d{3}-\d{2}-\d{4}\b",
    "credit_card": r"\b\d{4}[\s-]?\d{4}[\s-]?\d{4}[\s-]?\d{4}\b",
}


def scrub_pii(text: str) -> Tuple[str, Dict[str, str]]:
    """Remove PII from text. Returns scrubbed text and a mapping for re-hydration."""
    mapping = {}
    counter = 0

    for pii_type, pattern in PII_PATTERNS.items():
        for match in re.finditer(pattern, text):
            placeholder = f"[{pii_type.upper()}_{counter}]"
            mapping[placeholder] = match.group()
            text = text.replace(match.group(), placeholder, 1)
            counter += 1

    return text, mapping


def rehydrate(text: str, mapping: Dict[str, str]) -> str:
    """Restore original values from placeholders in agent output."""
    for placeholder, original in mapping.items():
        text = text.replace(placeholder, original)
    return text


# Usage
crm_notes = """
Follow up with John Smith (john.smith@acme.corp, 555-867-5309).
Deal size: $2.4M. Internal note: willing to offer 40% discount.
"""

scrubbed, pii_map = scrub_pii(crm_notes)
# scrubbed: "Follow up with John Smith ([EMAIL_0], [PHONE_1])..."
# pii_map: {"[EMAIL_0]": "john.smith@acme.corp", "[PHONE_1]": "555-867-5309"}

# Only `scrubbed` goes to the LLM API
# After getting the response, rehydrate if sending to the user
```

### 2. Field-level access control on prompt context

Define explicitly which CRM fields the agent can include in prompts:

```python
ALLOWED_FIELDS_FOR_EMAIL_DRAFT = {"first_name", "company_name", "product_interest"}

def build_prompt_context(contact: dict, task: str) -> dict:
    """Filter contact data to only fields allowed for this task type."""
    allowed = ALLOWED_FIELDS_FOR_EMAIL_DRAFT  # look up by task type
    filtered = {k: v for k, v in contact.items() if k in allowed}
    return filtered

# Full CRM record has 30+ fields; only 3 make it into the prompt
```

### 3. Audit logging of outbound data

Log what gets sent to the LLM API so you can detect drift:

```python
import hashlib
import json
import logging

logger = logging.getLogger("agent.audit")

def send_to_llm(prompt: str, model: str = "gpt-4") -> str:
    # Log a hash for auditing without storing the full prompt in plaintext
    prompt_hash = hashlib.sha256(prompt.encode()).hexdigest()
    prompt_size = len(prompt)

    logger.info(
        "llm_call",
        extra={
            "model": model,
            "prompt_hash": prompt_hash,
            "prompt_bytes": prompt_size,
            "contains_pii": bool(re.search(r"|".join(PII_PATTERNS.values()), prompt)),
        },
    )

    if prompt_size > 4000:
        logger.warning("Large prompt detected; review for data over-inclusion")

    # Actual API call here
    ...
```

### 4. Disable LLM provider logging where possible

Most API providers offer zero-data-retention options. Use them:

```bash
# OpenAI: opt out of data usage for training
# Set in API request headers or organization settings
curl https://api.openai.com/v1/chat/completions \
  -H "Authorization: Bearer $OPENAI_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "gpt-4",
    "messages": [{"role": "user", "content": "..."}]
  }'
# Ensure your org has "Data usage for improving model performance" disabled
```

## Key Takeaways

- Every prompt sent to an LLM API is a potential data leak. Treat prompt construction as a data export operation.
- Scrub PII before building the prompt, not after.
- Use field-level allowlists to prevent context over-fetching.
- Audit outbound prompt size and content as part of your security monitoring.

## Further Reading

- [OWASP Top 10 for LLM Applications: LLM06 Sensitive Information Disclosure](https://owasp.org/www-project-top-10-for-large-language-model-applications/)
- [NIST Privacy Framework](https://www.nist.gov/privacy-framework)
- [GDPR Article 25: Data Protection by Design and Default](https://gdpr-info.eu/art-25-gdpr/)
