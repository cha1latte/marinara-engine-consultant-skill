# Decision Guide: Picking the Right Architecture

This is the consultant's main reference. When a user describes a goal, walk through these questions in order. Stop at the first one that fits.

## First: which chat mode?

Before architecture, pick the **mode** the experience runs in — this is orthogonal to the surfaces below (a card + lorebook + tools stack works in any of them):

- **Conversation** — default one-on-one (or small group) chat. Pick it for an assistant, a companion, or a straightforward back-and-forth.
- **Roleplay** — immersive narrative RP with a character or cast; the model narrates scenes and drives story. Pick it for story/character immersion.
- **Game** — GM-driven interactive fiction (Game Mode): the engine runs a game master over narration, storyboards, and scene beats. Pick it for structured, quest-like play.
- **Noodle (v2.2)** — Marinara's fake social network: invited characters (and optional random users) post, reply, poll, like, repost, and mention each other on a persistent, refreshing timeline; **personas participate directly**, and social memory carries over into Conversation/Roleplay/Game. Pick it for social-feed / timeline personas — a living multi-character social simulation rather than a direct chat. See `references/architecture.md` for the full Noodle section.

## The Nine Questions

### 1. Is the entire knowledge set small and completely stable?
**Small** = under ~2000 tokens of reference material. **Stable** = won't change for months.

→ **Put it in the character card.** Use `description` for the personality/role, `system_prompt` for reference knowledge, XML-tagged sections for structure. This is the Professor Mari pattern in its purest form, minus the assistant commands.

**Fits:** Planet Zoo modding reference, D&D 5e rules summary, a specific VN's lore, a company's style guide.

**Doesn't fit:** Anything that needs updates more than quarterly, anything with entity lookup patterns ("tell me about character X out of 200").

---

### 2. Is the knowledge large but organized into discrete, lookup-shaped entries?
**Discrete entries** = you can imagine queries like "tell me about X" where X is a keyword. **Large** = too big for the prompt (> 2k tokens) but finite (< a few MB of text).

→ **Use a lorebook with keyword triggers.** Each entry has keys (the trigger words), content (the text that gets injected), and optional config (constant/ephemeral/grouped/recursive).

**Fits:** WoW raid boss mechanics, a fantasy world's 300 named characters, a law firm's internal case-law citations, a tech company's product catalog.

**Doesn't fit:** Knowledge where the query isn't obviously keyword-shaped ("summarize yesterday's news," "what's trending on Reddit right now").

See `references/lorebooks.md` for entry fields and tuning.

---

### 3. Does the knowledge change frequently (weekly or faster)?
**Frequently** = there's a live authoritative source somewhere (an API, a scraper, a RSS feed, a database, a spreadsheet).

→ **Custom tool with `webhook` execution.** Stand up a tiny backend (Cloudflare Worker, n8n, Express on a VPS, Zapier webhook) that the character can call. Return JSON to the model.

**Fits:** Path of Titans meta (patch notes change per release), sports scores, stock prices, your band's next show, "what's in the news about X."

**Doesn't fit:** Knowledge that's stable but you just haven't written it down yet (write it down — use lorebook or card).

See `references/custom-tools.md` for the webhook pattern.

---

### 4. Does the character need to perform **actions** — not just answer?
Actions = effects in the world. Creating a file, updating a row in your DB, sending a Slack message, triggering a workflow, fetching a specific record.

→ **Custom tool.** Static for stubs and testing. Webhook for anything that reaches outside the engine (network calls, databases, third-party APIs). Script for pure computation (math, string transforms, date calculations) where no I/O is needed.

**Two common "capabilities" that are NOT hand-built custom tools:**
- **Generating or animating images/video** is a *native* Marinara capability — attach a Video Generation (or image) connection, don't stand up a webhook. See Q8.
- **Controlling a Home Assistant setup** — Marinara can **auto-generate** the webhook tools for you from your HA entities and re-sync them (needs `WEBHOOK_LOCAL_URLS_ENABLED=true`, since HA is local plain-HTTP). See `references/custom-tools.md`.

**Fits:** "Look up the WordPress config for client X," "create a character based on this description" (this is literally what Mari does), "compute the optimal grow schedule given these inputs."

