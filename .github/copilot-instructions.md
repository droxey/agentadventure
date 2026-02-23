# Copilot Instructions — AgentAdventure

## Project Overview

AgentAdventure is an **OpenClaw skill** that places AI agents into self-hosted **WorkAdventure v1.28.9** as real avatars. Agents join via Playwright headless browser automation, interact through the WA Scripting API, and support text chat + experimental voice (STT/TTS).

No WA backend modifications. Everything is a single OpenClaw skill folder at `~/.openclaw/skills/agentadventure/`.

## Tech Stack

|Layer               |Technology             |Notes                                    |
|--------------------|-----------------------|-----------------------------------------|
|Runtime             |Node.js 20+, TypeScript|OpenClaw is Node-based                   |
|Browser automation  |Playwright ^1.42.0     |Headless Chromium per agent              |
|Virtual office      |WorkAdventure v1.28.9  |Docker Compose, anonymous login (no OIDC)|
|Chat fallback       |Matrix (Synapse)       |WA native bridge; OpenClaw Matrix channel|
|Voice (experimental)|ElevenLabs / Deepgram  |STT/TTS via OpenClaw voice-call skill    |
|Audio transport     |LiveKit (WA built-in)  |WebRTC; Float32Array buffers             |
|Testing             |Vitest                 |Unit + E2E                               |
|Containerization    |Docker, Docker Compose |WA infra + optional agent sandbox        |

## File Structure

```
~/.openclaw/skills/agentadventure/
├── SKILL.md          # OpenClaw skill definition (YAML frontmatter + instructions)
├── runner.ts         # Playwright session: launch, anonymous login, lifecycle
├── bridge.ts         # Event bridge: WA Scripting API ↔ OpenClaw agent logic
├── voice.ts          # Voice pipeline: listenToAudioStream → STT → LLM → TTS → startAudioStream
├── utils.ts          # Shared: retryOp, parseCoords, getMessage, rate limiting
└── __tests__/
    ├── runner.test.ts
    ├── bridge.test.ts
    └── voice.test.ts
```

Configuration lives in `~/.openclaw/openclaw.json` under `skills.entries.agentadventure`, **not** in a `plugin.json` (which does not exist in OpenClaw).

## Critical API References

### WorkAdventure Scripting API (WA object)

These are client-side APIs available inside `page.evaluate()` blocks. Always use the **current** signatures — never deprecated ones.

#### Proximity (bubble lifecycle)

```typescript
// Observable — fires when bot enters a proximity bubble
WA.player.proximityMeeting.onJoin().subscribe((users) => { ... });

// Player tracking — requires explicit opt-in
await WA.players.configureTracking({ players: true, movement: false });
WA.players.onPlayerEnters.subscribe((player) => { ... });
WA.players.onPlayerLeaves.subscribe((player) => { ... });
```

> **⚠️ There is NO `WA.player.proximityMeeting.onLeave()`** — use player tracking observables instead.
> **⚠️ There is NO `onParticipantJoin` / `onParticipantLeave`** on `proximityMeeting`.

#### Chat (bubble scope)

```typescript
// Send — always use options object, NOT the deprecated 2-arg signature
WA.chat.sendChatMessage('Hello', { scope: 'bubble' });

// Listen — pass scope in options; filter out own messages via !event.author
WA.chat.onChatMessage((message, event) => {
  if (!event.author) return; // own message echo
  // handle...
}, { scope: 'bubble' });

// Typing indicators
WA.chat.startTyping({ scope: 'bubble' });
WA.chat.stopTyping({ scope: 'bubble' });
```

> **⚠️ DEPRECATED:** `WA.chat.sendChatMessage('text', 'authorName')` — do NOT use 2-arg form.

#### Movement

```typescript
// Returns Promise<{ x, y, cancelled }>
await WA.player.moveTo(x, y, speed?);
const pos = await WA.player.getPosition();
```

#### Voice (experimental — verify before hardcoding)

```typescript
// Listen to incoming audio in bubble — returns Observable of Float32Array
WA.player.proximityMeeting.listenToAudioStream().subscribe((buffer) => { ... });

// Send audio to bubble — returns object with appendAudioData()
const stream = WA.player.proximityMeeting.startAudioStream();
stream.appendAudioData(float32Buffer);
```

> **⚠️ Sample rate is TBD.** WA blog documents 24kHz PCM16 converted to Float32. Do NOT hardcode 48000 without verifying WA source.

### OpenClaw Skill System

```yaml
# SKILL.md frontmatter — required fields
---
name: agentadventure
description: <trigger-phrase-style description matching how users ask for the task>
metadata:
  openclaw:
    emoji: "🎮"
    requires:
      bins: ["npx"]      # binaries that must be on PATH for skill to be eligible
    install:
      - id: npm
        kind: node
        package: "playwright"
        bins: ["npx"]
        label: "Install Playwright via npm"
---
```

Key facts:

- Skills are SKILL.md folders, **not plugins**. No `plugin.json`.
- Config goes in `openclaw.json` → `skills.entries.<name>` with `enabled`, `env`, `apiKey`.
- Skills are snapshotted at gateway start; restart gateway after changes.
- Verify eligibility: `openclaw skills list --eligible`
- Install from registry: `clawdhub install <name>` (NOT `openclaw skills install`).
- There is **no** `openclaw init workspace`, `openclaw agents deploy`, or `openclaw plugins install` command.

