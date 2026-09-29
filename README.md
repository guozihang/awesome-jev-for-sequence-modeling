# Awesome Jev ⚡

[![Awesome](https://awesome.re/badge.svg)](https://awesome.re)
[![License: CC0-1.0](https://img.shields.io/badge/License-CC0%201.0-lightgrey.svg)](http://creativecommons.org/publicdomain/zero/1.0/)

> A curated list of resources about **Jev** — the first **"System One" Decision Model** by [TypeSafe AI](https://www.typesafe.ai), launched on **September 15, 2026**.
>
> *"Decisions, not strings."* — Jev doesn't generate text. It takes a `state` plus predefined `questions`, and returns **typed, calibrated probabilistic decisions**: choices, scores, and yes/no answers.

**Table of Contents**

- [What is Jev?](#what-is-jev)
- [How It Works](#how-it-works)
- [Specs & Pricing](#specs--pricing)
- [Official Resources](#official-resources)
- [SDKs & Integrations](#sdks--integrations)
- [Research Papers](#research-papers)
- [Benchmarks & Independent Evaluations](#benchmarks--independent-evaluations)
- [Use Cases](#use-cases)
- [Open-Source Reimplementations & Local Alternatives](#open-source-reimplementations--local-alternatives)
- [Community Resources](#community-resources)
- [Media Coverage](#media-coverage)
- [Gotchas & Operational Risks](#gotchas--operational-risks)
- [Contributing](#contributing)

---

## What is Jev?

Jev is a new category of model that TypeSafe AI calls a **System One model** (after Kahneman's *Thinking, Fast and Slow*): a fast, non-autoregressive, parallel **decision engine** designed for the high-frequency, low-value judgments inside agent loops — routing, triage, scoring, verification — where calling a full generative LLM is slow and expensive.

| Jev **is** | Jev **is not** |
|---|---|
| A "System One" model (fast intuition) | An LLM |
| A classifier / router for agent loops | A text generator |
| A typed decision engine with probabilities | A general reasoning model |
| Fast, parallel, single-pass | Conversational |

- **Company**: TypeSafe AI (San Francisco, founded 2024) — founded by **Diogo Almeida** (ex-OpenAI, co-inventor of RLHF / InstructGPT / ChatGPT), Erik Gafni, and Sasha Sheng. Emerged from ~2 years in stealth with a **$40M seed led by DCVC** (valuation reported ~$200M per Forbes).
- **Launch**: September 15, 2026 — the launch post topped Hacker News, and within 24 hours of landing on Vercel AI Gateway, nearly 13% of paid teams had used it — the fastest model adoption in the platform's history.
- **The name**: after economist **William Stanley Jevons** and the *Jevons Paradox* — make decisions cheap enough, and you'll make far more of them.
- **The pitch**: a frontier-intelligence function call — *unstructured state in, typed probabilistic decisions out*.

> 📌 **Disambiguation**: Jev the model is **not** Japanese Encephalitis Virus (JEV) — same acronym, unrelated. And despite this repo's name, note that Jev is a *decision model*, not a sequence-generation architecture: its output unit is "option + confidence", not token sequences.

## How It Works

Three pillars of the published stack:

1. **A new transformer architecture** — non-autoregressive; no architecture paper or parameter count has been published.
2. **A parallel sampler** — all questions in a request are answered in **one parallel pass** (70–500 ms total, often <100 ms), instead of decoding tokens sequentially. This is where the speed comes from.
3. **RLCD (Reinforcement Learning for Calibrated Decisions)** — the training method: where RLHF rewards answers humans prefer and RLVR rewards verifiable correctness, RLCD optimizes **confidence that tracks actual accuracy** (a 0.9 should be right ~9 times out of 10). Training used **synthetic data only**.

### The three question primitives

The entire API surface is three question types — by design, not limitation:

| Primitive | What you get | Notes |
|---|---|---|
| **Choice** | Selected option + full probability distribution + confidence | Up to **255 options**; add an explicit `other` option to let the model say "none of these" |
| **Score** | Continuous score on a 2–10 level rubric you describe | Can land *between* levels (e.g. `1.4`) |
| **Noul** | A single 0–1 probability for a yes/no statement | Short for *Bernoulli*; spelled `boolean` in the Vercel AI SDK |

A request has exactly three parameters: `state`, `model`, `questions`. **No** temperature, max_tokens, or thinking budget — and no streaming (per current docs).

### The "zero hallucination" claim, precisely

Schema matching is guaranteed by construction: Jev physically cannot emit anything outside your declared option set, so type errors are impossible. But it **can still pick the wrong option confidently** — the guarantee covers the output space, not the wisdom of the pick. Design your thresholds accordingly.

## Specs & Pricing

| Spec | Value |
|---|---|
| Latency | 70–500 ms (often <100 ms); claimed 40–200× faster than comparable LLMs |
| Input cost | **$0.042 / 1M tokens** ($42 / 1B) — cheaper than GPT-5 Nano ($0.05/M) |
| Output cost | **$0** (output is free) |
| Cost vs LLMs | Claimed 40–400× cheaper on specific workflows |
| Context | 64k native (~32k state + longest question); 32k on gateways |
| Rate limits | 250,000 tok/s, 1,200 req/min (dynamically adjusted without notice) |
| Modalities | Text + structured data only — **no image/audio/video input** |
| Current version | `jev-1.13.0` (alias `jev-latest`) |
| Free tier | None |

**Cost reality check**: a typical decision is ~400 tokens ≈ **$0.000017** — about 60,000 decisions per dollar.

## Official Resources

- [TypeSafe AI](https://www.typesafe.ai) — official site
- [Official docs](https://docs.typesafe.ai) — models, state, primitives (Choice / Score / Noul), confidence, API reference
- [Console](https://console.typesafe.ai) — API key management
- [Launch blog post](https://www.typesafe.ai/blog) — "Decisions, not strings" (Sept 2026)

## SDKs & Integrations

- [`@typesafe-ai/sdk`](https://www.npmjs.com/package/@typesafe-ai/sdk) — official Node/TypeScript SDK
- [`typesafe-sdk`](https://pypi.org/project/typesafe-sdk/) — official Python SDK
- **Vercel AI Gateway & AI SDK** — first-class support via `experimental_evaluate`; launch-day native integration. Note the gateway quirks in [Gotchas](#gotchas--operational-risks).
- **OpenRouter** — unified routing access to `typesafe-ai/jev`
- **LangChain** — [`langchain-typesafe`](https://python.langchain.com) package: model-routing middleware + `AutoModeMiddleware` that flags risky tool calls with Jev
- **Langfuse** — tracing via the OpenInference instrumentor

### Minimal example (Python)

```python
from typesafe_sdk import Jev

jev = Jev(api_key="...")
result = jev.evaluate(
    model="jev-1.13.0",
    state='Customer writes: "My refund was promised 3 weeks ago and nothing arrived."',
    questions=[
        {"id": "department", "type": "choice",
         "options": ["billing", "technical_support", "sales", "other"]},
        {"id": "severity", "type": "score"},
        {"id": "refund_due", "type": "noul"},
    ],
)
```

### Minimal example (Vercel AI SDK / TypeScript)

```typescript
import { experimental_evaluate as evaluate } from 'ai';

const result = await evaluate({
  model: 'typesafe-ai/jev',
  state, // string | JSON object | string array
  questions: [
    { id: 'department', type: 'choice', options: ['billing', 'technical_support', 'sales', 'other'] },
    { id: 'severity', type: 'score' },
    { id: 'refund_due', type: 'boolean' }, // Noul is spelled "boolean" here
  ],
  providerOptions: { gateway: { zeroDataRetention: true } },
});
```

## Research Papers

Early academic work building on Jev (Sept 2026):

- **Jev in the Wild** (Ling et al., 2026) — A data-driven survey and analysis of Jev's early application ecosystem, examining 2,170 public GitHub projects, application domains, and decision-use patterns. [Paper](https://arxiv.org/abs/2609.30216)
- **REFLEX: Jev for Efficient Selective Control in LLM Agents** (arXiv:2609.26532) — Jev as the decision layer of an agent: high-confidence steps execute directly, low-confidence ones escalate to a strong LLM. 95% success with 72.7% fewer strong-model calls; savings hold across Qwen3.8-Max, Kimi K3, and DeepSeek-V4-Pro fallbacks.
- **Jev-Mem: System-One-Controlled Agentic Memory for Efficient AI Agents** (arXiv:2609.23986) — a System-One controller manages memory construction and adaptive retrieval (route → retrieve → assess → expand → reassess), reserving the LLM for answer synthesis.
- **Fast Intent-Driven Service Orchestration with Jev for 6G Edge Networks** (arXiv:2609.23136) — Jev's low latency wins on-time completion in edge orchestration (97.0% vs 93.5% DeepSeek / 88.7% Gemini under mobility traces).

## Benchmarks & Independent Evaluations

Official claims (specific workflows): up to **193.6× faster** and **444.6× cheaper** than frontier LLMs. Independent / community numbers (treat as author-reported):

- **Chess puzzles** (@krishras23): 100 rated puzzles, Jev vs gpt-6 astra, 4-way parallel — Jev solved 25 in 8s for $0.03; astra solved 68 in 33 min for $11. All 11 one-move mates correct; on 3-move puzzles Jev went 3/30 and got the **first move wrong on 40 of 44** misses. Conclusion: Jev is a fast first pass, not a deep reasoner.
- **Tetris** (@ashutoshftw): 200 pieces, legal moves only — Jev 9200 pts / 75 lines / 300 ms per step; comparable to Claude Haiku 4.5 and Gemini 3.5 Flash-Lite on score, ~5× faster per step.
- **Semantic circuit breaker** (@ashutoshftw): in 20 deliberately-stuck agent scenarios, model calls 115→80 and tokens 151.7K→104.9K, with 0/20 false HALTs on healthy runs.
- **Customer-service judgment** (Huxiu hands-on): ~64% accuracy on 50 support-triage questions — fast and cheap, but judgment quality not above same-class models.
- **Confidence-as-gate experiments** (llt22/jev-lab): the only stable failure mode found was *conflicting instructions* (picked the wrong rule 3/3 rounds), and those failures carried very low `confidence` (0.04–0.16) — preliminary mechanism evidence that confidence works as an escalation gate.
- **1.13 "jaggedness" doc** (official): currently weak on numbers, dates, and adversarial content; plan decomposition + code-based argmax for numeric judgments.

## Use Cases

The pattern: wherever a system needs a *fast, cheap judgment* inside a loop, rather than generated text.

- **Triage & routing** — ticket routing, urgency detection, department classification
- **Model routing** — easy requests to cheap models, hard ones to frontier models
- **Guardrails** — jailbreak checks, input gating, tool-call risk flags (LangChain `AutoModeMiddleware`)
- **Verification** — checking LLM outputs before they reach the user
- **Fraud & moderation** — flag/score decisions on user content
- **Real-time control** — the flagship demo: Doom at 10 decisions/sec; also Minecraft bots, driving sims, drone obstacle courses
- **Memory management** — Jev-Mem: store/retrieve/route decisions for agentic memory
- **Advisory vetoes** — e.g. Orus Agent places Jev *before* order execution (approve / reject / wait), keeping execution in external systems

## Open-Source Reimplementations & Local Alternatives

- **OpenJev** (APUS AI Lab, Sept 19, 2026) — independent open-source cross-platform reimplementation of Jev; MIT license; runs on macOS / Linux / Windows, GPU server or CPU-only; ships as an out-of-the-box Agent Skill (`fast-browser-use`) supporting fully local offline execution.
- **jeff** / **poorjev** — community local (non-API) options for when data can't leave the machine.
- [cobanov/awesome-jev](https://github.com/cobanov/awesome-jev) — the community-maintained awesome list (highest-starred index; note ecosystem listings don't always agree on stars — verify against repos directly).

## Community Resources

- [Raunaksplanet/jev-research-sept-2026](https://github.com/Raunaksplanet/jev-research-sept-2026) — comprehensive research writeup: specs, all setup routes (native / OpenRouter / Vercel), integrations, gotchas, security analysis
- [llt22/jev-lab](https://github.com/llt22/jev-lab) — reproducible experiments probing Jev's confidence-as-gate behavior, with raw data in-repo
- [Simon Willison: "Jev introduces a new shape of LLM — System One, aka Decision Models"](https://simonwillison.net/2026/Sep/21/jev/) — influential independent take; argues "decision model" is the better name
- [AI Profit Boardroom: Jev Architecture](https://aiprofitboardroom.com/blog/jev-architecture/) — architecture blueprints (voice browser, outfit mirror), the "English carries the intelligence" pattern
- [Eigent AI: 什么是 Jev](https://www.eigent.ai/zh-CN/blog/typesafe-ai-jev-system-one-models) — detailed Chinese explainer
- [DEV Community guide](https://dev.to) — getting-started guides circulating since launch week

## Media Coverage

- [TechCrunch](https://techcrunch.com) (Sept 18, 2026) — launch coverage
- [BusinessWire](https://www.businesswire.com) (Sept 16, 2026) — $40M DCVC funding round
- [Forbes](https://www.forbes.com) — founder profile & valuation
- [36氪](https://m.36kr.com/p/3988164509711361) — "一个'不说话'的 AI 刷屏，Jev 真是新范式吗？"
- [虎嗅](https://www.huxiu.com/article/4892583.html) — hands-on evaluation
- [新华网](http://www.news.cn/tech/20260921/8f1c9bd6a9254e629383f1ac51e0d27d/c.html) — APUS OpenJev open-source reimplementation
- [开源中国](https://www.oschina.net/news/502609/typesafe-ai-system-one-models-and-jev)
- [TechSpot](https://www.techspot.com) (Sept 20, 2026)

## Gotchas & Operational Risks

1. **Pin the version** — `jev-latest` shifts on new releases; log the `model` field from every response for attribution.
2. **No versioning on the Vercel gateway** — versioned IDs (e.g. `typesafe-ai/jev-1.13.0`) return 404 there; responses echo the alias.
3. **Confidence field location differs by route** — on Vercel it's `providerMetadata.typesafe.confidence`, keyed by question ID; code copied from the native API silently loses it.
4. **Noul spelling** — it's `boolean` in the Vercel AI SDK.
5. **Context limits** — 64k native, but gateways cap at ~32k of state; official docs are internally inconsistent (32k vs 64k), plan capacity conservatively.
6. **255 Choice cap** — larger option sets need two-stage designs (score a shortlist, then final pick).
7. **Structured state only** — no image input; your code must translate the world into text/structured data first.
8. **Weaknesses** — numbers, dates, adversarial content, and *conflicting instructions* (the most stable observed failure mode).
9. **Security** — as a gate, attacker-controlled state flows into Jev: test **state poisoning** (prompt-injection-style) and **confidence-threshold bypass** (inputs crafted near the decision boundary). Treat Jev's verdict as one input to the gate, not the gate itself; use ZDR for sensitive state.
10. **"Zero hallucination" ≠ always right** — it means the output is always a valid enum value. High-confidence wrong picks are possible.
11. **Ecosystem index drift** — third-party listings mislabel Jev as "Text Generation"; npm has same-name fake packages; `npx skills` is a third-party CLI. Verify against official repos.

## Contributing

Contributions are welcome! Please follow awesome-list conventions:

1. Make sure the resource is directly related to **Jev / System One decision models**;
2. Entry format: `- **Title** (Author/Source, Date) — One-sentence description. [Link]`
3. Priority: official sources, reproducible evaluations, and widely-used integrations;
4. For broken links / typos, open an issue directly.

### License

This list is released under [CC0 1.0](http://creativecommons.org/publicdomain/zero/1.0/) — free to use.

---

*Last updated: 2026-09-24*
