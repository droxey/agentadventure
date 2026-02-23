# CLAUDE.md — AgentAdventure

## Project Summary

AgentAdventure is an **OpenClaw skill** that places AI agents into self-hosted **WorkAdventure v1.28.9** as real avatars. Each agent runs inside a headless Chromium browser controlled by Playwright, interacting via the WA Scripting API for movement, proximity chat, and experimental voice (STT/TTS). No WorkAdventure backend modifications are required.

**Current state:** Planning/specification repository. The source code (`runner.ts`, `bridge.ts`, `voice.ts`, `utils.ts`) has not been implemented yet — only pseudo-code exists in `PLAN.md` and `.github/copilot-instructions.md`.

## Repository Structure

```
agentadventure/
├── README.md                       # Feature overview, architecture diagrams, quick start, troubleshooting
├── PLAN.md                         # Detailed integration plan: architecture, pseudo-code, risks, phased TODOs
├── LICENSE                         # Apache 2.0
├── CLAUDE.md                       # This file
└── .github/
    └── copilot-instructions.md     # API references, code patterns, common mistakes
```

### Planned Implementation Structure (to be created)

```
~/.openclaw/skills/agentadventure/
├── SKILL.md          # OpenClaw skill definition (YAML frontmatter + instructions)
├── runner.ts         # Playwright session: launch, anonymous login, lifecycle, retry
├── bridge.ts         # Event bridge: WA Scripting API <-> OpenClaw agent logic
├── voice.ts          # Voice pipeline: listenToAudioStream -> STT -> LLM -> TTS -> startAudioStream
├── utils.ts          # Shared helpers: retryOp, parseCoords, getMessage, rate limiting
└── __tests__/
    ├── runner.test.ts
    ├── bridge.test.ts
    └── voice.test.ts
```

## Tech Stack

| Layer              | Technology              | Notes                                     |
|--------------------|-------------------------|-------------------------------------------|
| Runtime            | Node.js 20+, TypeScript | OpenClaw is Node-based                    |
| Browser automation | Playwright ^1.42.0      | Headless Chromium per agent               |
| Virtual office     | WorkAdventure v1.28.9   | Docker Compose, anonymous login (no OIDC) |
| Chat fallback      | Matrix (Synapse)        | WA native bridge; OpenClaw Matrix channel |
| Voice              | ElevenLabs / Deepgram   | STT/TTS via OpenClaw voice-call skill     |
| Audio transport    | LiveKit (WA built-in)   | WebRTC; Float32Array buffers              |
| Testing            | Vitest                  | Unit + E2E                                |
| Containerization   | Docker, Docker Compose  | WA infra + optional agent sandbox         |

## Key Architecture

- **Skill entry point:** `SKILL.md` with YAML frontmatter (name, description, requires, install)
- **Session lifecycle:** Playwright opens headless Chromium -> navigates to WA URL -> anonymous login (display name + Woka picker) -> waits for canvas
- **Event bridge:** `page.exposeFunction('onWAEvent', ...)` bridges browser callbacks to Node runtime; `page.evaluate()` executes WA Scripting API calls
- **Command flow (outbound):** Agent logic -> OpenClaw gateway -> skill runner -> `page.evaluate()` -> WA action
- **Event flow (inbound):** WA event fires -> injected listener calls `window.onWAEvent()` -> Playwright bridges to gateway -> agent processes
- **Voice pipeline:** `listenToAudioStream` (Float32Array) -> STT -> LLM -> TTS -> `startAudioStream`/`appendAudioData`
- **Error recovery:** Retry (3 attempts) -> fallback (voice->text) -> restart (browser crash->new session)

## Development Workflow

### Prerequisites

- Docker and Docker Compose
- Node.js v20+
- Git

### Setup

```bash
# 1. Deploy WorkAdventure
git clone https://github.com/workadventure/workadventure.git && cd workadventure
cp .env.template .env
docker-compose -f docker-compose.yaml -f docker-compose-no-oidc.yaml up -d

# 2. Install OpenClaw
npm install -g openclaw@latest
openclaw gateway start

# 3. Set up skill folder
mkdir -p ~/.openclaw/skills/agentadventure
cd ~/.openclaw/skills/agentadventure && npx playwright install chromium

# 4. Verify
openclaw skills list --eligible
```

### Configuration

All config lives in `~/.openclaw/openclaw.json` under `skills.entries.agentadventure`:

```json
{
  "skills": {
    "entries": {
      "agentadventure": {
        "enabled": true,
        "env": {
          "WA_URL": "http://play.workadventure.localhost/",
          "WA_BOT_NAME": "AgentBot"
        }
      }
    }
  }
}
```

Skills are snapshotted at gateway start — restart the gateway after any changes.

### Testing

- **Unit:** Vitest on `retryOp`, `parseCoords`, `getMessage`, event handler logic
- **E2E:** Playwright against real WA Docker instance
- **Voice E2E:** Fake audio stream -> STT -> LLM -> TTS -> verify playback
- **Error E2E:** Kill browser mid-session -> verify auto-restart and recovery logs
- **Performance targets:** <20% CPU/mem overhead per agent; voice latency <500ms

## Critical API Conventions

### WorkAdventure Scripting API

**Chat — always use options object, NOT the deprecated 2-arg form:**