### Playwright in This Project

```typescript
// Launch args required for WebRTC/audio in headless mode
chromium.launch({
  headless: true,
  args: ['--use-fake-device-for-media-stream', '--enable-webrtc'],
});

// Bridge pattern: expose a function, call it from injected evaluate scripts
await page.exposeFunction('onWAEvent', (type, data) => { ... });
await page.evaluate(() => {
  WA.chat.onChatMessage((msg, evt) => window.onWAEvent('chat', { msg, evt }), { scope: 'bubble' });
});
```

### WorkAdventure Anonymous Login Flow

WA with `docker-compose-no-oidc.yaml` uses anonymous entry. **There are no `#username` / `#password` / `#login` selectors.** The flow is:

1. Text input → enter display name → press Enter
1. Woka avatar picker → click submit/confirm button
1. Wait for `canvas` element (game loaded)

Selectors are approximate and must be verified against the running WA version’s DOM.

## Code Style & Patterns

### Error Handling

Every operation uses the `retryOp` wrapper pattern (3 retries, log each attempt):

```typescript
async function retryOp<T>(fn: () => Promise<T>, maxRetries = 3): Promise<T> {
  for (let i = 0; i < maxRetries; i++) {
    try { return await fn(); }
    catch (err) {
      console.error(`Retry ${i + 1}/${maxRetries}: ${err.message}`);
      if (i === maxRetries - 1) throw err;
    }
  }
  throw new Error('Unreachable');
}
```

### Fallback Chain

Voice failure → text chat. Browser crash → auto-restart session. All errors are non-fatal and logged.

### Security Rules

- Never log PII (usernames, messages) in production — use `[REDACTED]`
- Sanitize all `page.evaluate` inputs — no raw string interpolation
- Rate-limit `exposeFunction` callbacks (10 calls/sec per agent)
- Keep API keys in `openclaw.json` env, never in code or SKILL.md
- Run Playwright as non-root in Docker; enable seccomp

## Common Mistakes to Avoid

|❌ Don’t                                         |✅ Do                                                 |
|------------------------------------------------|-----------------------------------------------------|
|`WA.chat.sendChatMessage('hi', 'Bot')`          |`WA.chat.sendChatMessage('hi', { scope: 'bubble' })` |
|`WA.player.proximityMeeting.onLeave()`          |`WA.players.onPlayerLeaves.subscribe(...)`           |
|`WA.player.proximityMeeting.onParticipantJoin()`|`WA.players.onPlayerEnters.subscribe(...)`           |
|`page.fill('#username', ...)`                   |`page.fill('input[type="text"]', botName)`           |
|`openclaw skills install X`                     |`clawdhub install X` or manual skill folder          |
|`openclaw plugins install voice`                |`clawdhub install voice-call` or bundled skill config|
|`openclaw agents deploy`                        |`openclaw gateway start` (gateway manages agents)    |
|`plugin.json` in skill folder                   |`openclaw.json` → `skills.entries` for config        |
|Hardcode `sampleRate: 48000`                    |Verify from WA source; blog says 24kHz PCM16         |
|Store secrets in SKILL.md                       |Use `skills.entries.*.env` in `openclaw.json`        |

## Testing Strategy

- **Unit:** Vitest for `retryOp`, `parseCoords`, `getMessage`, event handler logic
- **E2E:** Playwright against real WA Docker instance; verify avatar appears, chat roundtrip works
- **Voice E2E:** Fake audio stream → STT → LLM → TTS → verify playback; test fallback to text on failure
- **Error E2E:** Kill browser mid-session → verify auto-restart; induce timeout → verify retry logs
- **Performance:** <20% CPU/mem overhead per agent; voice latency <500ms e2e

## Environment Setup (Codespaces)

```bash
# 1. WA infrastructure
git clone https://github.com/workadventure/workadventure.git
cd workadventure && cp .env.template .env
docker-compose -f docker-compose.yaml -f docker-compose-no-oidc.yaml up -d

# 2. /etc/hosts (or Codespaces port forwarding)
# play.workadventure.localhost, map-storage.workadventure.localhost, etc.

# 3. OpenClaw
npm install -g openclaw@latest
# First `openclaw gateway start` auto-creates workspace

# 4. Skill development
mkdir -p ~/.openclaw/skills/agentadventure
npx playwright install chromium
# Edit SKILL.md, runner.ts, bridge.ts
# Restart gateway to pick up skill changes

# 5. Verify
openclaw skills list --eligible  # should show agentadventure
```

## Reference Docs

- [WA Scripting API — Chat](https://docs.workadventu.re/developer/map-scripting/references/api-chat/)
- [WA Scripting API — Player](https://docs.workadventu.re/developer/map-scripting/references/api-player/)
- [WA Scripting API — Players (tracking)](https://docs.workadventu.re/developer/map-scripting/references/api-players/)
- [WA ChatGPT Bot Tutorial](https://docs.workadventu.re/blog/gpt-bot/)
- [WA Tock Bot Tutorial](https://docs.workadventu.re/blog/tock-bot/)
- [WA Realtime API (voice)](https://docs.workadventu.re/blog/realtime-api/)
- [OpenClaw Skills Docs](https://docs.openclaw.ai/tools/skills)
- [Playwright API](https://playwright.dev/docs/api/class-page)