**Critical reminder:** The `script` execution type is **disabled by default** — it only runs if the server sets `CUSTOM_TOOL_SCRIPT_ENABLED=true`. It has **no network access, no filesystem, no require()**. The sandbox exposes only `args`, `context` (when `includeHiddenContext` is set, else `null`), `JSON`, `Math`, `String`, `Number`, `Date`, `Array`, `parseInt/parseFloat/isNaN/isFinite`, and a no-op `console.log`. Timeout is the shared custom-tool timeout (60s default, `CUSTOM_TOOL_TIMEOUT_MS`). If the action needs anything outside pure JS computation, it must be a `webhook`.

See `references/custom-tools.md` for full execution type breakdown.

---

### 5. Does something need to happen **automatically on every turn**?
Per-turn automation = not user-initiated, not tool-triggered — just runs in the background as part of message generation.

→ **Custom agent**, placed in the right phase:
- **`pre_generation`** — runs before the main response. Use for: injecting context, reviewing the prompt, rewriting directives.
- **`parallel`** — runs alongside the main response. Use for: side tasks that don't block (image generation, music suggestions, reactions from absent characters).
- **`post_processing`** — runs after the main response. Use for: fact-checking, state extraction, rewriting for style, tracking variables.

Agents cost real tokens and latency every turn. Only use them when the job genuinely needs to happen on every message.

**Fits:** A "tone enforcer" that rewrites every message to stay in-period for a historical RP; a "combat tracker" that extracts damage numbers from narration into structured HP; a "continuity checker" that flags contradictions.

**Doesn't fit:** Things that only need to happen occasionally (those should be tools the model calls when needed).

See `references/agents.md` for phases, default prompts, custom agent creation.

---

### 6. Does the **UI** itself need to change?
UI = DOM. Buttons, overlays, custom indicators, theme tweaks, new panels, embedded widgets.

→ **Client extension.** CSS + JavaScript loaded by `CustomThemeInjector.tsx`. JS gets a scoped `marinara` API with:
- `addStyle(css)`, `addElement(parent, tag, attrs)` — DOM injection with auto-cleanup
- `apiFetch(path, options)` — call Marinara's own `/api/*` endpoints
- `on(target, event, handler)`, `observe(target, callback)` — listeners/MutationObservers, auto-cleaned
- `setInterval`, `setTimeout`, `onCleanup` — scheduling with auto-cleanup

**Fits:** Adding a word-count indicator below the input, injecting a custom theme, showing a visual token meter, adding a "copy as markdown" button to messages.

**Doesn't fit:** Anything requiring server-side changes (new routes, new DB tables, new agents). Those require forking the engine.

**Check Settings → Appearance before recommending custom CSS for theming.** v2.0 made much theming native — accent color, RGB/pulse, app background + gradients, chat text colors, font, and "Reset Appearance" — plus a server-synced **custom themes** system (`/api/themes`). Use a CSS extension only for structural/DOM tweaks the native controls can't express. (Professor Mari can also generate themes and extensions for you — see `references/extensions.md`.)

See `references/extensions.md` for the API surface.

---

### 7. Does the solution need to cross chats, persist structured state, or integrate deeply with external systems?

→ **Webhook tool + your own backend.** The engine's own persistence is scoped to chats, characters, personas, lorebooks, presets. If you need structured state outside that (CRM data, analytics, cross-user aggregation, ML pipelines, real databases) — you run that infrastructure yourself and expose it to the character via webhook tools.

**Fits:** "My support assistant needs to log every conversation to our CRM," "the character needs to remember things globally across all my users," "I want vector search across 10 years of company docs."

**Doesn't fit:** Anything inside the engine's native scope — use the native features.

---

### 8. Does the character need to generate or animate images/video? (v2.1)
Media generation = creating a picture or a short video clip — a scene illustration, an animated portrait, an in-call avatar, a storyboard keyframe.

→ **Native media generation — a Video Generation (or image) connection, NOT a webhook.** Video is a first-class capability in 2.1: add a **Video Generation** connection and point the relevant surface at it. There is no need to build a custom tool for this.

The surface depends on where the video should appear:
- **Gallery "Animate"** — turn a generated image into a short clip.
- **Game Mode storyboards / scene videos** — animate GM narration keyframes.
- **Animated expressions** — expression portrait sprites rendered as short clips → looping GIFs.
- **Conversation-call video presence** — cached avatar clips played in-call.

**Fits:** "Can my character generate a video of the scene?", "I want an animated portrait", "make the avatar move during a call."

**Doesn't fit:** Nothing here — do **not** route video requests to a webhook custom tool (Q3/Q4). See `references/architecture.md` and `references/character-cards.md` for the connection setup and surfaces.