```typescript
// CORRECT
WA.chat.sendChatMessage('Hello', { scope: 'bubble' });
WA.chat.onChatMessage((message, event) => { ... }, { scope: 'bubble' });

// WRONG — deprecated
WA.chat.sendChatMessage('Hello', 'BotName');
```

**Proximity — there is NO `onLeave()` on `proximityMeeting`:**

```typescript
// Bubble lifecycle
WA.player.proximityMeeting.onJoin().subscribe((users) => { ... });

// Player tracking (requires explicit opt-in)
await WA.players.configureTracking({ players: true, movement: false });
WA.players.onPlayerEnters.subscribe((player) => { ... });
WA.players.onPlayerLeaves.subscribe((player) => { ... });
```

**Movement:**

```typescript
await WA.player.moveTo(x, y, speed?);
const pos = await WA.player.getPosition();
```

**Voice (experimental — verify sampleRate from WA source before hardcoding):**

```typescript
WA.player.proximityMeeting.listenToAudioStream().subscribe((buffer) => { ... });
const stream = WA.player.proximityMeeting.startAudioStream();
stream.appendAudioData(float32Buffer);
```

### OpenClaw Skill System

- Skills are `SKILL.md` folders — there is **no** `plugin.json`
- Config goes in `openclaw.json` -> `skills.entries.<name>` with `enabled`, `env`, `apiKey`
- Install from registry: `clawdhub install <name>` (NOT `openclaw skills install`)
- Verify eligibility: `openclaw skills list --eligible`
- There is **no** `openclaw init workspace`, `openclaw agents deploy`, or `openclaw plugins install`

### Playwright Patterns

```typescript
// Required launch args for WebRTC/audio in headless mode
chromium.launch({
  headless: true,
  args: ['--use-fake-device-for-media-stream', '--enable-webrtc'],
});

// Bridge pattern
await page.exposeFunction('onWAEvent', (type, data) => { ... });
await page.evaluate(() => {
  WA.chat.onChatMessage((msg, evt) => window.onWAEvent('chat', { msg, evt }), { scope: 'bubble' });
});
```

### WA Anonymous Login Flow

WA with `docker-compose-no-oidc.yaml` uses anonymous entry. There are **no** `#username` / `#password` / `#login` selectors. The flow is:

1. Text input -> enter display name -> press Enter
2. Woka avatar picker -> click submit/confirm button
3. Wait for `canvas` element (game loaded)

Selectors are approximate and must be verified against the running WA version's DOM.

## Common Mistakes to Avoid

| Don't | Do |
|-------|-----|
| `WA.chat.sendChatMessage('hi', 'Bot')` | `WA.chat.sendChatMessage('hi', { scope: 'bubble' })` |
| `WA.player.proximityMeeting.onLeave()` | `WA.players.onPlayerLeaves.subscribe(...)` |
| `WA.player.proximityMeeting.onParticipantJoin()` | `WA.players.onPlayerEnters.subscribe(...)` |
| `page.fill('#username', ...)` | `page.fill('input[type="text"]', botName)` |
| `openclaw skills install X` | `clawdhub install X` or manual skill folder |
| `openclaw plugins install voice` | `clawdhub install voice-call` or bundled skill config |
| `openclaw agents deploy` | `openclaw gateway start` (gateway manages agents) |
| `plugin.json` in skill folder | `openclaw.json` -> `skills.entries` for config |
| Hardcode `sampleRate: 48000` | Verify from WA source; blog says 24kHz PCM16 |
| Store secrets in SKILL.md | Use `skills.entries.*.env` in `openclaw.json` |

## Code Style & Patterns

### Error Handling

Every operation uses the `retryOp` wrapper (3 retries, log each attempt):

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

Voice failure -> text chat. Browser crash -> auto-restart session. All errors are non-fatal and logged.

### Security Rules

- Never log PII (usernames, messages) in production — use `[REDACTED]`
- Sanitize all `page.evaluate` inputs — no raw string interpolation
- Rate-limit `exposeFunction` callbacks (10 calls/sec per agent)
- Keep API keys in `openclaw.json` env, never in code or SKILL.md
- Run Playwright as non-root in Docker; enable seccomp

## Key Reference Documents

| Document | Purpose |
|----------|---------|
| `README.md` | Feature overview, architecture diagrams, quick start guide, troubleshooting |
| `PLAN.md` | Full integration plan: architecture, pseudo-code snippets, risk matrix, phased TODOs, security audit checklist, deployment guide |
| `.github/copilot-instructions.md` | API references, correct WA Scripting API signatures, OpenClaw skill patterns, Playwright config, common mistakes |

## External References

- [WA Scripting API — Chat](https://docs.workadventu.re/developer/map-scripting/references/api-chat/)
- [WA Scripting API — Player](https://docs.workadventu.re/developer/map-scripting/references/api-player/)
- [WA Scripting API — Players (tracking)](https://docs.workadventu.re/developer/map-scripting/references/api-players/)
- [WA ChatGPT Bot Tutorial](https://docs.workadventu.re/blog/gpt-bot/)
- [WA Realtime API (voice)](https://docs.workadventu.re/blog/realtime-api/)
- [OpenClaw Skills Docs](https://docs.openclaw.ai/tools/skills)
- [Playwright API](https://playwright.dev/docs/api/class-page)
