# Pitchwise — Agentic Pitch Coaching System

A multi-agent AI system that classifies, coaches, and iteratively scores startup pitches against a firm's proprietary rubric — built to give a startup accelerator's partners a reliable first-pass review before pitches reach human judgment.

## The Problem

Pitchwise -- a startup accelerator -- was drowning in founder pitch drafts ahead of demo day. Partner review quality was slipping under volume, and generic AI writing tools failed because they didn't reflect the firm's own opinionated, hard-won standards — only what a model picked up from the open internet.

## What This Builds

An **evaluator-optimizer agent loop** that takes a rough pitch draft and returns a polished, rubric-scored revision:

1. **Classify** — a structured-output agent sorts the pitch into `consumer`, `b2b_saas`, or `deep_tech`.
2. **Coach** — a category-specific agent produces the first rewrite.
3. **Evaluate** — an agent pulls the matching rubric live from an MCP tool server (not baked into the prompt) and scores the draft criterion-by-criterion.
4. **Optimize** — if any criterion fails, a revision agent rewrites the draft to address the flagged gaps, and the loop repeats (capped at 3 iterations).

```
draft ─▶ Classifier ─▶ Coach ─▶ ┌─────────────────────────────┐
                                │  Evaluator ──(fail)──▶ Optimizer │
                                │      │                       │
                                │    (pass) ──────▶ final draft │
                                └─────────────────────────────┘
                                   ▲ rubric fetched via MCP tool call
```

## Tech Stack

| Layer | Technology |
|---|---|
| Agent orchestration | [OpenAI Agents SDK](https://github.com/openai/openai-agents-python) (`Agent`, `Runner`) |
| Tool integration | **MCP (Model Context Protocol)** — rubric served over `MCPServerStdio`, not hardcoded in prompts |
| Structured outputs | **Pydantic** schemas (`PitchClassification`, `PitchEvaluation`) enforcing typed, validated agent responses |
| LLM | `gpt-4o-mini` |
| Runtime | Python, `asyncio` |

**Why MCP matters here:** the rubric lives in a separately managed server (`pitchwise_server.py`), decoupled from the agent prompts. Partners can update their playbook without touching any model code, and every evaluation automatically uses the current standard — no risk of stale rubric copies drifting across prompts.

## Design Decisions That Mattered

- **Rubric-grounded scoring, not model opinion.** The evaluator is explicitly instructed to score *only* against the fetched rubric — never its own sense of what makes a good pitch — which is what off-the-shelf AI coaching tools got wrong.
- **All-or-nothing pass criteria.** A pitch passes only when *every* rubric criterion is satisfied; one strong section can't compensate for a missing one. This mirrors how the partners actually review.
- **No hallucinated traction.** The optimizer is instructed never to invent facts (customers, metrics, market size) that aren't already in the draft. Missing evidence surfaces as an explicit placeholder (e.g. `[FIRST CUSTOMER NEEDED]`) instead of a fabricated data point — critical for a system whose output partners need to trust.
- **Bounded iteration.** A 3-round cap prevents burning API calls on pitches that need new facts, not better phrasing, to pass.

## Results

Ran end-to-end on 6 founder pitch drafts across all three categories:

| Pitch | Category | Iterations to converge | Outcome |
|---|---|:-:|---|
| A | Consumer | 1 | ✅ Passed |
| B | B2B SaaS | 3 | ⚠️ Capped — missing traction evidence |
| C | Deep Tech | 3 | ⚠️ Capped — missing named first customer |
| D | — | 2 | ✅ Passed |
| E | — | 1 | ✅ Passed |
| F | — | 1 | ✅ Passed |

**4 of 6 pitches (67%) reached a passing score automatically**, most within a single revision — meaning the majority of first-pass review work can be fully offloaded from partners.

The 2 pitches that didn't converge are the more interesting result: the system correctly identified that further rewriting wouldn't help, because the gap was a **missing business fact** (no named first customer, no concrete traction signal), not a writing problem. Rather than mask that gap with plausible-sounding filler, the loop surfaced it explicitly — turning a potential failure mode (AI over-polishing a weak underlying case) into a useful signal for the founder about what evidence to go get before their next partner session.

## Repo Layout

```
pitchwise_server.py           MCP server exposing `get_rubric()` — the firm's playbook as a tool
openai_setup.py                Environment/config bootstrap
PitchwiseAgenticBuild.ipynb    Full build: agents, evaluator-optimizer loop, test run, analysis, partner memo
requirements.txt               openai-agents, openai, mcp
```

## Author

**Nguyen Bui** — [mayxbui@gmail.com](mailto:mayxbui@gmail.com)
