# Marinara Engine: Architecture Overview

Marinara Engine is a local-first AI roleplay and chat frontend. Users run it on their own machine (Windows installer, Docker, Node.js clone, or Termux on Android), plug in their own AI API keys (OpenAI, Anthropic, Google, xAI, OpenRouter, Mistral, Cohere, or any OpenAI-compatible endpoint), and chat with AI characters.

Repo: https://github.com/Pasta-Devs/Marinara-Engine
Built by SpicyMarinara & Pasta-Devs. License: AGPL-3.0.

## The Big Picture

```
┌─────────────────────────────────────────────────────────────┐
│  Client (React / Tailwind v4, PWA-capable)                  │
│  - Chat UI  - Agents panel  - Lorebook editor  - Extensions │
└──────────────────────────────┬──────────────────────────────┘
                               │ fetch /api/*
┌──────────────────────────────┴──────────────────────────────┐
│  Server (Fastify, TypeScript, file-native JSON store)       │
│  Routes: characters, lorebooks, agents, custom-tools,       │
│          generate, chats, presets, connections, etc.        │
│  ↓                                                          │
│  Generation pipeline (packages/server/src/routes/generate*) │
│   ├─ Assemble system prompt from: card + scenario +         │
│   │   connected chats + awareness + lorebook injection      │
│   ├─ Run pre-generation agents (inject, review, rewrite)    │
│   ├─ Stream response from the configured LLM provider       │
│   │    (with tool-calling if tools are enabled)             │
│   ├─ Run parallel agents (images, music, echo reactions)    │
│   └─ Run post-processing agents (state extraction, editor)  │
│  ↓                                                          │
│  File store: messages, chats, characters, personas,         │
│  lorebooks, presets, agents, custom-tools, etc.             │
└─────────────────────────────────────────────────────────────┘
```

## The Core Objects

### Character
A persona the user chats with. Follows the V2 character card spec. Stored as JSON in the file-native store (`DATA_DIR/storage`). Fields include `name`, `description`, `personality`, `scenario`, `first_mes`, `system_prompt`, plus Marinara-specific extensions (appearance, backstory, name/dialogue colors, schedule, etc.). Can be created in the app, imported from SillyTavern, imported from PNG files with embedded metadata, or browsed from Chub.ai via the Bot Browser.

See `references/character-cards.md`.

### Persona
The user's own identity in chats. Same shape as characters but simpler. Has name, description, personality, appearance, avatar, and optional custom colors. Multiple personas per user; the active persona is substituted for `{{user}}` macros in prompts.

### Lorebook
A keyword-triggered knowledge base. Entries have `keys` (trigger words), `content` (what gets injected when a key is matched in recent chat context), and a bunch of tuning knobs (sticky, cooldown, delay, selective, grouped, recursive). Can be global, per-character, or per-chat.

See `references/lorebooks.md`.

### Preset
A full prompt configuration. Controls the order and content of system prompt sections, generation parameters (temperature, top-p, max tokens), and optional choice blocks. The default is **Marinara's Universal Preset v12** (`packages/server/src/db/default-preset.json`), and as of v2.0 presets carry prompts for **Conversation and Game modes** too, not just roleplay.

### Connection
An API configuration pointing at an LLM provider: provider name, API key (encrypted at rest with AES-256), model, base URL, max context length. Users need at least one connection to chat. Per-chat overrides are supported. Connections can also target **non-LLM media providers** — `image_generation` and `video_generation` (v2.1) — which are handled by the media services, not the LLM provider registry; a video connection can be flagged **"Default for Videos"** as a per-chat fallback. Root-level connections can be reordered by drag or moved into folders via a saved Custom sort order (`connection.ts sortOrder`, `connection-folders.ts sortOrder`) (v2.1).

### Agent
An autonomous LLM sub-system that runs during generation. 21 built-ins plus user-defined custom agents. Each agent has a phase (pre/parallel/post), a system prompt, an optional dedicated connection, and settings. All disabled by default — users enable only what they need.

See `references/agents.md`.

### Custom Tool
A user-defined function the main chat model can call during generation. Three execution types: `static` (hardcoded response), `webhook` (POST to a URL), `script` (sandboxed JS, no network). Exposed to the model as OpenAI-compatible function definitions.

See `references/custom-tools.md`.

### Extension
Client-side CSS + JS loaded at runtime. Gets a scoped `marinara` API for DOM injection, event handling, and calling the Marinara server's own API.

See `references/extensions.md`.