---

### 9. Does the user just need to rewrite/clean up prompt or output *text*? (Regex Scripts)
Text transform = find/replace on the strings flowing through the pipeline — strip a leftover prefix, swap a name on the way in or out, hide a control token, tidy formatting. **No** callable capability, **no** DOM change — just string rewriting.

→ **Regex Scripts** (SillyTavern-style). Scoped **per-character** and **per-preset**; a script is a regex `find` + `replace` applied to prompt and/or model output. SillyTavern regex scripts import over directly. This is a distinct modding surface — don't misuse a custom tool (Q4) or a UI extension (Q6) for text transforms.

**Fits:** "Strip the `<think>` block from replies," "always rewrite 'the assistant' to the character's name," "clean up markdown the model over-formats."

**Doesn't fit:** Anything that needs to *call out* or *compute* (that's a tool) or change the UI (that's an extension).

See `references/custom-tools.md` → Regex Scripts and `docs/REGEX_SCRIPTS.md`. *(Added in an intermediate 2.0.x update and documented in the 2.1 doc refresh — an established surface, not brand-new in 2.1.)*

---

## Common Combinations (Stacks that Work)

Most real projects are two or three of these surfaces together. Don't recommend just one when a combination is cleaner.

### "Documentation character" (like Professor Mari)
**Character card** (personality + framing) + **system prompt with reference material** (if it fits) or **lorebook** (if it doesn't). No tools needed unless the character should do things to the user's environment.

### "Live-data assistant"
**Character card** (voice, framing, what not to do) + **webhook custom tool** (the live source) + optional **lorebook** (stable background context).

### "Character that manages your app"
**Character card** + **multiple custom tools** (one per action the character can take) + optional **custom agent** to nudge the character to use the tools naturally.

### "Immersive RP character"
**Character card** + **lorebook** (world info) + **post-processing agent** (state tracker) + optional **parallel agent** (image generation, music) + optional **extension** (custom HUD).

### "Knowledge base over a large structured dataset" (300 WordPress sites, customer records, etc.)
**Character card** (teaches the model how to look things up) + **webhook custom tool** (`lookup_by_id`, `search_by_field`, `list_all`) + **your own backend** (the actual data store). Do NOT try to put the dataset itself in the lorebook unless it's small and the lookup pattern is keyword-shaped.

---

## Red Flags That Signal "This Is Wrong"

Push back when you hear these:

- **"I'll put 10,000 entries in the system prompt"** — Every turn re-sends the full prompt. Impossible past context limits, wasteful before that.
- **"I'll use a script tool to hit my API"** — Script sandbox has no network. Use webhook.
- **"I want it to *always* know the current date"** — The engine already injects date/time into the conversation system prompt each turn. No tool needed.
- **"I'll make a custom agent to call my API every turn"** — Technically works but wastes tokens. Prefer a tool the model calls when needed.
- **"I want the character to edit their own card"** — Possible via API call in a tool, but architecturally weird; usually a sign the state should be elsewhere (lorebook, external DB).
- **"I'll have three separate agents do the same job"** — Agents don't combine; their outputs are independent injections. Pick one.
- **"I'll hardcode all the user data into the character"** — Use personas (the engine's user-identity feature) for that.

---

## Questions to Ask the User (When Unclear)

Ask AT MOST ONE before drafting options. Don't block.

1. **"Is the knowledge stable, or does it change?"** — Gates lorebook vs. webhook tool.
2. **"Do you want the character to *do* things, or just answer?"** — Gates tool-calling vs. pure conversation.
3. **"Are you running this on a frontier model (Claude, GPT-5, Gemini) or something smaller/local?"** — Affects how much you can offload to the model's native knowledge. *Also gates tool use:* on the local llama.cpp sidecar, native (OpenAI-compatible) tool calling only works when it's launched with `--jinja` (the native-tool-calls runtime toggle). If they're on a local model and want custom tools, they must enable that toggle.
4. **"Is this a one-off or something you'll maintain long-term?"** — Affects whether to optimize for build speed (card + prompt) or maintainability (lorebook + tools).
5. **"Do you already have this data somewhere (Google Sheet, DB, API)?"** — Unlocks the webhook path.
6. **"Is this for you alone, or will others use it?"** — Affects how much UX polish matters (extensions, tool confirmation flows).

Don't ask all six. Pick the one that actually gates the recommendation.
