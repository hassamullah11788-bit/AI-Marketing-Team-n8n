AI Marketing Team

A multi-agent AI system built on n8n that generates complete marketing campaigns from a single request. A team of specialized AI agents — Copywriter, SEO Writer, Email Specialist, Social Media Manager, and Brand Voice Specialist — collaborates sequentially to produce publish-ready marketing content, delivered via Telegram.

---

Overview

A user submits a campaign brief through a Telegram chat interface:

> "Create a launch campaign for our new project management tool."

The system generates and returns:

- **Website copy** — headlines, calls-to-action, product descriptions
- **SEO blog post** — long-form, keyword-optimized content with structured headers
- **Email sequence** — a multi-part welcome/nurture series
- **Social media content** — a Twitter/X thread and LinkedIn posts
- **Brand voice review** — a final consistency pass unifying tone across all deliverables

Output is compiled automatically and returned as one or more messages, with dynamic chunking to accommodate platform message-length limits.

---

Architecture

Telegram Trigger
│
▼
Copywriter Agent
│
▼
SEO Writer Agent
│
▼
Email Specialist Agent
│
▼
Social Media Manager Agent
│
▼
Brand Voice Specialist Agent
│
▼
Message Splitter
│
▼
Telegram Output

Each specialist is implemented as an independent n8n sub-workflow, invoked sequentially via **Execute Sub-workflow** nodes. Each runs its own LLM call with a role-specific prompt and a temperature setting tuned to its task — higher for creative copywriting, lower for structured or precision-sensitive content.

---

## Tech Stack

| Component | Purpose |
|---|---|
| n8n | Workflow orchestration and agent coordination |
| Groq (`llama-3.1-8b-instant`) | Inference for specialist agents |
| Google Gemini (`gemini-2.0-flash`) | Inference for the final compilation step |
| Telegram Bot API | User-facing chat interface |
| ngrok | Local webhook tunneling for development |

---

## Design Decisions

**Sequential execution over autonomous tool-calling.**
The initial design used a single orchestrator agent that autonomously selected which specialist tools to invoke. In practice, issuing five to six LLM calls in rapid succession consistently exceeded free-tier rate limits — both token-per-minute and request-per-minute — across every provider tested. Restructuring the system into a fixed sequential pipeline, where each specialist executes only after the prior step completes, eliminated these collisions and substantially improved reliability. This trades dynamic task routing for deterministic, production-stable execution — an appropriate tradeoff for a free-tier deployment, and a deliberate one rather than an accidental simplification.

**Multi-provider architecture.**
Specialist agents run on Groq, chosen for inference speed and favorable free-tier throughput on smaller models. The final compilation step — which processes substantially more tokens, since it consumes the combined output of all five specialists — runs on Gemini instead, drawing from a separate quota pool with a higher per-request ceiling. This prevents any single provider's rate limit from bottlenecking the full pipeline.

**Dynamic content chunking.**
Generated campaign content frequently exceeds Telegram's 4,096-character message limit. Rather than truncating output or hardcoding a fixed split, a dedicated Code node dynamically partitions the final text into appropriately sized segments before dispatch.

**Constrained output formatting.**
Each specialist's prompt explicitly excludes commentary, meta-narration, and "before/after" framing, ensuring the final deliverable reads as production-ready content rather than a visible AI editing process.

---

## Setup

1. Import each workflow JSON file from `/workflows` into n8n (Director, Copywriter, SEO Writer, Email Specialist, Social Media Manager, Brand Voice Specialist).
2. Provision API credentials:
   - Groq API key ([console.groq.com](https://console.groq.com))
   - Google Gemini API key ([aistudio.google.com](https://aistudio.google.com))
   - Telegram Bot token (via [@BotFather](https://t.me/botfather))
3. Attach credentials to the corresponding Chat Model nodes in each workflow.
4. Configure a public webhook URL (ngrok for local development, or your n8n Cloud instance's native URL).
5. Publish the five specialist workflows, followed by the Director workflow.
6. Submit a campaign brief via Telegram to verify end-to-end execution.

---

 Planned Improvements

- **Human-in-the-loop approval** — pause execution pending explicit review before content is finalized or published.
- **CRM integration** — route generated campaigns and leads into HubSpot or an equivalent platform.
- **Parallelized execution with provider load-balancing** — distribute specialist calls concurrently across multiple providers on paid API tiers to reduce total latency.
- **Additional specialist agents** — Video Script Writer, Paid Ads Specialist.
- **Persistent memory** — replace in-memory session state with a Postgres-backed memory store for cross-session context retention.


<img width="602" height="488" alt="image" src="https://github.com/user-attachments/assets/2b94afee-17fd-48b2-9b20-baff1527ca52" />
<img width="933" height="413" alt="Workflow" src="https://github.com/user-attachments/assets/de3565a9-cd9c-4181-ba7d-cec16b0a58d3" />


