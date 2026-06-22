# Awesome On-Device Mobile LLMs

*Production learnings from the [DataSapien](https://datasapien.com) Mobile SDK team.*

Curated, battle-tested guidance for shipping on-edge LLMs in real mobile apps — not another generic model list.

![Version](https://img.shields.io/badge/version-v0.0.1-blue)

---

## What we build

The [DataSapien Mobile SDK](https://docs.datasapien.com/docs/mobile-sdk/intro) is an on-device intelligence engine for Android, iOS, Flutter, and React Native. You configure flows and models in the **Orchestrator** (web dashboard); the SDK syncs and runs everything on the phone — LLM inference, local data vault, rules, and remotely published user journeys.

- [Developer portal](https://docs.datasapien.com/)
- [Hello World Journey tutorial](https://docs.datasapien.com/docs/quickstart/hello-world-journey) — end-to-end on-device flow in ~30 minutes

---

## Runtime comparison

We evaluated the runtimes mobile teams most often ask about. **SDK status** reflects what ships today; **R&D status** reflects active experimentation.

| Runtime | Android | iOS | Model formats | Acceleration | Mobile maturity | SDK status | R&D status |
| --- | --- | --- | --- | --- | --- | --- | --- |
| **llama.cpp** | Yes | Yes | GGUF | CPU, Metal (iOS), GPU/Vulkan (Android builds) | Production-ready | **Shipped (default)** | Production |
| **LiteRT** | Yes | Yes | TFLite / LiteRT graphs | CPU, NNAPI, GPU, NPU (Android) | Production-ready | **Optional module** | **Experimenting** |
| **Cactus** | Yes | Yes | Cactus-native | Platform-dependent | Maturing | **Optional module** | Evaluating |
| **MLX** | No | Yes (Apple Silicon) | MLX | CPU, GPU, Neural Engine | Growing iOS ecosystem | Planned | Planned |

### Experimenting with LiteRT-LM

We are actively experimenting with **LiteRT** through the SDK's optional `litert-module`, while **llama.cpp** remains the default for production workloads.

**Why we're exploring it:**

- **Performance and memory** — LiteRT-LM is built for efficient on-device inference across CPU, GPU, and NPU backends; we want to measure how it compares to GGUF + llama.cpp on the devices our customers actually ship on.
- **Agentic workflows** — features like thinking mode, constrained decoding, and function calling map closely to DataSapien Journeys (script steps, structured routing, on-device orchestration) — similar to grammar guardrails we use with llama.cpp in production ([intent classifier lab report](https://datasapien.com/building-a-real-on-device-intent-classifier-with-a-small-language-model/)).
- **Cross-platform** — a unified Google AI Edge stack on Android and iOS fits our plug-in runtime factory without changing `IntelligenceService` APIs.

Read Google's overview: [Blazing fast on-device GenAI with LiteRT-LM](https://developers.googleblog.com/blazing-fast-on-device-genai-with-litert-lm/) (Google Developers Blog, May 2026).

**Results coming soon** — we are benchmarking latency, memory, and output quality against llama.cpp on representative customer use cases. DataSapien experimentation results will be published in this README and as a lab report soon. LiteRT experimentation does not change current SDK behavior for existing integrations.

### Why we started with llama.cpp

- **Cross-platform** — one integration path for Android and iOS (GGUF via llama.cpp; iOS ships as `llama.xcframework`).
- **Ecosystem** — broad Hugging Face GGUF availability lets us map customer experiments to models quickly.
- **Proven on mobile** — active community, Metal on iOS, CPU/GPU paths on Android.
- **Fits our delivery model** — provision a model URL in Orchestrator, download to device, run locally via `IntelligenceService`.
- **LiteRT under evaluation** — Gemma-centric and agentic flows where the Google AI Edge stack may outperform raw GGUF; results pending.

### Pluggable runtime roadmap

We started llama.cpp-first because it covered the widest set of customer experiments with a single native stack. The SDK already uses a **factory pattern** with optional engines: LiteRT and Cactus load as plug-in modules when present on the classpath. LiteRT experimentation feeds directly into this path (`litert-module` classpath detection) — no app-level API changes when an engine graduates from R&D to recommended. **MLX** and additional runtimes are on the near-term roadmap — the goal is to match each customer use case to the best engine without rewriting app integration.

---

## How we pick models

There is no single "best" local LLM. Selection is always contextual: RAM, download size, latency budget, battery, and task type. We start from the **task**, test the **smallest plausible model**, and step up only when quality fails.

*Source: [DataSapien Lab Report: What's the Best Local LLM?](https://datasapien.com/datasapien-lab-reprwhats-the-best-local-llm/) (Feb 2026)*

### Three-tier approach (production)

| Tier | Model | Typical use case |
| --- | --- | --- |
| High-quality reasoning | Gemma-3n-e4b-it (Q4_K_M) | Complex analysis, multimodal journeys (e.g. YouTube Persona) |
| Fast, efficient inference | Qwen 2.5 SLM | Summarization, dynamic screen generation |
| Ultra-lightweight | Gemma 3 270M (Q8_0) | Classification, structured extraction, simple summarization |

**Principle:** model fit + task fit + audience fit — not the biggest model that fits on a flagship phone.

### Lab report: on-device intent classifier

We built a real **binary intent classifier** for a search box (support vs sales routing) — fully on-device, no cloud, sub-second latency. Full walkthrough: [Building a Real On-Device Intent Classifier with a Small Language Model](https://datasapien.com/building-a-real-on-device-intent-classifier-with-a-small-language-model/) (Feb 2026).

| Model | Size | Latency (after load) | Observed behavior | Verdict |
| --- | --- | --- | --- | --- |
| Gemma-3-270m-it-q8_0 | ~280 MB | ~0.1–0.2 s | Ignored decision boundaries; noisy outputs | Too weak |
| Qwen 2.5 0.5B | ~500 MB | ~0.1–0.2 s | Label collapse (always one class) | Unstable |
| Gemma 3n 4eb | ~4.5 GB | ~3.0–4.9 s | Good semantics, impractical size/cost | Good but expensive |
| **Qwen 2.5 3B (Q4_K_M)** | ~1.6–2 GB | ~0.2–0.7 s | Stable binary decisions, strong semantic separation | **Selected** |

*Raw inference data available on request — see blog post.*

**Takeaways we apply in production:**

1. **Deterministic routing** — `temperature: 0`, `topK: 1` for UX flows that need a single label, not creative variation.
2. **Prompt as contract** — define labels and boundaries; let the model handle semantics, not keyword lists.
3. **Grammar guardrails** — GBNF / regex output constraints as a final hardening step when smaller models leak extra text.

---

## Privacy and architecture

Most GDPR friction traces back to one decision: **centralising personal data**. On-device intelligence changes the compliance surface — not by avoiding regulation, but by removing custody of raw user data on your servers.

*Source: [GDPR Doesn't Block Innovation. Centralised Architecture Does.](https://datasapien.com/gdpr-architecture-not-regulation/) (Feb 2026)*

| GDPR article | Centralised systems | On-device (DataSapien architecture) |
| --- | --- | --- |
| **Purpose limitation (Art. 5(1)(b))** | Every new feature can require new consent and legal review | Device uses user's data for user's benefit; no secondary server-side use |
| **Data minimisation (Art. 5(1)(c))** | Justify every field you hold centrally | Nothing collected centrally; nothing transmitted by default |
| **Profiling (Art. 4(4))** | Recommendation engines trigger enhanced scrutiny | Device reasons about its owner — self-knowledge, not third-party surveillance |
| **Right to erasure (Art. 17)** | Purge DBs, backups, data pipelines, processors | Delete local store on device |

In the SDK:

- **[IntelligenceService](https://docs.datasapien.com/docs/mobile-sdk/services/intelligence-service)** — runs rules and on-device LLMs locally.
- **[MeDataService](https://docs.datasapien.com/docs/mobile-sdk/services/medata-service)** — stores local context in the on-device **Data Vault** for Journeys and personalization without sending raw data upstream.

Learn more: [Zero-Party Data & Zero-Shared Data](https://docs.datasapien.com/docs/overview#zero-party--zero-shared-data)

---

## Journeys: remotely deployed, on-edge agentic flows

Campaigns and intelligent flows should not wait on App Store review. **Journeys** are multi-step flows designed in the Orchestrator and executed on-device by the SDK — configuration, not a new binary.

*Example: [Ship a Father's Day Campaign Without Shipping an App Update](https://datasapien.com/ship-a-fathers-day-campaign-without-shipping-an-app-update/)*

| Capability | What it does |
| --- | --- |
| Flow designer | Screen, Question, Script, MeData, and conditional steps |
| JS script runner | On-device logic without shipping a new app build |
| Local LLM runner | llama.cpp via `IntelligenceService` |
| MeData collectors | Structured local context stored in the Data Vault |
| Managed API connectors | External APIs orchestrated from the device |
| Remote publish | Orchestrator → SDK sync; live in minutes, not review cycles |

**Father's Day gift-finder (concrete flow):** home-screen banner → shopper answers questions about their dad → on-device logic builds a curated gift list → tap to add to the app's cart → checkout in the native flow. Answers stay on the phone (**Zero-Shared Data**). MeData captured (e.g. "father's interests") can target a smarter campaign next year — still evaluated on-device.

- [Hello World Journey](https://docs.datasapien.com/docs/quickstart/hello-world-journey) — build your first flow
- [Emotica showcase](https://docs.datasapien.com/docs/showcase/ds-emotica) — on-device emotion tracking + AI analysis

---

## SDK services

| Service | Role |
| --- | --- |
| [IntelligenceService](https://docs.datasapien.com/docs/mobile-sdk/services/intelligence-service) | Rules and AI/LLM invoke, model download and load |
| [MeDataService](https://docs.datasapien.com/docs/mobile-sdk/services/medata-service) | On-device Data Vault — local context for Journeys and personalization |
| [JourneyService](https://docs.datasapien.com/docs/mobile-sdk/services/journey-service) | Execute remotely published Journeys |

**On-edge intelligence today:** rules for deterministic flows, on-device LLMs for generative tasks. Use rules when the logic is known; reach for an LLM when the output space is open-ended. See also: [Intelligence Service API Reference docs](https://docs.datasapien.com/api-reference/api-reference/sdk-reference/intelligence-service).

---

## How we work with customers

We map client requests to on-edge LLM fit before picking a model or runtime:

1. **Task** — classification, routing, summarization, open-ended chat?
2. **Device floor** — minimum RAM, offline requirement, latency budget.
3. **Privacy** — can any data leave the device? What stays in the vault?
4. **Experiment** — prototype in Orchestrator and test on real devices before production rollout.

Model and runtime selection is **use-case driven**, not leaderboard-driven. We iterate with customers until the smallest model that passes their quality bar ships reliably on their audience's devices.

---

## Learn more

| Resource | Link |
| --- | --- |
| Developer docs | [docs.datasapien.com](https://docs.datasapien.com/) |
| Hello World Journey | [Getting started](https://docs.datasapien.com/docs/quickstart/hello-world-journey) |
| SDK installation (Android / iOS / Flutter / RN) | [Installation guides](https://docs.datasapien.com/docs/category/installation) |
| Best local LLM lab report | [Blog](https://datasapien.com/datasapien-lab-reprwhats-the-best-local-llm/) |
| Intent classifier lab report | [Blog](https://datasapien.com/building-a-real-on-device-intent-classifier-with-a-small-language-model/) |
| GDPR architecture | [Blog](https://datasapien.com/gdpr-architecture-not-regulation/) |
| Journeys without app update | [Blog](https://datasapien.com/ship-a-fathers-day-campaign-without-shipping-an-app-update/) |
| DataSapien | [datasapien.com](https://datasapien.com) |

---

## Share your production use case

What on-edge LLM use cases are you shipping or exploring?

Open a thread in [GitHub Discussions](https://github.com/Data-Sapien/awesome-on-device-mobile-llms/discussions) — use the **Production use cases** template and tell us:

- Task and user flow
- Device / latency constraints
- Model and runtime (if any)
- Privacy requirements

We use community input to prioritize lab reports, runtime plug-ins, and documentation.

---

*Maintained by the [DataSapien](https://datasapien.com) engineering team.*
