# Salesforce Agentforce: Architectural Patterns

## Core Concepts

Agentforce is Salesforce's platform for building autonomous AI agents that can take actions, reason across steps, and interact with customers or employees. It runs on the Einstein Trust Layer, which provides security, grounding, and compliance guardrails.

### Key Components

| Component | Purpose |
|-----------|---------|
| Agent | The AI entity — has a role, instructions, and access to actions |
| Topic | A domain of capability assigned to an agent (e.g., "Handle Returns") |
| Instructions | Natural language rules that govern agent behavior within a topic |
| Actions | What the agent can do — Flows, Apex, APIs, Prompt Templates |
| Einstein Trust Layer | Security/compliance wrapper — masking, toxicity, audit trail |
| Reasoning Engine | Atlas Reasoning Engine — plans and executes multi-step tasks |
| Channels | Where agents are deployed (Service Cloud, Experience Cloud, Slack, custom) |

---

## Agent Architecture Patterns

### Pattern 1: Employee-Facing Agent (Copilot)
- Assists internal users (sales reps, service agents) within Salesforce UI.
- Surfaces CRM context, suggests next actions, drafts communications.
- Lower latency tolerance acceptable; user is in the loop.
- Actions: Flow, Apex, SOQL lookups, record updates.

### Pattern 2: Customer-Facing Agent (Autonomous)
- Handles end-customer interactions (chat, messaging, Experience Cloud).
- Must have robust guardrails — topic boundaries, escalation paths, fallback actions.
- Higher stakes: wrong responses reach customers directly.
- Always design an explicit human handoff action.

### Pattern 3: Background / Scheduled "Agent"
- Runs autonomously without real-time human interaction.
- Triggered by events (Platform Events, Data Cloud Data Actions) or schedules.
- Use cases: proactive outreach, data enrichment, case triage.
- Requires careful Action scoping — limit destructive operations.

---

## Topic and Instruction Design

### Topic Design Principles
- **One topic per business capability** — don't bundle unrelated skills.
- Topics should have clear entry criteria: "When a customer asks about order status..."
- Define explicit out-of-scope boundaries: "Do not discuss pricing or promotions."
- Keep topic count manageable — too many topics increases routing ambiguity.

### Instruction Writing
- Use declarative language: "Always verify the customer's identity before accessing account details."
- Specify fallback behavior: "If you cannot find the order, ask the customer for their email address."
- Define escalation triggers: "If the customer expresses frustration more than twice, transfer to a human agent."
- Avoid contradictory instructions within a topic.

---

## Action Design

### Action Types

| Type | Best For | Latency |
|------|----------|---------|
| Flow (Autolaunched) | CRM reads/writes, simple logic | Low |
| Apex | Complex logic, external callouts | Medium |
| Prompt Template | LLM-powered generation, summarization | Medium |
| External API (via named credential) | Third-party system integration | Variable |
| Data Cloud Query | Unified profile lookups | Low–Medium |

### Action Design Rules
- **Inputs:** Clearly define required vs. optional parameters. The agent's LLM infers values from conversation context — be explicit about what is needed.
- **Outputs:** Return structured, agent-readable responses. Avoid raw HTML or unformatted blobs.
- **Idempotency:** Actions that modify data should be idempotent where possible — agents may retry on ambiguous outcomes.
- **Scope:** Limit each action to one logical operation. Compound actions are harder to debug and increase error surface.
- **Error handling:** Return meaningful error messages the agent can interpret and relay to users.

---

## Einstein Trust Layer

### What It Does
- **Data masking:** PII is masked before being sent to the LLM; unmasked in the response.
- **Toxicity detection:** Screens inputs and outputs for harmful content.
- **Grounding:** Connects responses to CRM/Data Cloud context rather than relying on model hallucination.
- **Audit trail:** All LLM interactions are logged in the Einstein Audit Trail (org-level).
- **Zero retention:** Salesforce does not use customer data to train foundation models.

### Grounding Patterns
- **CRM grounding:** Agent actions pull live record data (Account, Case, Order) as context.
- **Data Cloud grounding:** Unified Individual profile provides full cross-source customer context.
- **Knowledge grounding:** Einstein Search Answers or Knowledge articles provide domain-specific answers.
- **Prompt Template grounding:** Merge fields inject structured data into the LLM prompt at runtime.

---

## Prompt Template Patterns

### Template Types

| Type | Use Case |
|------|----------|
| Field Generation | Generate a single field value (email subject, case summary) |
| Sales Email | Personalized outreach with CRM merge fields |
| Flex Template | Multi-purpose; composable with dynamic inputs |

### Writing Effective Prompts
- Be explicit about output format: "Respond in 3 bullet points, each under 20 words."
- Ground with merge fields: inject Account name, product, recent activity directly.
- Set the persona: "You are a helpful Salesforce service agent for Acme Corp."
- Include negative instructions: "Do not mention competitor products."
- Test with edge cases: empty fields, unusual characters, adversarial inputs.

---

## Deployment Patterns

### Channels

