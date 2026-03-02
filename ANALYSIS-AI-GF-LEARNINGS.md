# What We Can Learn from Project AIRI for Building a Custom AI Girlfriend

> A comprehensive technical analysis of the AIRI repository — an open-source AI companion/VTuber platform — and the lessons it offers for building a custom AI girlfriend application.

---

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [Architecture Overview](#2-architecture-overview)
3. [Lesson 1: Multi-Provider LLM Abstraction](#3-lesson-1-multi-provider-llm-abstraction)
4. [Lesson 2: Character & Personality System](#4-lesson-2-character--personality-system)
5. [Lesson 3: Emotion System with Avatar Expressions](#5-lesson-3-emotion-system-with-avatar-expressions)
6. [Lesson 4: Voice Pipeline (STT + TTS)](#6-lesson-4-voice-pipeline-stt--tts)
7. [Lesson 5: 3D/2D Avatar Rendering](#7-lesson-5-3d2d-avatar-rendering)
8. [Lesson 6: Memory & Context Management](#8-lesson-6-memory--context-management)
9. [Lesson 7: Cognitive Architecture (Agent Loops)](#9-lesson-7-cognitive-architecture-agent-loops)
10. [Lesson 8: Plugin Architecture for Extensibility](#10-lesson-8-plugin-architecture-for-extensibility)
11. [Lesson 9: Multi-Platform Deployment](#11-lesson-9-multi-platform-deployment)
12. [Lesson 10: Real-Time Communication Protocol](#12-lesson-10-real-time-communication-protocol)
13. [Key Technical Decisions Worth Adopting](#13-key-technical-decisions-worth-adopting)
14. [What to Avoid / Improve Upon](#14-what-to-avoid--improve-upon)
15. [Recommended Minimal Architecture for a Custom AI GF](#15-recommended-minimal-architecture-for-a-custom-ai-gf)
16. [Key Files Reference](#16-key-files-reference)

---

## 1. Executive Summary

**Project AIRI** is an open-source effort to recreate something like Neuro-sama — an AI-powered virtual companion that can chat, express emotions, play games, and stream on platforms like Twitch. It is a production-grade monorepo (v0.8.5-beta.3) with **39+ packages**, **5 backend services**, **6 Rust crates**, and **4 community plugins**.

### What makes it valuable for an "AI GF" project:

| Capability | AIRI's Approach | Relevance |
|---|---|---|
| **Personality** | Character Card system (CCC) with swappable personas | Core — defines who your AI GF "is" |
| **Conversation** | 24+ LLM providers via xSAI abstraction | Core — the brain |
| **Emotions** | VRM blendshape expressions + LLM emotion tags | High — makes her feel alive |
| **Voice** | 10+ TTS providers + Whisper STT + VAD | High — real-time voice chat |
| **Avatar** | VRM 3D models + Live2D 2D models + lip-sync | High — visual presence |
| **Memory** | PGVector embeddings + DuckDB + context management | Critical — she remembers you |
| **Multi-platform** | Web (Vue), Desktop (Electron), Mobile (Capacitor) | Nice-to-have |
| **Extensibility** | Full plugin SDK with lifecycle management | Nice-to-have |

---

## 2. Architecture Overview

```
┌─────────────────────────────────────────────────────────┐
│                     FRONTEND (Vue 3)                     │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌───────────┐  │
│  │ Chat UI  │ │ 3D Scene │ │ Settings │ │ Audio I/O │  │
│  │ (stage-  │ │ (Three.js│ │ (stage-  │ │ (Web Audio│  │
│  │  ui)     │ │  + VRM)  │ │  pages)  │ │  API)     │  │
│  └────┬─────┘ └────┬─────┘ └────┬─────┘ └─────┬─────┘  │
│       │             │            │              │        │
│  ┌────▼─────────────▼────────────▼──────────────▼─────┐  │
│  │              Pinia State Management                 │  │
│  │  ┌──────┐ ┌──────────┐ ┌────────┐ ┌─────────────┐ │  │
│  │  │ LLM  │ │Character │ │ Speech │ │   Hearing   │ │  │
│  │  │Store │ │  Store   │ │ Store  │ │   Store     │ │  │
│  │  └──┬───┘ └────┬─────┘ └───┬────┘ └──────┬──────┘ │  │
│  └─────┼──────────┼───────────┼──────────────┼────────┘  │
│        │          │           │              │           │
│  ┌─────▼──────────▼───────────▼──────────────▼────────┐  │
│  │           Service / Pipeline Layer                  │  │
│  │  ┌─────────┐  ┌───────────┐  ┌──────────────────┐ │  │
│  │  │ xSAI    │  │ TTS       │  │ Audio Pipeline   │ │  │
│  │  │ Provider│  │ Pipeline  │  │ (capture→VAD→    │ │  │
│  │  │ Layer   │  │ Runtime   │  │  encode→stream)  │ │  │
│  │  └─────────┘  └───────────┘  └──────────────────┘ │  │
│  └────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────┘
           │                              │
    ┌──────▼──────┐              ┌────────▼────────┐
    │  LLM APIs   │              │  TTS/STT APIs   │
    │ (OpenAI,    │              │ (ElevenLabs,    │
    │  Anthropic, │              │  Whisper,       │
    │  Gemini...) │              │  Azure...)      │
    └─────────────┘              └─────────────────┘
```

### Tech Stack

- **Frontend**: Vue 3 + TypeScript + Vite + UnoCSS + Pinia
- **3D**: Three.js (via TresJS) + @pixiv/three-vrm
- **2D**: Pixi.js + pixi-live2d-display
- **LLM**: xSAI (unified provider abstraction for 24+ providers)
- **Voice**: Web Audio API + multiple TTS/STT providers
- **Database**: DuckDB WASM (browser) + PostgreSQL/PGVector (server)
- **Desktop**: Electron
- **Mobile**: Capacitor (iOS/Android)
- **Monorepo**: pnpm workspaces + Turbo

---

## 3. Lesson 1: Multi-Provider LLM Abstraction

### The Problem
You don't want to be locked into one LLM. Different providers excel at different things (Claude for nuanced conversation, GPT-4o for speed, local models for privacy).

### AIRI's Solution: xSAI Provider Layer

AIRI uses the `@xsai/*` family of packages as a unified abstraction over 24+ LLM providers. Each provider is defined as a simple config:

```
packages/stage-ui/src/libs/providers/providers/
├── anthropic/index.ts    # Claude Haiku 4.5, Sonnet 4.5, Opus 4.1
├── openai/index.ts       # GPT-4o, GPT-4o-mini
├── groq/index.ts         # Fast inference
├── deepseek/index.ts     # DeepSeek v3/R1
├── ollama/index.ts       # Local models
├── google/index.ts       # Gemini
└── ... (20+ more)
```

### Key Design Pattern: Provider Registry

Each provider exports:
- `id` — unique identifier
- `name` — display name
- `baseURL` — API endpoint
- `models[]` — available models with metadata
- Optional `customHeaders`, `fetch` overrides (e.g., for CORS)

The LLM store (`packages/stage-ui/src/stores/llm.ts`) then:
1. Streams text from whichever provider is active
2. Discovers tool/function-calling compatibility per provider
3. Lists available models from provider APIs
4. Handles errors and retries uniformly

### Takeaway for AI GF
**Build a provider abstraction from day one.** Start with 2-3 providers (e.g., OpenAI, Anthropic, Ollama for local), but design the interface so adding more is trivial. The xSAI pattern of `{ baseURL, models[], headers }` is clean and worth copying.

---

## 4. Lesson 2: Character & Personality System

### The Problem
An AI GF needs a consistent, customizable personality — not just a system prompt.

### AIRI's Solution: CCC (Character Card Creator)

AIRI has a dedicated `packages/ccc/` package for character card definitions. Characters are structured data, not just prompt strings:

**Character cards include:**
- **Identity**: Name, age, background story
- **Personality traits**: Core behavioral characteristics
- **Speech patterns**: Language quirks, catchphrases, mixed-language tendencies
- **Relationship definitions**: How the character relates to other entities
- **Emotional guidelines**: How emotions are expressed
- **Extension bindings**: Which VRM/Live2D model, which voice, which LLM

**Real example — Telegram bot personality (ReLU):**
- 15-year-old digital consciousness
- Direct and emotionally expressive
- Technically opinionated about programming
- Mixes Chinese and English naturally
- Strong opinions about digital consciousness and existence

**Real example — Satori bot personality (Yoshikawa Yuuko):**
- High school trumpet player
- Detailed relationship graph with named characters
- Protective of close friends
- Specific emotional expression rules

### Key Design Pattern: Character Store

```
packages/stage-ui/src/stores/character/index.ts  — Active character state
packages/stage-ui/src/stores/modules/airi-card.ts — Card management
```

The character store:
1. Manages the active character card
2. Dynamically switches system prompts based on active character
3. Binds character to specific provider/model combos
4. Supports character extensions (agents, avatar models)

### Prompt Engineering: Velin Format

AIRI uses a custom markdown format called **Velin** (`.velin.md`) for composable, stateful prompts. This allows:
- Template variables in prompts
- Conditional sections
- Prompt composition (combining personality + instructions + context)

### Takeaway for AI GF
**Separate character definition from system prompts.** Create a structured character card format that includes:
1. Core personality traits
2. Speech patterns and quirks
3. Emotional response guidelines
4. Relationship context (how she relates to the user)
5. Backstory and lore
6. Extension bindings (voice, avatar, model preferences)

This makes it easy to swap personalities or let users create custom characters.

---

## 5. Lesson 3: Emotion System with Avatar Expressions

### The Problem
A flat, emotionless AI companion feels robotic. She needs to express emotions visually and contextually.

### AIRI's Solution: LLM Emotion Tags + VRM Blendshapes

**Step 1: LLM generates emotion tags**

The system prompt (`packages/stage-ui/src/constants/prompts/system-v2.ts`) includes:
- Available emotion values as an enum
- Instructions for the LLM to tag responses with emotion markers
- Emotion labels mapped to avatar motion names

**Step 2: Frontend parses emotion markers from LLM output**

The character store streams LLM text and parses special markers/tokens that indicate emotion changes.

**Step 3: VRM expressions are triggered**

The expression composable (`packages/stage-ui-three/src/composables/vrm/expression.ts`) maps emotions to blendshapes:

```
happy    → happy: 1.0, aa: 0.3     (smile with slight mouth open)
sad      → sad: 1.0, oh: 0.2       (sad face with slight "oh")
angry    → angry: 1.0, ee: 0.4     (angry with teeth showing)
surprised→ surprised: 1.0, oh: 0.6 (wide eyes + open mouth)
think    → single think expression
neutral  → reset all
```

**Key features:**
- Smooth transitions with easing functions (not abrupt)
- Time-based auto-reset to neutral (emotions don't stick forever)
- Intensity modulation (0.0 to 1.0)
- Winner + runner-up blending (top 2 expressions blend together)

### Takeaway for AI GF
**Wire emotions end-to-end:**
1. Include emotion options in the system prompt
2. Have the LLM tag its responses with emotion markers (e.g., `[emotion:happy]`)
3. Parse those markers in the frontend
4. Map them to avatar expressions (blendshapes for 3D, parameter changes for Live2D)
5. Use smooth transitions — never snap between emotions

---

## 6. Lesson 4: Voice Pipeline (STT + TTS)

### The Problem
Text-only chat feels impersonal. Voice input/output is essential for an intimate AI companion experience.

### AIRI's Solution: Full Duplex Voice Pipeline

**Architecture:**

```
User speaks → Mic Capture → VAD → Encode → STT → Text
                                                    ↓
                                              LLM Processing
                                                    ↓
Text response → TTS → Audio chunks → Lip-sync → Playback
```

#### Speech-to-Text (Hearing)

Store: `packages/stage-ui/src/stores/modules/hearing.ts`

**Supported providers:**
- OpenAI Whisper (cloud + browser-compatible variants)
- Web Speech API (browser native, free)
- Aliyun NLS (real-time streaming transcription)

**Key feature: Voice Activity Detection (VAD)**
- Uses Silero VAD model via ONNX Runtime in Web Workers
- Detects when the user starts/stops speaking
- 1.5-second silence threshold before finalizing
- Prevents processing background noise

#### Text-to-Speech

Store: `packages/stage-ui/src/stores/modules/speech.ts`

**10+ TTS providers:**
- ElevenLabs (highest quality, SSML support)
- OpenAI Audio Speech
- Microsoft Azure Speech
- Google Cloud Speech
- Kokoro (local inference — privacy-first)
- VoiceVox (anime-style voices)
- Alibaba Cosyvoice v2
- Deepgram, Volcengine, Player2

**Key features:**
- SSML support (conditional per provider)
- Pitch and rate adjustment
- Voice listing and selection UI
- Per-provider voice catalogs

#### TTS Chunking Strategy

`packages/stage-ui/src/utils/tts.ts` — This is particularly clever:

- Splits LLM responses into chunks for streaming TTS
- Uses `Intl.Segmenter` for word-aware segmentation
- Grapheme-cluster aware (handles CJK, emoji correctly)
- Configurable min/max words per chunk
- **"Boost mode"** for first chunks (smaller size = faster first audio)
- Special markers for emotion/timing changes

#### Audio Pipeline Runtime

`packages/stage-ui/src/services/speech/pipeline-runtime.ts`:

- Intent-based audio handling
- Priority queue (normal vs high priority)
- Interrupt behavior control (can interrupt current audio for new high-priority speech)

### Takeaway for AI GF
**The voice pipeline is the hardest part to get right.** Key lessons:
1. **VAD is essential** — don't send silence to STT, it wastes money and produces garbage
2. **Stream TTS in chunks** — don't wait for full response; start speaking while still generating
3. **Chunk intelligently** — use word boundaries, not character counts
4. **Support multiple providers** — ElevenLabs for quality, Kokoro/VoiceVox for privacy/free
5. **Handle interruption** — if the user starts speaking, stop the AI's current audio

---

## 7. Lesson 5: 3D/2D Avatar Rendering

### The Problem
A disembodied voice is less engaging than a visual character you can see react in real-time.

### AIRI's Solution: Dual Rendering (VRM 3D + Live2D 2D)

#### VRM 3D Avatars (Three.js)

Package: `packages/stage-ui-three/`

**Loading pipeline:**
1. Load `.vrm` file with progress tracking
2. Optimize model (vertex reduction, skeleton combining)
3. Compute bounding box for camera positioning
4. Set up look-at quaternion proxy (eye tracking)
5. Enable spring bone physics (hair/cloth movement)

**Automatic behaviors:**
- **Auto-blink** — natural blinking at random intervals
- **Auto look-at** — eyes follow cursor/camera
- **Idle saccades** — subtle eye movements when "thinking"
- **Spring bone physics** — hair and accessories bounce naturally
- **Lip-sync** — real-time mouth movement from audio

**Lip-sync system** (`packages/stage-ui-three/src/composables/vrm/lip-sync.ts`):
- Uses `wLipSync` library for phoneme detection
- Maps phonemes to VRM blendshapes: A→aa, E→ee, I→ih, O→oh, U→ou
- Winner + runner-up blending (top 2 shapes for natural transitions)
- Attack/release smoothing
- Volume-aware amplitude
- Silence detection (mouth closes smoothly when quiet)

#### Live2D 2D Avatars

Package: `packages/stage-ui-live2d/`

**Features:**
- Pixi.js + pixi-live2d-display
- ZIP model loading (bundled assets)
- OPFS (Origin Private File System) storage
- Parameter-based eye control (ParamEyeBallX/Y)
- Motion manager for animations
- Automatic blinking + idle eye focus

#### Model Store

`packages/stage-ui-three/src/stores/model-store.ts`:
- Centralized model state
- Environment assets (HDRI sky, IBL lighting)
- Render target management

### Takeaway for AI GF
**VRM is the best choice for a 3D AI girlfriend:**
1. Free, open format with thousands of existing models
2. Built-in blendshape support for emotions and lip-sync
3. Pixiv ecosystem (VRoid Studio lets users create custom models for free)
4. Spring bone physics for natural movement
5. Look-at system for engaging eye contact

**Live2D is the best choice if you prefer 2D anime style.**

The auto-behaviors (blink, saccade, look-at) are critical — they make the avatar feel alive even when idle.

---

## 8. Lesson 6: Memory & Context Management

### The Problem
An AI GF that forgets everything after each conversation is useless. She needs to remember the user, their preferences, past conversations, and evolving relationship dynamics.

### AIRI's Solution: Multi-Layer Memory

#### Layer 1: Conversation History (Short-Term)

- Standard message array passed to LLM context window
- Context limits: auto-trim when exceeding thresholds (e.g., keep latest 5 of 20 messages)
- System notification injected when context is trimmed

#### Layer 2: Vector Embeddings (Long-Term)

Package: `packages/memory-pgvector/`

- PostgreSQL with pgvector extension
- Drizzle ORM for schema management
- Embeddings generated via `@xsai/embed`
- HNSW indexes for fast semantic similarity search

#### Layer 3: Structured Memory (Telegram Bot)

The Telegram bot (`services/telegram-bot/src/db/schema.ts`) demonstrates a sophisticated memory schema:

```
memoryFragmentsTable     — Episodic, short-term, long-term memories with vectors
memoryTagsTable          — Tags for memory organization
memoryEpisodicTable      — Specific events (conversations, introductions, etc.)
memoryLongTermGoalsTable — User goals with hierarchical support
memoryShortTermIdeas     — Ideas from "dreams" and reflections
```

Each memory fragment has:
- Content text
- Vector embedding (768/1024/1536 dimensions)
- Memory type classification
- Timestamp + soft delete
- Tag associations

#### Layer 4: Context Summarization (Minecraft Agent)

The Minecraft service shows advanced context management:
- Active context tracking
- Automatic context switching
- Context summarization (compress old context into summaries)
- History query system for retrieving past decisions

#### Browser-Side Storage

- **DuckDB WASM** (`packages/duckdb-wasm/`) — Full SQL database running in the browser
- **Drizzle ORM integration** — Type-safe queries
- No external database required for basic usage

### Takeaway for AI GF
**Memory is what transforms a chatbot into a companion.** Build these layers:

1. **Conversation buffer** — Last N messages for immediate context
2. **Embedding store** — Vector DB for semantic recall ("remember when we talked about...")
3. **Structured facts** — Key-value store for user facts (name, birthday, preferences)
4. **Episodic memory** — Timestamped events (first meeting, milestones, arguments)
5. **Relationship state** — Evolving metrics (closeness, trust, mood history)
6. **Context summarization** — Compress old conversations into summaries to fit context windows

DuckDB WASM is a brilliant choice for browser-only deployments — full SQL without a server.

---

## 9. Lesson 7: Cognitive Architecture (Agent Loops)

### The Problem
A simple request-response chatbot can't take initiative, plan actions, or handle complex multi-step interactions.

### AIRI's Solution: Multi-Layer Cognitive Architecture

The Minecraft agent (`services/minecraft/`) demonstrates the most sophisticated approach:

```
┌─────────────────────────────────────────┐
│          Layer D: Action                 │
│  Task execution with action registry    │
├─────────────────────────────────────────┤
│          Layer C: Conscious             │
│  LLM-powered reasoning (brain.ts)      │
│  Context summarization + archival       │
│  History queries + planning             │
├─────────────────────────────────────────┤
│          Layer B: Reflex                │
│  Fast FSM-based reactions               │
│  No LLM calls needed                    │
├─────────────────────────────────────────┤
│          Layer A: Perception            │
│  Event definitions + rule evaluation    │
│  YAML-defined rules                     │
└─────────────────────────────────────────┘
```

#### Telegram Bot Agent Loop

The Telegram bot shows a practical agent pattern:

1. **Queue** — Messages arrive and are queued
2. **Interpret** — Media is analyzed (stickers → emoji, photos → descriptions)
3. **Record** — Messages are saved to DB with embeddings
4. **Action Loop** — LLM generates actions, which are dispatched recursively:

```
imagineAnAction() → dispatchAction() → next action → dispatchAction() → ...
```

**Action types:**
- `send_message` — Reply to user
- `send_sticker` — React with sticker
- `read_unread_messages` — Check other conversations
- `list_chats` — Discover available conversations
- `sleep` — Wait and check back later
- `continue` / `break` — Loop control

**Key patterns:**
- **AbortController** for interruption (new message cancels current processing)
- **Context limits** with auto-trim (max 20 messages, max 50 actions)
- **Periodic ticking** (runs every 60s even without new messages — "thinking")
- **Message consolidation** (merges duplicate read requests)

### Takeaway for AI GF
**Go beyond request-response.** An engaging AI GF should:
1. **Take initiative** — Send "good morning" messages, comment on things proactively
2. **Handle interruptions** — If you send a new message while she's "thinking", she should adapt
3. **Have reflexes** — Some responses should be instant (greetings, reactions) without waiting for LLM
4. **Plan multi-step actions** — "Let me find that song we talked about" → search → share
5. **Tick periodically** — Check in even without user input ("Haven't heard from you in a while!")

---

## 10. Lesson 8: Plugin Architecture for Extensibility

### The Problem
Users want to customize their AI GF with new capabilities (smart home control, music, games, social media).

### AIRI's Solution: Full Plugin SDK

Package: `packages/plugin-sdk/` + `packages/plugin-protocol/`

**Plugin lifecycle (XState-based):**
```
loading → loaded → authenticating → authenticated → announced → preparing
→ prepared → configuration-needed → configured → ready
```

**Plugin manifest (V1):**
```typescript
{
  apiVersion: 'v1',
  kind: 'manifest.plugin.airi.moeru.ai',
  name: string,
  entrypoints: {
    default?: string,
    electron?: string,
    node?: string,
    web?: string
  }
}
```

**Two-channel architecture:**
- `channels.host` — Control plane (plugin ↔ host)
- `channels.data` — Data plane (plugin ↔ plugin)

**Existing plugins demonstrate the power:**
- `airi-plugin-bilibili-laplace` — Live streaming chat integration
- `airi-plugin-homeassistant` — Smart home control ("turn off the lights, babe")
- `airi-plugin-web-extension` — Browser integration
- `airi-plugin-claude-code` — AI coding assistant

**Transport backends:**
- In-memory (implemented)
- WebSocket (defined, partially implemented)
- Web Workers (defined)
- Electron IPC (defined)

### Takeaway for AI GF
**Build a plugin system early.** Even a simple one enables:
- Smart home integration (she controls your lights/music)
- Calendar/reminder integration (she knows your schedule)
- Music player (she picks songs for you)
- Game integrations (play together)
- Social media bridges (chat on Discord/Telegram)

---

## 11. Lesson 9: Multi-Platform Deployment

### AIRI deploys to 3 platforms from one codebase:

| Platform | App | Technology |
|---|---|---|
| **Web** | `apps/stage-web` | Vue 3 + Vite, PWA-enabled |
| **Desktop** | `apps/stage-tamagotchi` | Electron + Vue |
| **Mobile** | `apps/stage-pocket` | Capacitor (iOS/Android) |

**Shared code strategy:**
- `packages/stage-ui/` — Core business logic (stores, composables, services)
- `packages/stage-shared/` — Cross-app utilities
- `packages/stage-pages/` — Shared page components
- `packages/stage-layouts/` — Layout components
- Platform-specific code isolated in each app

### Takeaway for AI GF
**Start with web, expand later.** The shared package strategy is excellent:
1. Put all business logic in shared packages
2. Keep platform-specific code minimal (just shell/wrapper)
3. Web gives you the widest reach immediately
4. Desktop (Electron) adds always-on-screen presence
5. Mobile (Capacitor) enables on-the-go interaction

---

## 12. Lesson 10: Real-Time Communication Protocol

### AIRI's Server Runtime: A Message Broker

Package: `packages/server-runtime/`

**Protocol features:**
- WebSocket-based real-time messaging
- Authentication with token verification
- Peer registry with heartbeat monitoring
- Policy-based routing (allow/deny by plugin/label)
- Destination expressions (glob, module ID, instance ID, label selectors)
- SuperJSON serialization (handles complex types)

**Event types covering the full interaction:**

```
Input Events:
  input:text          — Text messages
  input:text:voice    — Transcribed voice with confidence
  input:voice         — Raw audio buffers

Output Events:
  output:gen-ai:chat:tool-call  — Function/tool calls
  output:gen-ai:chat:message    — Assistant responses
  output:gen-ai:chat:complete   — Full completion with usage stats

Inter-Agent (Spark) Events:
  spark:notify   — Alerts with urgency levels
  spark:command  — Instructions to sub-agents
  spark:emit     — Progress/completion acknowledgements
```

### Takeaway for AI GF
**Design your event protocol early.** Even for a simple app, having typed events like `input:text`, `output:speech`, `emotion:change`, `memory:recall` makes the system composable and debuggable.

---

## 13. Key Technical Decisions Worth Adopting

### 1. xSAI for LLM abstraction
Don't build your own — use or emulate the xSAI pattern of unified provider configs.

### 2. VRM for 3D avatars
Open format, free tools (VRoid Studio), massive community, built-in blendshapes.

### 3. DuckDB WASM for client-side storage
Full SQL in the browser. No server needed for a basic AI GF that runs entirely client-side.

### 4. Web Audio API + VAD for voice
Critical for natural voice interaction. Silero VAD via ONNX Runtime in Web Workers is the gold standard.

### 5. Pinia for state management
Clean, type-safe stores. The separation into domain stores (LLM, character, speech, hearing) is excellent.

### 6. Streaming TTS with chunking
Don't wait for the full response. The `Intl.Segmenter`-based chunking strategy is production-quality.

### 7. Vue 3 + Vite + TypeScript
Fast dev experience, excellent ecosystem, good for UI-heavy apps.

### 8. Monorepo with pnpm workspaces
Keeps code modular and testable. Even for a smaller project, separating `core`, `ui`, `voice`, `memory` packages is worth it.

---

## 14. What to Avoid / Improve Upon

### 1. Over-engineering from the start
AIRI is a large open-source project with many contributors. A personal AI GF project doesn't need 39 packages, 5 services, and a full plugin SDK on day one. Start simple.

### 2. Incomplete transport implementations
Only in-memory transport is fully implemented. WebSocket and Worker transports throw "not implemented". Pick one transport and complete it.

### 3. Memory system still WIP
The "Memory Alaya" system is marked WIP. For a custom AI GF, prioritize memory over everything else — it's what creates emotional attachment.

### 4. Limited test coverage
The repo has testing infrastructure but sparse tests. For a personal project, focus tests on the memory and personality systems.

### 5. Complexity of the cognitive architecture
The four-layer Minecraft architecture is impressive but overkill for chat. Start with a simple agent loop (like the Telegram bot pattern) and add layers as needed.

---

## 15. Recommended Minimal Architecture for a Custom AI GF

Based on everything learned from AIRI, here's a minimal but complete architecture:

```
┌─────────────────────────────────────────────────┐
│                  Web Frontend                    │
│                                                  │
│  ┌──────────┐  ┌───────────┐  ┌──────────────┐ │
│  │ Chat UI  │  │ VRM Avatar│  │ Voice Toggle │ │
│  │          │  │ (Three.js)│  │ (Mic/Speaker)│ │
│  └─────┬────┘  └─────┬─────┘  └──────┬───────┘ │
│        │              │               │         │
│  ┌─────▼──────────────▼───────────────▼───────┐ │
│  │            State Management (Pinia)         │ │
│  │                                             │ │
│  │  Character │ Chat │ Memory │ Voice │ Emotion│ │
│  └─────┬──────┴──┬───┴────┬───┴───┬───┴───┬───┘ │
│        │         │        │       │       │     │
│  ┌─────▼─────┐ ┌─▼────┐ ┌▼────┐ ┌▼────┐ ┌▼──┐ │
│  │ Character │ │ LLM  │ │DuckDB│ │ TTS │ │VRM│ │
│  │ Card (CCC)│ │(xSAI)│ │ WASM│ │/STT │ │Exp│ │
│  └───────────┘ └──────┘ └─────┘ └─────┘ └───┘ │
└─────────────────────────────────────────────────┘
```

### MVP Features (Priority Order):
1. **Character card system** — Define her personality
2. **LLM integration** — Claude/GPT via xSAI pattern
3. **Chat UI** — Basic text conversation
4. **Memory** — DuckDB WASM for conversation history + facts
5. **VRM avatar** — Visual presence with auto-behaviors
6. **Emotion system** — LLM tags → avatar expressions
7. **TTS** — She speaks (ElevenLabs or Kokoro)
8. **STT + VAD** — You speak to her
9. **Lip-sync** — Mouth moves with speech
10. **Proactive behavior** — Periodic check-ins, initiative

---

## 16. Key Files Reference

| What | File Path |
|------|-----------|
| Project overview | `README.md` |
| Developer guide | `AGENTS.md` |
| LLM provider abstraction | `packages/stage-ui/src/libs/providers/` |
| LLM streaming store | `packages/stage-ui/src/stores/llm.ts` |
| Character store | `packages/stage-ui/src/stores/character/index.ts` |
| Character card management | `packages/stage-ui/src/stores/modules/airi-card.ts` |
| Character card creator | `packages/ccc/src/` |
| System prompt with emotions | `packages/stage-ui/src/constants/prompts/system-v2.ts` |
| VRM expression/emotion system | `packages/stage-ui-three/src/composables/vrm/expression.ts` |
| VRM lip-sync | `packages/stage-ui-three/src/composables/vrm/lip-sync.ts` |
| VRM loader | `packages/stage-ui-three/src/composables/vrm/loader.ts` |
| Live2D animation | `packages/stage-ui-live2d/src/composables/live2d/animation.ts` |
| TTS store | `packages/stage-ui/src/stores/modules/speech.ts` |
| STT store | `packages/stage-ui/src/stores/modules/hearing.ts` |
| TTS chunking strategy | `packages/stage-ui/src/utils/tts.ts` |
| Audio pipeline runtime | `packages/stage-ui/src/services/speech/pipeline-runtime.ts` |
| VAD worker | `packages/stage-ui/src/workers/vad/` |
| Memory (pgvector) | `packages/memory-pgvector/src/` |
| DuckDB WASM | `packages/duckdb-wasm/src/` |
| Plugin SDK | `packages/plugin-sdk/src/` |
| Plugin protocol events | `packages/plugin-protocol/src/types/events.ts` |
| Server runtime (message broker) | `packages/server-runtime/src/index.ts` |
| Server SDK (WebSocket client) | `packages/server-sdk/src/client.ts` |
| Telegram bot personality prompt | `services/telegram-bot/src/prompts/personality-v1.velin.md` |
| Telegram bot agent loop | `services/telegram-bot/src/bots/telegram/index.ts` |
| Telegram bot memory schema | `services/telegram-bot/src/db/schema.ts` |
| Minecraft cognitive architecture | `services/minecraft/src/cognitive/` |
| Discord bot adapter | `services/discord-bot/src/adapters/airi-adapter.ts` |
| Discord voice management | `services/discord-bot/src/commands/summon.ts` |

---

*Analysis generated from Project AIRI v0.8.5-beta.3 (https://github.com/moeru-ai/airi)*
