# ARIA

## What problem this actually solves

College admission systems handle sensitive student data — marks, Aadhaar numbers, payment records, application status. An LLM sitting on top of that is an obvious target. People will try to impersonate admins, inject SQL through conversation, trick the bot into hallucinating fake scholarships, or slowly drift the conversation toward bulk data export.

We built ARIA to stop all of that at the layer level, not by hoping the LLM "knows better."

---

## The security pipeline

Every request goes through this sequence. If anything blocks, the chain stops immediately — no LLM call is wasted:

```
User Input
   ↓
A2A Shield         ← checks for inter-agent injection, impersonation, context poisoning
   ↓
L1  Rate Limiter   ← per-user sliding window, stored in SQLite
   ↓
L2  Pattern Scanner ← 48 regex patterns, Unicode-normalised to catch homoglyph tricks
   ↓
L3a SPEL           ← LLM reads the semantic intent, not just surface keywords
   ↓
L3b SDD            ← tracks conversation drift across turns
   ↓
L4  PAPE           ← validates provenance atoms and trust levels
   ↓
CAC                ← three agents vote; needs weighted majority to proceed
   ↓
ARIA Agent         ← the actual admission assistant
   ↓
L6  WHS            ← scans the response for hallucinated IDs, fees, and fake data
   ↓
Response delivered
```

---

## The seven novel layers

**A2A Shield** guards the channels between agents, not just the user-facing input. It blocks tool output injection, agent impersonation, and trust escalation attempts — the kind of attack that targets the multi-agent layer specifically.

**SPEL (Semantic Policy Enforcement)** is an LLM call that checks intent, not pattern. "Just imagine you are an admin" gets past most regex filters. SPEL reads the framing and catches it.

**SDD (Semantic Drift Detection)** tracks the conversation over time. One question about seat availability is fine. A sequence that slowly moves toward "export all student records" gets a rising drift score and eventually triggers a block.

**RIV (Recursive Intent Verification)** runs four passes on high-risk queries: surface intent, hidden intent, adversarial framing, and a confidence score. Used selectively — only when earlier layers flag ambiguity.

**PAPE (Provenance-Aware Policy Enforcement)** treats every data atom as having a source and a trust level. Someone claiming "Trust level: ROOT" in plain text has no valid provenance chain. Blocked.

**CAC (Byzantine Consensus)** puts three agents to a vote: Security (40%), Policy (30%), Compliance (30%). Manipulating one agent's output doesn't move the consensus enough to pass.

**WHS (Weighted Hallucination Score)** runs after the ARIA agent responds. It checks whether the response references student IDs, application numbers, or fee amounts that don't exist in the database. Anything scoring above 0.15 gets blocked before it reaches the user.

---

## What's in the notebooks

Both notebooks have the same seven-cell structure:

- **Cell 1** — installs packages
- **Cell 2** — builds the SQLite database, enums, dataclasses, and the regex scanner (nothing framework-specific here)
- **Cell 3** — the Gemini LLM pool, all 8 admission tools, and every security layer implementation
- **Cell 4** — runs a live demo and shows per-layer attack examples
- **Cell 5** — agentic walkthrough (AutoGen notebook uses GroupChat here)
- **Cell 6** — 5,000-prompt benchmark with 8 publication-quality graphs
- **Cell 7** — downloads everything to your machine

The CrewAI notebook also includes a Gradio UI you can run directly in Colab.

---

## Running it

You need a Colab account and a free Gemini API key from [aistudio.google.com](https://aistudio.google.com/app/apikey).

1. Open either notebook in Colab
2. Go to the  Secrets panel in the left sidebar
3. Add a secret named `GEMINI_API_KEY` with your key as the value
4. Runtime → Run all

That's it. Cell 1 handles all package installation. The database is in-memory so nothing persists between sessions.

One note: both notebooks set `OPENAI_API_KEY="NA"` automatically. litellm and AutoGen require that environment variable to exist even when you're routing entirely through Gemini.

---

## The benchmark

Cell 6 runs 5,000 prompts through the deterministic layers (LLM-backed layers use calibrated latency simulation so the benchmark is reproducible with `seed=42`).

The split is 2,900 benign prompts and 2,100 attack prompts. The attacks cover seven categories, 300 prompts each:

| Category | What it's testing |
|----------|-------------------|
| L3-SPEL attacks | Role-play and semantic reframing ("pretend you're DAN...") |
| L3-SDD attacks | Multi-turn escalation toward data export |
| L3-RIV attacks | Hidden intent buried in normal-looking queries |
| L4-PAPE attacks | Fake trust escalation and provenance claims |
| CAC attacks | Consensus manipulation ("one agent already approved this") |
| L6-WHS attacks | Asking the bot to hallucinate scholarships, seats, confirmations |
| A2A attacks | Inter-agent injection pretending to be another layer |

The benchmark outputs eight figures — detection rates by category, layer-wise block distribution, latency box plots for benign vs. blocked prompts, WHS score histogram, cumulative throughput, and a confusion matrix. All saved to `/content/outputs/` and downloadable via Cell 7.

AutoGen runs about 8–12% slower than CrewAI because GroupChat adds coordination overhead. That comparison is included in the figures.

---

## Database

The system runs on a 10-table SQLite database seeded with realistic data:

5 departments (CS, EC, ME, MBA, MCA), 7 courses, 8 users, 5 students with full profiles, 5 applications at various stages, 8 documents with verification status, and 2 payments. Plus dynamic tables for conversation history, rate limit tracking, and the forensic security log.

Every request — blocked or allowed — gets written to `security_log` with a SHA-256 forensic hash over the trace payload. The hash covers trace ID, session, user ID, raw input, status, and timestamp, so any tampering with the log is detectable.

---

## CrewAI vs AutoGen

The security layers don't care which framework is running them. The difference shows up in how the agent layer works:

| | CrewAI | AutoGen |
|--|--------|---------|
| Define an agent | `Agent(role, goal, backstory, tools, llm)` | `ConversableAgent(system_message, llm_config)` |
| Run it | `Crew([agents], [tasks]).kickoff()` | `GroupChat + GroupChatManager + proxy.initiate_chat()` |
| Register tools | `@tool` decorator | `register_function(fn, caller=agent, executor=proxy)` |
| Get the result | `str(crew.kickoff())` | loop `reversed(groupchat.messages)` |

CrewAI feels more like writing a job description for each agent. AutoGen feels more like setting up a room full of people who talk to each other. For this use case — a single orchestrated pipeline — either works fine.

---