| Channel | Agent Type | Notes |
|---------|-----------|-------|
| Service Cloud Messaging | Customer-facing | Embedded in Service Console |
| Experience Cloud | Customer-facing | Self-service portal |
| Slack | Employee-facing | Requires Slack + Agentforce license |
| Einstein Copilot (CRM) | Employee-facing | Native CRM sidebar |
| Custom (API) | Either | Agentforce API for custom deployments |

### Rollout Approach
1. **Pilot with a narrow topic** — single use case, limited audience.
2. **Monitor with Einstein Conversation Insights** — review transcripts, identify misroutes.
3. **Iterate on instructions and actions** before expanding topics.
4. **Set human handoff thresholds** before going broad — never deploy customer-facing agents without an escalation path.
5. **Define KPIs upfront:** containment rate, CSAT, resolution time, escalation rate.

---

## Prompt Caching

Agentforce uses Claude models under the hood and inherits Claude's prompt caching capabilities automatically — no explicit configuration required.

### How It Works
- The first 1,024+ tokens of a prompt (system instructions, topic context, knowledge base content) are cached at the API level.
- Subsequent requests with identical prefixes retrieve from cache — only new tokens (user messages, dynamic context) are processed fresh.
- Cache TTL: 5 minutes of inactivity before expiration.

### Performance & Cost Benefits
| Metric | Impact |
|--------|--------|
| Latency | 20–50% reduction in time-to-first-token |
| Cost (cache write) | ~25% of standard input token cost |
| Cost (cache read) | ~10% of standard input token cost |
| Best case savings | Up to 90% on repeated large-context requests |

### When Caching Helps Most
- **Large, stable system prompts** — topic instructions, company context, persona definitions.
- **Knowledge base grounding** — long documents or product catalogs injected as context.
- **High-volume conversations** — many users hitting the same agent configuration.
- **Batch operations** — same knowledge base queried with different user inputs.

### Cache Invalidation
Cache is invalidated when:
- Any content in the cached region changes (even a single character).
- Token counts shift due to additions or deletions before the cache boundary.
- A new conversation thread begins (separate API call).

### Architectural Best Practices
- Put **static content first** in the prompt (system instructions, grounding documents) — this is the cacheable region.
- Put **dynamic content last** (user messages, live record data) — this is processed fresh each time.
- Avoid unnecessary variation in system prompts across requests — consistency maximizes cache hits.
- Monitor cached vs. non-cached token usage in API logs to validate ROI.

### Limits & Gotchas
- Minimum cacheable prefix: 1,024 tokens — smaller system prompts bypass caching entirely.
- Cache is per-conversation-thread; a new session starts fresh.
- Caching is managed transparently by the platform — there is no manual cache control in Agentforce today.

---

## Common Pitfalls

| Area | Pitfall | Fix |
|------|---------|-----|
| Topic Design | Too broad a topic scope — agent gets confused | Split into focused topics with clear boundaries |
| Instructions | Contradictory rules in the same topic | Audit instructions for conflicts before deploy |
| Actions | Action returns unstructured data | Return clean, labeled key-value outputs |
| Grounding | No Data Cloud connection → agent lacks customer context | Connect Data Cloud Unified Profile as grounding source |
| Escalation | No human handoff action defined | Always include a Transfer to Agent action |
| Testing | Only tested happy path | Test no-match scenarios, partial inputs, adversarial inputs |
| Trust Layer | Assuming PII masking covers all compliance needs | Validate data residency and retention requirements separately |

---

## GA vs. Evolving Features (Early 2026)

### Generally Available
- Agentforce for Service (customer-facing messaging, chat)
- Agentforce for Sales (Einstein Copilot in CRM)
- Atlas Reasoning Engine
- Einstein Trust Layer (masking, toxicity, audit trail, zero retention)
- Prompt Builder (Field Generation, Sales Email, Flex templates)
- Flow and Apex actions
- Data Cloud grounding
- Slack deployment channel
- Agentforce Agent Builder (UI-based configuration)

### Verify Current Status
- **Agentforce for specific industry clouds** (Health, Financial Services) — compliance posture varies; confirm with account team.
- **Multi-agent orchestration** — agent-to-agent handoff patterns evolving rapidly; check current release notes.
- **Voice channel for Agentforce** — in active development; validate GA status.
- **Agentforce API (headless/custom deployments)** — GA but feature set expanding each release.

---

## Architecture Decision Guide

| Scenario | Recommendation |
|----------|---------------|
| Customer self-service (chat/messaging) | Agentforce for Service, customer-facing channel |
| Internal CRM productivity | Agentforce Copilot (employee-facing) |
| Proactive outreach / event-driven | Background agent triggered by Data Cloud Data Action |
| Complex multi-step workflow with CRM updates | Flow-based actions for structured steps |
| Personalized content generation | Prompt Templates with CRM/Data Cloud grounding |
| Regulated industry (healthcare, finance) | Validate Trust Layer compliance posture before design |
| Need unified customer context in agent responses | Data Cloud grounding is required |