### Professor Mari (built-in assistant)
Marinara's built-in assistant, seeded at first run and not deletable. **As of v2.0 she is the Home-screen assistant** (no longer a normal Conversation-mode character): users talk to her from the Home screen, where a Pi-backed *workspace agent* can inspect the local app and — with browser approval for database changes — create content (characters, personas, lorebooks, chats) and navigate panels. She replaced the standalone character/persona/lorebook *maker* modals and their generation routes, removed in v2.0.

See `references/character-cards.md` → The Professor Mari Pattern, and `docs/PROFESSOR_MARI.md`.

## Chat Modes

Marinara has three chat modes. Each has a different prompt assembly pipeline and UI.

### Conversation Mode 💬
Discord-style DMs. Casual text, no asterisks, no narration. Characters have:
- **Schedules** — weekly timetables of activities, generated by an LLM based on personality
- **Statuses** — derived from schedule: online, idle, dnd (do-not-disturb), offline
- **Talkativeness** — 0–1, affects autonomous messaging rate
- **Selfies** — characters can emit `[selfie]` to generate a selfie-style image
- **Memory commands** — characters can send memories to other characters via `[memory: target="..."]`
- **Scenes** — conversation characters can initiate temp roleplay chats via `[scene: ...]`
- **Turn games (UNO)** — v2.0 added UNO/turn-game support for Conversation chats: in-character setup flow, bot turns, and board state (route `/api/turn-games`)
- **Reactions (v2.1)** — characters can emit `[react: emoji="😂"]` (or a custom `[react: emoji=":name:"]`) to badge the user's latest message, or `[react: emoji="🙄" to "Character Name"]` to react to another character; can target an individual character's segment within a merged multi-character reply, and a react to the persona name (or "User") targets the user's latest message. Conversation-mode only (ignored elsewhere). Gated by a per-command **Reactions** card toggle (setup wizard / Chat Settings). Grammar in `character-commands.ts` REACT_RE (also accepts `[react: "😂"]` / `[react: 😂]`).

Group DMs are supported — the engine picks who speaks next.

**Conversation Calls (v2.1)** — Discord-style audio/video calls inside Conversation chats. Gated globally by `ttsSettings.callAudioEnabled` (default false, Settings → TTS) plus a per-chat opt-in. Characters can initiate **incoming calls** (with an optional greeting that plays after the user answers). The desktop/mobile call surface has a call-only chat, speaking highlights, mute / camera / screen-share, a soundboard, minimized active-call popouts, call history cards, and a **post-call summary** injected back into the chat.
- **Voice input** — `callAudioInputMode` (`tts.ts:142`, default `local_whisper`) selects the mic path: `system` (provider-native audio), `auto`, `transcribe` (browser speech recognition), or `local_whisper` (downloadable Whisper Tiny/Base transcription). `callVideoInputEnabled` (`tts.ts:144`, default false) gates the camera/screen-share UI. (`callSttConnectionId`/`callSttModel` are deprecated.)
- Character **video presence** in calls, the **Sprites → Clips** editor, and the in-call `[custom_clip: …]` / `[react:]` commands are covered in `references/character-cards.md`.

### Roleplay Mode 🎭
Traditional creative-writing roleplay. Rich narration, prose. Supports:
- Full agent stack (world state, trackers, sprites, backgrounds, weather, combat)
- VN-style character sprite overlays — a chat can show all enabled sprite owners (the hard-coded 3-sprite cap was removed in v2.1)
- Scene videos (v2.1) — per-image **Animate** and Gallery Video actions render MP4s via a Video Generation connection, shown as pinnable/draggable video overlays (shared with Game Mode and Visual Novel galleries)
- Custom backgrounds with crossfade transitions
- Weather particle effects (rain, snow, thunderstorm, fog, cherry blossoms, aurora)
- Time-of-day lighting (dawn, day, dusk, night)
- Game HUD with character stats, quests, world state

