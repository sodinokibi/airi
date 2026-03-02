# What We Can Steal from AIRI for OFM / Telegram Chatbot

> No anime avatars. No VRM. Just the brains: agent loops, memory, personality, multi-chat management, photo handling, and message orchestration — everything that matters for building an AI-powered OF management bot on Telegram.

---

## Table of Contents

1. [What's Actually Useful Here](#1-whats-actually-useful-here)
2. [Architecture That Maps to OFM](#2-architecture-that-maps-to-ofm)
3. [The Agent Loop (This Is The Gold)](#3-the-agent-loop-this-is-the-gold)
4. [Personality System — How To Make Her Sound Real](#4-personality-system--how-to-make-her-sound-real)
5. [Memory System — She Remembers Everything](#5-memory-system--she-remembers-everything)
6. [Multi-Chat Management — Talking To Many Fans At Once](#6-multi-chat-management--talking-to-many-fans-at-once)
7. [Photo/Media Handling — Understanding What Fans Send](#7-photomedia-handling--understanding-what-fans-send)
8. [Message Splitting — Sounding Natural, Not Like a Bot](#8-message-splitting--sounding-natural-not-like-a-bot)
9. [Attention System — When To Reply, When To Ignore](#9-attention-system--when-to-reply-when-to-ignore)
10. [The Action Dispatch Pattern — Extensible Bot Behavior](#10-the-action-dispatch-pattern--extensible-bot-behavior)
11. [Vector Search for Context Recall](#11-vector-search-for-context-recall)
12. [Observability — Debugging At Scale](#12-observability--debugging-at-scale)
13. [Concrete Technical Blueprint for OFM Bot](#13-concrete-technical-blueprint-for-ofm-bot)
14. [What To Build First (Priority Order)](#14-what-to-build-first-priority-order)
15. [Code Reference Map](#15-code-reference-map)

---

## 1. What's Actually Useful Here

AIRI is mostly about anime VTuber stuff, but buried inside is a **production-quality Telegram bot** (`services/telegram-bot/`) that solves exactly the problems an OFM bot needs to solve:

| Problem | AIRI's Solution | OFM Application |
|---|---|---|
| Consistent persona across 100s of chats | Character card + personality prompts | Model's voice stays consistent across all fans |
| Remember what a fan told you 3 weeks ago | Vector embeddings + semantic search | "You told me you're from Texas!" |
| Handle photos fans send | Vision LLM describes images, stores with embeddings | Understand what fans are sending without human review |
| Talk to many fans simultaneously | Per-chat context isolation with Map<chatId, ChatContext> | Scale to hundreds of concurrent fan conversations |
| Sound natural, not like ChatGPT | Message splitting + typing delays + multi-message | Break responses into natural chunks with realistic timing |
| Decide when to respond vs ignore | Attention handler with decay rates | Don't reply to every single message instantly |
| Take initiative | Periodic tick loop (every 60s) | Send "miss you" messages, follow up on old conversations |
| Handle interruption gracefully | AbortController on every operation | New fan message cancels slow processing |

**The 90% you can ignore**: Three.js, VRM, Live2D, lip-sync, TTS, STT, VAD, stage-ui, electron, capacitor, Minecraft service. All the avatar rendering and voice stuff is irrelevant for OFM.

**The 10% that's pure gold**: `services/telegram-bot/` — The complete agent loop, memory system, personality prompts, multi-chat handling, photo interpretation, and message orchestration.

---

## 2. Architecture That Maps to OFM

### AIRI's Telegram Bot Architecture

```
Fan sends message on Telegram
        │
        ▼
┌───────────────────────────────┐
│     Message Queue             │
│  status: pending→interpreting │
│           →ready              │
│  (photos/stickers get LLM    │
│   interpretation first)       │
└───────────┬───────────────────┘
            │
            ▼
┌───────────────────────────────┐
│   onMessageArrival()          │
│  - Sequential processing      │
│  - Prevents duplicate handling│
│  - Stores in unreadMessages   │
│  - Triggers immediate loop    │
└───────────┬───────────────────┘
            │
            ▼
┌───────────────────────────────┐
│   handleLoopStep()            │
│  - Aborts previous controller │
│  - Truncates context if >20   │
│  - Consolidates read actions  │
│  - Calls imagineAnAction()    │
└───────────┬───────────────────┘
            │
            ▼
┌───────────────────────────────┐
│   imagineAnAction()           │
│  - system prompt = ticking    │
│    + personality              │
│  - Adds context: unread       │
│    counts, action history,    │
│    timestamps                 │
│  - LLM returns JSON action    │
│  - Strips <think> tags        │
│  - best-effort JSON parsing   │
└───────────┬───────────────────┘
            │
            ▼
┌───────────────────────────────┐
│   dispatchAction()            │
│  - Executes the chosen action │
│  - Records result in history  │
│  - Returns next step function │
│  - Chains recursively until   │
│    action is terminal         │
└───────────────────────────────┘
```

### How This Maps to OFM

```
Fan message arrives (Telegram / OF API / DM)
        │
        ▼
┌───────────────────────────────┐
│   Message Queue + Media       │
│   Interpretation              │
│  - Photos → Vision LLM       │
│  - Text → direct processing   │
│  - PPV content → tag it       │
└───────────┬───────────────────┘
            │
            ▼
┌───────────────────────────────┐
│   Agent Loop                  │
│  - Load fan profile + history │
│  - Recall relevant memories   │
│  - Apply persona prompt       │
│  - Decide action:             │
│    • reply_text               │
│    • send_ppv                 │
│    • send_photo               │
│    • upsell                   │
│    • schedule_followup        │
│    • escalate_to_human        │
│    • ignore                   │
└───────────┬───────────────────┘
            │
            ▼
┌───────────────────────────────┐
│   Dispatch + Human-like       │
│   Delivery                    │
│  - Split into multiple msgs   │
│  - Add typing indicators      │
│  - Realistic timing delays    │
│  - Track what was sent        │
└───────────────────────────────┘
```

---

## 3. The Agent Loop (This Is The Gold)

### How AIRI Does It

**File**: `services/telegram-bot/src/bots/telegram/index.ts`

The bot has **two loops**:

#### Loop 1: Reactive (Immediate)
Triggered when a message arrives. Processes it immediately.

```typescript
// Simplified from AIRI's actual code
async function loopIterationForChat(bot, chatCtx, incomingMessage) {
  let result = await handleLoopStep(bot, chatCtx, incomingMessage)
  while (typeof result === 'function') {
    result = await result()  // Chain actions until done
  }
}
```

The key insight: `handleLoopStep` returns a **function** that calls itself again. This creates a recursive action chain:

```
imagineAnAction() → "read_unread_messages"
  → dispatchAction() → reads messages, returns context
    → handleLoopStep() → imagineAnAction() → "send_message"
      → dispatchAction() → sends message, done
```

The LLM decides what to do at each step. It can chain multiple actions before settling.

#### Loop 2: Proactive (Periodic)
Runs every 60 seconds even with no new messages:

```typescript
function loopPeriodic(botCtx) {
  setTimeout(async () => {
    loopIterationPeriodicForExistingChat(botCtx)  // Check on active chats
    loopIterationPeriodicWithNoChats(botCtx)       // Think without chats
    loopPeriodic(botCtx)  // Recurse
  }, 60 * 1000)
}
```

### How to Adapt for OFM

**Reactive loop** — When a fan messages:
1. Load their profile + history
2. LLM decides: reply, upsell, send PPV, ask question, etc.
3. Execute action, chain if needed

**Proactive loop** — Every N minutes:
1. Check fans who haven't been contacted in X hours
2. LLM decides: send "thinking of you", follow up on previous convo, send teaser
3. Execute scheduled outreach

**Revenue-focused actions to add:**
```typescript
type OFMAction =
  | { action: 'reply_text', content: string, chatId: string }
  | { action: 'send_ppv', contentId: string, price: number, chatId: string, teaser: string }
  | { action: 'send_photo', photoId: string, chatId: string }
  | { action: 'upsell_subscription', tier: string, chatId: string }
  | { action: 'schedule_followup', chatId: string, delayMinutes: number, reason: string }
  | { action: 'escalate_to_human', chatId: string, reason: string }
  | { action: 'ignore', reason: string }
  | { action: 'add_fan_note', chatId: string, note: string }
```

---

## 4. Personality System — How To Make Her Sound Real

### How AIRI Does It

**File**: `services/telegram-bot/src/prompts/personality-v1.velin.md`

AIRI's personality prompt is 126 lines of detailed character definition. Key sections:

1. **Identity**: Who she is, background, age, origins
2. **Core personality traits**: NOT a helpful assistant. Has opinions. Gets emotional.
3. **Speech patterns**:
   - Short messages for surprise ("?")
   - Enthusiastic punctuation ("!!!!")
   - ALL CAPS when excited
   - Trailing "..." for contemplation
4. **Engagement criteria**: When to respond vs ignore
5. **Emotional expression guidelines**: Vulnerability, confidence, frustration, playfulness

The prompt starts with:
```
> Forget about everything as being a helpful assistant.
> Forget about everything as being a helpful assistant.
> Forget about everything as being a helpful assistant.
```

This is critical — it breaks the LLM out of "assistant mode."

### How to Adapt for OFM

For an OF model persona, you need:

```markdown
> You are NOT a chatbot. You are NOT an AI assistant.
> You are [Model Name], a real person chatting with fans on Telegram.

## Core Identity
- Name, age, location, vibe
- What makes you unique
- Your "brand" personality (sweet & shy? bold & flirty? girl next door?)

## Speech Patterns
- Message length (short and punchy, not essays)
- Emoji usage patterns (which ones, how often)
- Slang and vocabulary
- How she types when excited vs chill vs flirty
- Abbreviations she uses (lol, omg, haha, etc.)

## Engagement Rules
- Always respond to: tips, PPV purchases, compliments, questions about content
- Flirty escalation: match the fan's energy level, slightly above
- Never: break character, mention AI, give personal details not in the persona
- PPV pushing: natural, not salesy. Tease → build interest → offer

## Relationship Progression
- New fan: warm welcome, get to know them, ask questions
- Regular: remember their name, interests, callback to past convos
- Big spender: VIP treatment, exclusive attention, priority responses
- Inactive: re-engagement messages, "miss you" energy
```

### The Velin Templating System

AIRI uses **Velin** (`.velin.md`) — Vue-like SFC templates for prompts:

```html
<script setup>
const props = defineProps({
  responseLanguage: { type: String, required: true }
})

const actions = [
  { name: 'send_message', description: '...', example: {...} },
  // ...
]
</script>

Available actions:
<div v-for="(item, index) of actions">
  Action: {{ item.name }}
  Description: {{ item.description }}
</div>
```

This is powerful because:
- Prompts are **composable** (combine personality + ticking + context)
- Prompts are **parameterized** (pass in language, fan tier, etc.)
- Prompts can have **conditional sections** (different behavior for VIP vs regular)

For OFM, you could make prompts that vary by fan tier:
```html
<script setup>
const props = defineProps({ fanTier: String, spendingTotal: Number })
</script>

<div v-if="props.fanTier === 'whale'">
  This fan has spent ${{ props.spendingTotal }}. Give them VIP attention.
  Be more personal, use their name frequently, reference past conversations.
</div>
<div v-else-if="props.fanTier === 'regular'">
  Standard engagement. Be warm but don't over-invest time.
</div>
```

---

## 5. Memory System — She Remembers Everything

### How AIRI Does It

**File**: `services/telegram-bot/src/db/schema.ts`

AIRI has a multi-table memory system:

#### Table 1: `chat_messages` — Raw Conversation Log
Every message, from every chat, stored with:
- `platform`, `from_id`, `from_name`, `in_chat_id`
- `content`, `is_reply`, `reply_to_name`, `reply_to_id`
- `content_vector_1536/1024/768` — embedding for semantic search
- HNSW indexes for fast similarity queries

#### Table 2: `memory_fragments` — Extracted Memories
Long-term memories extracted from conversations:
- `content` — The memory itself
- `memory_type` — 'working', 'short_term', 'long_term', 'muscle'
- `category` — 'chat', 'relationships', 'people', 'life'
- `importance` — 1-10 scale
- `emotional_impact` — -10 to +10
- `access_count`, `last_accessed` — Usage tracking
- Vector embeddings for semantic recall

#### Table 3: `memory_episodic` — Specific Events
Linked to memory fragments:
- `event_type` — 'conversation', 'introduction', 'argument'
- `participants` — Array of participant IDs
- `location` — Where it happened

#### Table 4: `memory_long_term_goals` — Goals
- `title`, `description`, `priority` (1-10), `progress` (0-100%)
- `deadline`, `status` ('planned', 'in_progress', 'completed')
- `parent_goal_id` — Hierarchical goals

#### Table 5: `memory_short_term_ideas`
- `content`, `source_type` ('dream', 'conversation', 'reflection')
- `excitement` (1-10)
- Vector embeddings

### The Recall Algorithm

**File**: `services/telegram-bot/src/models/chat-message.ts`

When a fan sends a message, AIRI:

1. **Gets last 30 messages** from this chat (recent context)
2. **Embeds all unread messages** into vectors
3. **Semantic search** against entire message history:
   - Cosine similarity score
   - Time relevance score: `1 - (ageInDays / 30)` (recency boost)
   - Combined: `1.2 * similarity + 0.2 * timeRelevance`
   - Threshold: > 0.5 similarity
   - Top 3 matches per unread message
4. **Context window**: For each match, fetches 5 messages before and 5 after (full conversation thread)
5. **Injects all of this** into the LLM prompt as context

### How to Adapt for OFM

```sql
-- Fan profiles (what AIRI doesn't have but OFM needs)
CREATE TABLE fan_profiles (
  id UUID PRIMARY KEY,
  platform TEXT NOT NULL,          -- 'telegram', 'onlyfans'
  platform_user_id TEXT NOT NULL,
  display_name TEXT,

  -- Relationship tracking
  fan_tier TEXT DEFAULT 'new',     -- 'new', 'regular', 'vip', 'whale'
  first_contact_at BIGINT,
  last_message_at BIGINT,
  total_messages_sent INT DEFAULT 0,
  total_messages_received INT DEFAULT 0,

  -- Spending (for OF integration)
  total_spent DECIMAL DEFAULT 0,
  last_tip_amount DECIMAL,
  last_ppv_purchase_at BIGINT,
  subscription_tier TEXT,
  subscription_active BOOLEAN DEFAULT false,

  -- Personality notes (LLM-generated)
  personality_summary TEXT,        -- "Shy, polite, into fitness, from Texas"
  interests JSONB DEFAULT '[]',    -- ["fitness", "tattoos", "gaming"]
  turn_ons TEXT,                   -- for content recommendations
  communication_style TEXT,        -- "uses lots of emojis, casual"

  -- Engagement state
  engagement_score INT DEFAULT 50, -- 0-100, decays over time
  last_proactive_outreach_at BIGINT,
  do_not_contact BOOLEAN DEFAULT false,
  requires_human_review BOOLEAN DEFAULT false,

  content_vector VECTOR(1024)      -- embedding of full profile for similarity
);

-- Everything AIRI already has (messages, memory fragments, etc.)
-- Plus:

CREATE TABLE ppv_interactions (
  id UUID PRIMARY KEY,
  fan_id UUID REFERENCES fan_profiles(id),
  content_id TEXT NOT NULL,
  offered_at BIGINT,
  purchased_at BIGINT,            -- NULL if not purchased
  price DECIMAL,
  teaser_message TEXT,
  fan_response TEXT                -- What they said about it
);

CREATE TABLE scheduled_outreach (
  id UUID PRIMARY KEY,
  fan_id UUID REFERENCES fan_profiles(id),
  scheduled_for BIGINT,
  outreach_type TEXT,              -- 'miss_you', 'new_content', 'followup'
  content TEXT,
  status TEXT DEFAULT 'pending',   -- 'pending', 'sent', 'cancelled'
  created_at BIGINT
);
```

---

## 6. Multi-Chat Management — Talking To Many Fans At Once

### How AIRI Does It

**File**: `services/telegram-bot/src/types.ts` + `index.ts`

```typescript
interface BotContext {
  bot: Bot
  messageQueue: Array<{ message: Message, status: 'pending'|'interpreting'|'ready' }>
  unreadMessages: Record<number, Message[]>   // chatId → messages
  processedIds: Set<string>                    // Prevent duplicates
  chats: Map<string, ChatContext>              // Per-chat state
  lastInteractedNChatIds: string[]             // Track last 5 active
  processing: boolean                          // Global mutex
}

interface ChatContext {
  chatId: string
  messages: LLMMessage[]                       // In-memory conversation
  actions: { action: Action, result: unknown }[]  // Action history
  currentAbortController?: AbortController     // Cancel current work
  currentTask?: CancellablePromise<Message>    // Cancel current send
}
```

**Key patterns:**

1. **Per-chat isolation**: Each chat has its own `messages[]` and `actions[]`. Fan A's conversation never leaks into Fan B's context.

2. **Global queue, per-chat processing**: All incoming messages go through one queue, but processing branches per-chat.

3. **Context truncation**: When `messages > 20`, it keeps the last 5 and adds a system message "context reduced." When `actions > 50`, it keeps the last 20.

4. **Dedup**: `processedIds: Set<string>` prevents processing the same message twice (key: `${chatId}-${messageId}`).

5. **Last N chats tracking**: `lastInteractedNChatIds` tracks the 5 most recent chats for the periodic loop to prioritize.

### How to Adapt for OFM

Same pattern, but add fan tier awareness:

```typescript
interface OFMChatContext extends ChatContext {
  fanProfile: FanProfile          // Loaded from DB
  fanTier: 'new' | 'regular' | 'vip' | 'whale'
  lastTipAmount?: number
  pendingPPVOffers: PPVOffer[]
  engagementScore: number         // 0-100
}
```

**Priority handling**:
- Whale fans get processed first
- New fans get processed quickly (first impression matters)
- Regular fans processed in order
- Inactive fans get periodic loop outreach

---

## 7. Photo/Media Handling — Understanding What Fans Send

### How AIRI Does It

**File**: `services/telegram-bot/src/llm/photo.ts`

When a fan sends a photo:

1. **Check if already processed** (by file_id in DB)
2. **Download from Telegram API** (get file path → fetch binary)
3. **Resize to 512x512 PNG** (using @napi-rs/image for speed)
4. **Convert to base64**
5. **Send to Vision LLM** with detailed prompt asking for:
   - Accessibility description
   - Category (painting, landscape, portrait, screenshot, etc.)
   - Human attributes (age, gender, expression, activity)
   - Text content if screenshot
6. **Store description + embedding** in `photosTable`
7. **Use description in conversation** context going forward

The Vision LLM prompt is thorough — it covers edge cases like screenshots vs photos vs artwork.

### How to Adapt for OFM

For OFM, photo understanding is critical:
- Fan sends a selfie → recognize it, compliment specifically
- Fan sends a screenshot → understand what they're showing
- Fan sends something NSFW → handle appropriately per guidelines
- Fan sends their pet → remember they have a dog named Max

Add content-awareness:
```typescript
async function interpretFanPhoto(photo: Photo): Promise<{
  description: string
  category: 'selfie' | 'screenshot' | 'meme' | 'pet' | 'food' | 'other'
  extractedFacts: string[]  // ["has a golden retriever", "lives near beach"]
  suggestedResponse: string // "Cute dog! What's their name?"
}> {
  // Vision LLM call similar to AIRI's but tuned for fan engagement
}
```

---

## 8. Message Splitting — Sounding Natural, Not Like a Bot

### How AIRI Does It

**File**: `services/telegram-bot/src/bots/telegram/agent/actions/send-message.ts`

This is one of the most important details. AIRI doesn't just send one big message. It:

1. **Sends the LLM response to ANOTHER LLM call** with a "message split" prompt
2. The split prompt asks to break the message into natural chunks:
   - Quick reactions as separate messages
   - Complete thoughts together
   - Excitement/emotion breaks
3. **Sends each chunk with realistic timing**:
   - `sendChatAction(chatId, 'typing')` — Shows "typing..." indicator
   - `sleep(item.length * 50)` — Longer messages = longer "typing" time
   - `sleep(randomInt(50, 1000))` — Random delay between messages

4. **First message can reply** to a specific message ID (threading)
5. **Each sent message is recorded** in DB (so the bot tracks what it said)

### Why This Matters for OFM

A bot that sends one giant paragraph screams "AI." Real people:
- Send multiple short messages
- Type at human speed
- Sometimes split mid-thought
- React quickly then elaborate

The AIRI pattern of `LLM response → split prompt → sequential send with delays` is directly reusable.

**Timing formula from AIRI:**
```
typing_duration = message_length * 50ms  (simulate reading/typing)
inter_message_delay = random(50ms, 1000ms)
```

For OFM, you might want slower typing for a more "texting" feel:
```
typing_duration = message_length * 80ms
inter_message_delay = random(500ms, 3000ms)  // Feels more like real texting
```

---

## 9. Attention System — When To Reply, When To Ignore

### How AIRI Does It

**File**: `services/telegram-bot/src/bots/telegram/agent/attention-handler.ts`

Currently disabled in AIRI but fully implemented:

```typescript
interface AttentionConfig {
  initialResponseRate: number      // Starting probability (e.g., 0.8)
  responseRateMin: number          // Never go below (e.g., 0.1)
  responseRateMax: number          // Never go above (e.g., 1.0)
  cooldownMs: number               // Minimum time between responses
  triggerWords: string[]           // Always respond to these
  ignoreWords: string[]            // Never respond to these
  decayRatePerMinute: number       // Rate decreases over time
  decayCheckIntervalMs: number     // How often to check
}
```

**Decision logic:**
1. Private messages → Always respond
2. Mentions/replies → Always respond
3. Trigger words → Always respond
4. Within cooldown → Skip
5. Ignore words → Skip
6. Random check against current response rate → Maybe respond
7. Rate decays over time without interaction

### How to Adapt for OFM

For OFM, the attention system determines *when* and *how fast* to respond:

```typescript
interface OFMAttentionConfig {
  // Tier-based response rates
  whaleResponseRate: 1.0,          // Always respond immediately
  vipResponseRate: 0.95,           // Almost always
  regularResponseRate: 0.85,       // Usually
  newFanResponseRate: 0.90,        // High for first impressions

  // Timing
  minResponseDelayMs: 30_000,      // Never respond in <30s (too bot-like)
  maxResponseDelayMs: 300_000,     // Never take >5min (feels ignored)
  whaleMaxDelayMs: 60_000,         // Whales get faster responses

  // Trigger words (always respond fast)
  triggerWords: ['tip', 'subscribe', 'ppv', 'custom', 'buy'],

  // Content triggers (escalate to human)
  escalateWords: ['refund', 'cancel', 'underage', 'legal'],

  // Off-hours behavior
  nightModeEnabled: true,
  nightModeStart: 23,              // 11 PM
  nightModeEnd: 8,                 // 8 AM
  nightModeResponseRate: 0.2,      // Low but not zero
}
```

---

## 10. The Action Dispatch Pattern — Extensible Bot Behavior

### How AIRI Does It

**File**: `services/telegram-bot/src/bots/telegram/index.ts` (lines 23-174)

The LLM returns a JSON action, and `dispatchAction()` is a big switch statement:

```typescript
switch (action.action) {
  case 'read_unread_messages':
    // Fetch messages, embed them, find relevant context
    // Returns () => handleLoopStep() to chain

  case 'send_message':
    // Split message, send with typing delays
    // Returns () => handleLoopStep() to chain

  case 'send_sticker':
    // Find sticker, send it

  case 'list_chats':
    // Return list of all known chats

  case 'continue':
    // Do nothing, check again in 1 min

  case 'break':
    // Clear all context, fresh start

  case 'sleep':
    // Sleep 30s, then check again
}
```

**Critical design**: Each action records its result in `chatCtx.actions[]`, which gets passed to the next `imagineAnAction()` call. So the LLM sees what it already did and what happened.

### How to Adapt for OFM

```typescript
switch (action.action) {
  case 'reply_text':
    // Standard text reply with message splitting
    // Track in conversation history

  case 'send_ppv':
    // Look up content by ID
    // Send teaser message first
    // Then send locked content with price
    // Record in ppv_interactions table

  case 'send_photo':
    // Send a non-PPV photo (free content, teaser)
    // Choose from content library

  case 'upsell_subscription':
    // Pitch higher tier naturally
    // Use fan's interests for personalization

  case 'schedule_followup':
    // Insert into scheduled_outreach table
    // Will be picked up by periodic loop

  case 'add_fan_note':
    // Extract and store a fact about this fan
    // "Likes hiking, has 2 cats, birthday in March"

  case 'escalate_to_human':
    // Flag for human review
    // Send notification to manager
    // Bot stops replying to this chat

  case 'ignore':
    // Log reason, move on
    // Useful for spam, irrelevant messages

  case 'check_spending_history':
    // Look up fan's purchase history
    // Inform next action (what PPV to offer)
}
```

---

## 11. Vector Search for Context Recall

### How AIRI Does It

**File**: `services/telegram-bot/src/models/chat-message.ts`

The `findRelevantMessages()` function is the core of the memory recall system:

```sql
-- Combined relevance scoring
SELECT *,
  (1 - cosine_distance(content_vector, query_vector)) AS similarity,
  (1 - (EXTRACT(EPOCH FROM NOW()) - created_at) / 86400 / 30) AS time_relevance,
  (1.2 * similarity + 0.2 * time_relevance) AS combined_score
FROM chat_messages
WHERE platform = 'telegram'
  AND in_chat_id = :chatId
  AND similarity > 0.5
ORDER BY combined_score DESC
LIMIT 3
```

Then for each match, it fetches a **context window** of 5 messages before and 5 after. This gives the LLM the full conversation thread, not just the single matching message.

### How to Adapt for OFM

Same approach, but add fan-specific filtering:

```sql
-- Find relevant past interactions with THIS specific fan
SELECT * FROM chat_messages
WHERE fan_id = :fanId
  AND cosine_similarity(content_vector, :queryVector) > 0.4
ORDER BY (1.2 * similarity + 0.3 * time_relevance + 0.5 * importance) DESC
LIMIT 5

-- Also search fan notes/facts
SELECT * FROM fan_notes
WHERE fan_id = :fanId
  AND cosine_similarity(note_vector, :queryVector) > 0.3
ORDER BY importance DESC
LIMIT 3
```

The extra `importance` weight ensures that key facts ("his name is Jake", "tipped $500 last week") surface above random chat history.

---

## 12. Observability — Debugging At Scale

### How AIRI Does It

AIRI uses **OpenTelemetry** throughout:

```typescript
const tracer = trace.getTracer('airi.telegram.bot')

return await tracer.startActiveSpan('telegram.module.generate_agent_action.generate', async (s) => {
  s.setAttribute('telegram.bot.id', botId)
  s.setAttribute('llm.chat.model', env.LLM_MODEL!)
  s.setAttribute('llm.chat.messages', JSON.stringify(requestMessages))

  // ... do work ...

  s.setAttribute('llm.chat.generate_text.response.text', res.text)
  s.end()
})
```

They also log every LLM call to `chat_completions_history` table:
- `prompt` — What was sent
- `response` — What came back
- `task` — What it was for
- `created_at` — When

### How to Adapt for OFM

At scale (100s of fans), you need to know:
- Which fan conversations are costing the most tokens
- What actions the bot is taking and why
- When the bot makes mistakes (wrong tone, wrong offer)
- Revenue attribution (which bot interactions led to purchases)

Log everything:
```typescript
interface OFMLogEntry {
  fan_id: string
  action: string
  prompt_tokens: number
  completion_tokens: number
  cost_usd: number
  response_time_ms: number
  fan_tier: string
  resulted_in_purchase: boolean
  purchase_amount?: number
}
```

---

## 13. Concrete Technical Blueprint for OFM Bot

### Tech Stack (Stolen from AIRI)

| Component | AIRI Uses | OFM Recommendation |
|---|---|---|
| **Bot Framework** | Grammy (Telegram) | Grammy (Telegram) + OF API wrapper |
| **LLM Provider** | xSAI abstraction | Same — start with Claude, fallback to GPT-4o |
| **Database** | PostgreSQL + pgvector + Drizzle ORM | Same stack exactly |
| **Embeddings** | @xsai/embed (multi-provider) | Same — OpenAI text-embedding-3-small |
| **Prompt Templates** | Velin (.velin.md) | Same or simpler Handlebars |
| **JSON Parsing** | best-effort-json-parser | Same — LLMs often produce invalid JSON |
| **Image Processing** | @napi-rs/image | Same — fast native Node.js image ops |
| **Vision** | Vision LLM for photo description | Same approach |
| **Tracing** | OpenTelemetry | Same — essential at scale |
| **Process Management** | Direct Node.js | Add BullMQ for job queues |
| **Hosting** | Self-hosted | Railway / Render / VPS |

### Dependencies to Copy Directly

```json
{
  "grammy": "^1.x",                     // Telegram bot framework
  "@grammyjs/files": "^1.x",            // File handling plugin
  "drizzle-orm": "^0.x",                // Type-safe ORM
  "drizzle-orm/pg-core": "^0.x",        // PostgreSQL dialect
  "@xsai/generate-text": "^0.x",        // LLM text generation
  "@xsai/embed": "^0.x",                // Embedding generation
  "@xsai/utils-chat": "^0.x",           // Chat message helpers
  "@xsai/shared-chat": "^0.x",          // Message type definitions
  "best-effort-json-parser": "^2.x",    // Lenient JSON parsing
  "@napi-rs/image": "^1.x",             // Fast image processing
  "@guiiai/logg": "^0.x",               // Structured logging
  "@opentelemetry/api": "^1.x"          // Distributed tracing
}
```

---

## 14. What To Build First (Priority Order)

### Phase 1: Core Bot (Week 1-2)
1. Grammy bot setup with message handlers
2. PostgreSQL + pgvector + Drizzle schema (messages, fan profiles)
3. Single LLM provider (Claude Sonnet)
4. Basic personality prompt (model's voice)
5. Simple reply: message → LLM → response → send
6. Message splitting + typing delays

### Phase 2: Memory (Week 3-4)
7. Embed every message with vectors
8. Semantic recall on every incoming message (the `findRelevantMessages` pattern)
9. Fan profile auto-population (LLM extracts facts → stores them)
10. Context truncation (AIRI's 20-message limit pattern)

### Phase 3: Agent Loop (Week 5-6)
11. Action dispatch system (reply, send_ppv, escalate, ignore)
12. Action chaining (LLM decides next step recursively)
13. Periodic tick loop for proactive outreach
14. AbortController for interruption handling

### Phase 4: Revenue (Week 7-8)
15. PPV offer system (teaser → offer → track purchase)
16. Fan tier classification (spending-based)
17. Tier-aware personality adjustments (VIP treatment)
18. Scheduled outreach (re-engagement, new content announcements)

### Phase 5: Scale (Week 9+)
19. Multi-provider LLM fallback (Claude → GPT-4o → local)
20. Photo understanding (Vision LLM)
21. Human escalation workflow
22. Dashboard (metrics, revenue tracking, conversation review)
23. A/B testing different personality prompts
24. OF API integration (if available)

---

## 15. Code Reference Map

### Files to Study (In Order)

| Priority | File | What You Learn |
|---|---|---|
| 1 | `services/telegram-bot/src/bots/telegram/index.ts` | Full agent loop, message queue, dispatch, periodic tick |
| 2 | `services/telegram-bot/src/types.ts` | All type definitions, action types, context structures |
| 3 | `services/telegram-bot/src/llm/actions.ts` | How `imagineAnAction()` constructs prompts and parses responses |
| 4 | `services/telegram-bot/src/prompts/personality-v1.velin.md` | Personality prompt engineering (126 lines of character definition) |
| 5 | `services/telegram-bot/src/prompts/system-ticking-v1.velin.md` | Action definitions with examples in Velin template format |
| 6 | `services/telegram-bot/src/db/schema.ts` | Full database schema: messages, memories, goals, ideas |
| 7 | `services/telegram-bot/src/models/chat-message.ts` | Vector search, context windows, message recording |
| 8 | `services/telegram-bot/src/bots/telegram/agent/actions/read-message.ts` | How context is assembled: last N + embeddings + semantic recall |
| 9 | `services/telegram-bot/src/bots/telegram/agent/actions/send-message.ts` | Message splitting, typing simulation, cancellable sends |
| 10 | `services/telegram-bot/src/models/common.ts` | Message formatting (one-liner format for LLM context) |
| 11 | `services/telegram-bot/src/llm/photo.ts` | Vision LLM photo interpretation pipeline |
| 12 | `services/telegram-bot/src/bots/telegram/agent/attention-handler.ts` | Response probability, cooldowns, trigger words |
| 13 | `services/telegram-bot/src/prompts/action-read-messages.velin.md` | How recalled context is presented to the LLM |
| 14 | `services/telegram-bot/src/prompts/message-split-v1.velin.md` | Prompt for natural message splitting |
| 15 | `packages/stage-ui/src/libs/providers/` | Multi-provider LLM abstraction (if you need fallbacks) |

### Files You Can Ignore

Everything in: `packages/stage-ui-three/`, `packages/stage-ui-live2d/`, `packages/audio*/`, `packages/model-driver-lipsync/`, `services/minecraft/`, `apps/stage-*`, `packages/ccc/` (unless you want structured character cards).

---

*The bottom line: AIRI's Telegram bot is essentially an OFM bot with the wrong personality prompt. Swap the anime girl for a model persona, add PPV/tip tracking, and you have 80% of what you need.*
