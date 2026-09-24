# Thinking, effort & models

This file describes the **steps followed** to add Concept 14 (**Thinking, effort & models**) to the Claude Agent SDK
Lab: what was read, what was decided, how it was tested, and what the tests changed.
To learn the concept itself, read [Tab14-Thinking-effort-and-models.md](Tab14-Thinking-effort-and-models.md).

| Concept | Topic | Routes | Explanation |
|---|---|---|---|
| 14 | `thinking`, `display`, `effort`, `fallbackModel`, `supportedModels()`, `applyFlagSettings({ effortLevel })`, `setMaxThinkingTokens()`, `setModel()` | `/api/c14/run`, `/models`, `/session`, `/send`, `/effort`, `/thinking`, `/model`, `/end` | [Tab14-Thinking-effort-and-models.md](Tab14-Thinking-effort-and-models.md) |

## How to run it

```powershell
npm run dev        # server on http://localhost:3001, web on the Vite port
```

`node_modules` was copied from sample13, so `npm install` is not needed. Open the **14. Thinking, effort & models** tab.

> **Only one sample can run at a time.** Every sample's server uses port **3001**. Stop the other samples'
> `npm run dev` first.

## Step 1: Choose the feature

The request again said "implement the following feature" with no feature text. `sample14/` was a copy of sample13
(without `node_modules` and `readme.md`). Four topics were offered: **Cost & usage tracking**, **Plugins**,
**File checkpointing & rewind** and **Thinking, effort & models**. **Thinking, effort & models** was chosen.

## Step 2: Read the existing samples

| Read | To learn |
|---|---|
| `server/concepts/02-options.ts`, `Concept02Options.tsx` | Building `options` only from the fields the user chose; the model list |
| `server/concepts/12-streaming-input.ts`, `Concept12StreamingInput.tsx`, `Tab12-Streaming-input.md` | The input queue, the sessions `Map`, `control()`, and what `setModel()` / `supportedModels()` already covered |
| `server/concepts/07-hooks.ts` | The `HookCallback` shape, reused for the Stop hook |
| `server/index.ts`, `server/sse.ts`, `src/App.tsx`, `src/styles.css`, `Tab1-query().md` | Mounting a router, SSE, the tab list, CSS to reuse, the concept table |

Concept 12 already switched models in a live session, so this concept adds only what is new: `thinking`, `effort`,
the capability fields of `supportedModels()`, and `fallbackModel`.

## Step 3: Check the types

`sdk.d.ts` (`0.3.281`): `Options.thinking` (`ThinkingConfig`: `adaptive` / `enabled` + `budgetTokens` / `disabled`,
with `display`), `Options.effort` (`EffortLevel`), `Options.maxThinkingTokens` (deprecated), `Options.fallbackModel`,
`ModelInfo` (`supportsEffort`, `supportedEffortLevels`, `supportsAdaptiveThinking`), `ModelUsage.thinkingTokens`,
`SDKThinkingTokensMessage`, `Query.setMaxThinkingTokens()`, `Query.applyFlagSettings()` (whose comment describes
`effortLevel`) and `BaseHookInput.effort`. There is no `setEffort()`.

## Step 4: Experiment before designing

Three scratch scripts called `query()` directly, with small puzzles that have a known answer:

1. **Eight configurations in parallel** (Haiku and Sonnet × default / enabled / disabled / effort levels / display).
2. **`supportedModels()`** on a prompt that never yields, then **`fallbackModel`** with a model that doesn't exist.
3. **One live session**, with `applyFlagSettings`, `setMaxThinkingTokens` and `setModel` between turns.

They showed that the default `display` gives empty thinking text, that Haiku thinks by default but has no effort,
that Sonnet 5 ignores a fixed `budgetTokens`, that the Stop hook reports the applied effort, and that repeating a
question in a session makes the model answer from the conversation. Each finding became a scenario or a hint (the
full table is in Step 2 of the Tab).

## Step 5: Design the concept

- **Part A as columns.** One `/run` per column, all started at the same time, so time and cost are comparable. Six
  preset comparisons, each column editable, up to four columns.
- **Show the effort that was really applied**, not only the option: a `Stop` hook sends an `effort` SSE event.
- **Read the thinking live**: `includePartialMessages: true` and the `thinking_delta` events.
- **Part B without a conversation**: `supportedModels()` on a `query()` whose prompt never yields, cached by the server.
- **Part C on the Concept 12 session pattern**, one route per control, a different preset question for each turn.
- **The same safety setup as before**: `tools: []`, `settingSources: []`, `strictMcpConfig: true`. Haiku and Sonnet in
  the presets (Opus only in *Models*), to keep the cost low.

## Step 6: Implement it

| File | What was done |
|---|---|
| `server/concepts/14-thinking-effort-models.ts` | New: `effortReporter()`, `/run`, `/models` (+ `loadModels()`), the session and control routes |
| `server/index.ts` | Mounted on `/api/c14` |
| `src/concepts/Concept14ThinkingEffortModels.tsx` | New: `Compare` + `ConfigEditor` (Part A), `Models` (Part B), `LiveSession` + `SessionTimeline` (Part C), shared `RunCard` |
| `src/App.tsx`, `src/styles.css` | The tab; comparison grid, thinking text, effort badge |

`npx tsc --noEmit -p .` passed, and `npx vite build` succeeded.

## Step 7: Test the routes

Only the Concept 14 router was mounted in a scratch server on port **3013** and driven by a Node script, like the
browser does:

| Test | Result |
|---|---|
| `GET /models` twice | 5 models; Haiku without effort; the second call `cached` in 0 ms |
| Bad model, with and without `fallbackModel` | `result/success` + `is_error: true` + a thrown loop, vs. `system/model_fallback` and a Haiku answer |
| *Thinking on / off*, *display* (Haiku) | Thinking tokens 0 to 781; `omitted` = empty text but 566 thinking tokens |
| *Effort ladder* (Sonnet, clock puzzle) | Hook `low` / `medium` / `high` / `max`; only `max` thought (144 tokens) |
| Live session, 6 turns | Every control applied from the next turn; `setMaxThinkingTokens(0)` stopped thinking even at `max`; 409 after the end |

One change came from the tests: the `system/model_fallback` card first guessed the field names. The real message has
`trigger`, `original_model`, `fallback_model` and `content`, and the card now shows those.

Costs: $0.0027 to $0.0056 per Part A column, $0.0032 for the fallback run, nothing for `supportedModels()`.

## Step 8: Run it in the real app

The real app (`npm run dev`) served the page and `/api/c14/models` and `/api/c14/run` through the Vite proxy (the
`Stop` hook's `effort` event arrived, then `done`). Port 5173 was taken by another process, so Vite used 5174. The dev
processes were stopped afterwards.

## Files added or changed

| File | Change |
|---|---|
| `server/concepts/14-thinking-effort-models.ts` | New: the Concept 14 routes |
| `server/index.ts` | Mounts `/api/c14` |
| `src/concepts/Concept14ThinkingEffortModels.tsx` | New: the Thinking, effort & models tab |
| `src/App.tsx`, `src/styles.css` | Tab, comparison grid, thinking text, effort badge |
| `Tab1-query().md` | Adds Concept 14 to the table |
| `Tab14-Thinking-effort-and-models.md` | Explanation of the concept |
| `Build-steps.md` | This file |
| `readme.md` | Same content as `Tab14-Thinking-effort-and-models.md` |