### Game Mode 🎮
A shipped mode (v2.0) — Roleplay + JRPG game loop. The model acts as GM; the engine handles mechanics. State machine with `exploration`, `dialogue`, `combat`, `travel_rest`. Tactical combat UI, dice rolls, skill checks, reputation, elemental reactions, quest tracking, auto-journaling. v2.0 also added a Roleplay HUD / Tracker Panel with **editable field locks** (manually pinned tracker values survive generated updates; v2.1 hardened this so tracker/custom-agent update agents can't bypass a locked field by renaming or replacing the locked row at the same position), **Game-Mode custom-agent selection** in Chat Settings, a setup wizard, and checkpoint restore. Routes: `/api/game`, `/api/game-assets`.

**Scene videos (v2.1)** — a dedicated Video Generation connection (`chat.ts gameVideoConnectionId`/`sceneVideoConnectionId`, selected under Chat Settings → Agents → Scene Videos or the setup wizard) renders MP4s into a scene-video store; they surface in the Gallery **Videos** tab with per-image **Animate** buttons and pinnable/draggable video overlays (also available in Roleplay and Visual Novel galleries).

**Turn storyboards (v2.1)** — a Prompt Director splits a completed GM narration into 2–6 anchored keyframes (usually 4), renders their media concurrently, and shows them in a draggable viewer that follows the current story section (reopenable from Gallery). Keyframes land in the Gallery Images tab; with the off-by-default **Automatic Storyboard Animations** each is also animated into an MP4. Per-chat: `gameStoryboardAutoIllustrationsEnabled`, `gameStoryboardAutoGenerationEnabled` (`chat.ts:477-480`); a manual "Create storyboard" button needs the Game Illustrator image connection. Types `game.ts:611-681`; see `docs/STORYBOARD_ENGINE_GUIDE.md`.

**Media prompt presets (v2.1)** — three per-chat selectors: **Illustration Prompt**, **Animation Prompt**, **Game Video Prompt**. Read-only built-ins: Still Keyframes, Comic Page, Colored Manga, B&W Manga (`game-storyboard-prompts.ts:85-113`) and **Cinematic Scene Video** (`game-video-prompts.ts:31-34`, the default Game Video Prompt); users get chat-local editable copies (`chat.ts` gameStoryboard*/gameVideo* template fields). The global video template key is `game.video` (renamed from `game.omniVideo` in v2.1; the legacy key is still read as a fallback).

**Game Illustrator dynamic prompts (v2.1)** — an optional Illustrator toggle (Chat Settings → Agents → Illustrator) for **Dynamic LLM Prompt Generation**: `gameImageDynamicPromptEnabled` (`chat.ts:250-251`) asks the chat/prompt LLM to rewrite Game Mode NPC-portrait, location-background, and key-moment illustration prompts before image gen (optionally via `illustratorPromptConnectionId`). Distinct from the post-processing `illustrator` agent. Related: `gameImageAutoGenerationEnabled` (`chat.ts:248`).

## Prompt Assembly (Simplified)

For a roleplay turn, the system prompt is assembled roughly as:

1. **Preset sections** (ordered, user-configurable) — typically:
   - System role / jailbreak
   - Character description + personality + scenario
   - Persona description
   - World info / lorebook entries (injected based on keyword matches in recent messages)
   - Examples of dialogue
   - Chat summary (if enabled and conversation is long)
   - Connected-chat awareness (if this chat is linked to another)
   - Agent injections (from pre-generation agents)
2. **Recent message history** — the chat's last N messages
3. **Depth-injected prompts** — special prompts inserted at specific depths in the history
4. **Post-history instructions** — note (v2.0): post-history *system* sections are now injected as **user-side** content at a configured depth (preserving metadata) rather than glued onto the pre-history system prompt

**v2.0 prompt-assembly changes worth knowing:** max-context enforcement prioritizes non-history material first and *then* windows recent history (preserving response/free-token headroom); assistant **prefill steering** seeds the generated message with the configured prefill; and **visible tracker context** is included. (CHANGELOG [2.0.0].)

**v2.1 prompt-assembly changes:** in 1:1 Conversation chats, assistant turns in prompt history are now **speaker-labeled with the character name**; **impersonate** assembly drops conflicting non-marker sections on fallback presets while explicitly-selected impersonate presets keep their normal sections (#3209), and smart group response order no longer overrides `/guided` or `/impersonate` (#3212); readable text attachments are no longer pre-truncated to 60,000 chars — they are bounded by the model's context window. (CHANGELOG [2.1.0].)

For conversation mode, assembly is simpler: system prompt (mode-specific framing + character card) + name lists + fetched context + message history.

The assembly lives in `packages/server/src/routes/generate.routes.ts` (~10,800 lines) and `packages/server/src/routes/generate/` helpers. It is **not user-extensible without forking**.

## Data Storage

**File-native JSON storage by default** (`STORAGE_BACKEND=files`), under `DATA_DIR/storage` (default `packages/server/data/storage`). SQLite is a **legacy opt-in** (`STORAGE_BACKEND=sqlite`); `DATABASE_URL` is only consulted to import old data when `DATA_DIR/storage` doesn't yet exist. All user data lives in the file store:
- chats, messages, chat_folders
- characters, personas, groups, persona_groups
- lorebooks, lorebook_entries
- presets, regex_scripts
- connections (with encrypted API keys)
- agents (both built-in configs and custom agents)
- custom_tools
- themes, extensions
- gallery (generated/uploaded images **and videos**; v2.1 renamed "clips" → **Videos** and split galleries into Images/Videos tabs, newest-first)

Fully local. Backing up = copying the `DATA_DIR/storage` directory (or the whole `DATA_DIR`). Sharing = exporting individual objects as JSON (characters, presets, lorebooks all have export endpoints).

## API Surface (Key Endpoints)

All under `/api/*`:
- `/api/characters` — CRUD, groups, import/export (JSON/PNG)
- `/api/personas` — (within characters routes)
- `/api/lorebooks` — CRUD, entries, export
- `/api/presets` (`/api/prompts`) — CRUD, sections, choice blocks
- `/api/connections` — CRUD, duplicate, test
- `/api/agents` — CRUD, toggle, echo messages
- `/api/custom-tools` — CRUD
- `/api/generate` — main SSE generation endpoint with the full agent pipeline
- `/api/chats` — CRUD, messages, connect/disconnect
- `/api/scene` — create/plan/conclude scene branches
- `/api/conversation` — schedule, status, autonomous message checks
- `/api/encounter` — combat init/action/summary
- `/api/game`, `/api/game-assets` — Game Mode loop and per-game asset management
- `/api/turn-games` — Conversation turn games (UNO)
- `/api/conversation-calls` — Conversation Calls (v2.1): `GET /chat/:chatId/status`, `POST /start`, `/:id/accept|decline|end`, `GET|POST /:id/messages`, `/:id/interruption|idle|media`, soundboard CRUD (`/soundboard`, `/soundboard/upload`, `/soundboard/:id/file`, `DELETE /soundboard/:id`), and character/persona video-clip endpoints (`/character-videos/:id` + `/generate`, `/custom/generate`, `/file/:kind`, `/custom/:clipId/file`; `/persona-videos/*` mirror)
- `/api/gallery/generate-scene-video`, `/api/gallery/scene-videos/:chatId`, `/api/gallery/scene-videos/file/:chatId/:filename` — scene-video generation (v2.1; rejects a connection whose provider ≠ `video_generation`); parallel `/api/game/scene-videos/*` plus `gameVideoConnectionId` on Game setup
- `/api/professor-mari/workspace` — Professor Mari's workspace agent: AI-assisted creation of characters/personas/lorebooks/chats. **This replaced the standalone `/api/character-maker`, `/api/lorebook-maker`, and `/api/persona-maker` routes, which were removed in v2.0.**
- `/api/bot-browser/*` — import from Chub, CharacterTavern, JannyAI, Pygmalion, Wyvern, DataCat

Full list in `docs/FRONTEND.md`.

## Providers Supported

- OpenAI (incl. ChatGPT subscription), Anthropic (incl. Claude subscription), Google (Gemini + Vertex AI), xAI (Grok), Mistral, Cohere, OpenRouter, NanoGPT
- Any custom OpenAI-compatible endpoint (use "Custom" provider)
- Local Model runtime: a **llama.cpp sidecar** (MLX on macOS Apple Silicon) that runs downloadable local models — including the built-in **Gemma 4** option offered on the Local Model card. When the native-tool-calls toggle is on it launches `llama-server` with `--jinja`, giving **OpenAI-compatible native tool calling**; useful both for offloading tracker/scene-analysis work and for running custom tools locally
- Image gen: Pollinations, Stability AI, Together AI, NovelAI, ComfyUI (with custom workflows), AUTOMATIC1111
- **Video Generation (v2.1)** — a separate connection provider type (`video_generation`, in the APIProvider enum) handled by `services/video/video-generation.ts` rather than the LLM provider registry. Powers scene videos, Game storyboards, animated expression sprites, and call video presence. Five service profiles (`VIDEO_DEFAULTS_SERVICES`) with default models: Gemini Omni (`gemini-omni-flash-preview`), Google Veo (`veo-3.1-generate-preview`), xAI Imagine (`grok-imagine-video-1.5`), OpenRouter Video (`google/veo-3.1`, any OR video id), Seedance 2.0 (`seedance-2-0`). Per-service defaults store under `connection.defaultParameters.videoGeneration`; a connection can be flagged **"Default for Videos"**. Source of truth: `docs/SCENE_VIDEO_GENERATION.md`.
  - Per-service duration/aspect/resolution profiles (`video-generation-defaults.ts:25-54`): Gemini Omni 10s/16:9 (duration baked into the prompt; rejects `duration_seconds`), Veo 8s/16:9/720p (accepts only 4/6/8s; forces 8s with an image ref), xAI 10s/16:9/720p, OpenRouter 10s/16:9/720p, Seedance 5s/16:9/720p (opt-in `temporaryPublicReferenceUploadEnabled` + expiry 1h/12h/24h/72h, default 12h).
  - Advanced → Video Generation settings (key `video-generation`): `sceneVideoDurationSeconds`=10 (clamp 1–60s), `callCustomClipDurationSeconds`=5 (call clips clamp 1–15s), `animatedExpressionClipDurationSeconds`=3 (clamp 1–8s), per-kind `callClipDurations` all 5s; plus reusable prompt templates.
  - Env/config (troubleshooting): `VIDEO_GEN_TIMEOUT_MS` (default 30 min); poll intervals `GOOGLE_VEO`/`XAI`/`OPENROUTER`/`SEEDANCE_VIDEO_POLL_INTERVAL_MS` (10/5/10/10s); first/last-frame refs need public HTTPS via `VIDEO_REFERENCE_PUBLIC_BASE_URL` or per-connection temp-upload; remote downloads are HTTPS-only, validated as MP4.
- **TTS / voice (v2.1)** — a voice-provider union distinct from the LLM list: `openai | elevenlabs | pockettts | xai` (`tts.ts:4`). **xAI TTS** was added in v2.1 with built-in voice fallbacks (separate from the LLM `xai`/Grok provider).

## Special Features Worth Knowing

- **Cross-chat awareness** — when the user mentions "yesterday," "last week," etc., the system retrieves relevant messages from the character's other chats and injects them as `<awareness>` XML.
- **Semantic memory (message RAG)** — messages are chunked (5 at a time), embedded with `all-MiniLM-L6-v2` running locally, and retrieved by cosine similarity. Top 8 chunks with threshold filtering, toggle per-chat.
- **Regex scripts** — user-defined find/replace that runs on inputs and/or outputs, for formatting cleanup or custom macro systems.
- **Macros** — `{{char}}`, `{{user}}`, `{{time}}`, `{{date}}`, `{{random::a,b,c}}`, and more in prompts.
- **Connected chats** — conversation ↔ roleplay bidirectional links with `<influence>` (conversation → RP) and `<ooc>` (RP → conversation) tags.
- **Spotify integration** — a Spotify DJ agent can control playback via built-in tools (`spotify_play`, `spotify_search`, etc.) to score the scene.
- **Discord webhook mirror** — per-chat Discord webhook that mirrors user and AI messages.
- **Haptics / Love Toys** — the `haptic` agent ("Haptic Feedback") drives devices via Intiface Central (Buttplug protocol) running locally (yes, really; it's in the README).
- **Slash commands** — client-parsed `/`-commands (`slash-commands.ts`). Notably **`/illustrate`** (alias `/ill`) in Conversation, Roleplay, and Game chats triggers the same illustration action as the Gallery Illustrate button without opening the Gallery (v2.1; 30-min timeout `ILLUSTRATE_SLASH_TIMEOUT_MS`).
- **In-app Documentation viewer (v2.1)** — the `docs/` guides are browsable inside Marinara via a **Documentation** button in the home-page footer (next to Replay Tutorial); a FAQ entry shows the on-disk docs path. Support/ideation answers can point users to the bundled docs in-app.

## What's NOT Built-In

Important for recommendations:
- No native external vector DB integration (lorebook uses local embeddings only).
- No plugin marketplace — extensions are manual CSS/JS install.
- No hooks into the prompt assembly pipeline — can't insert your own logic mid-generation without forking.
- No multi-user auth — it's single-user / local-first by design. Multi-user requires a proxy + auth layer you build.
- Script tools have no network, no filesystem, no `require`. Pure JS computation only.
- No built-in way to add new chat modes.
- No native GraphQL, no tRPC, no realtime subscriptions beyond SSE for generation.

If the user needs any of these, the answer is usually "webhook tool to your own backend" or "fork the engine."
