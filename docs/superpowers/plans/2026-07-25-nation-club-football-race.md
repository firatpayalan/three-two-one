# Nation–Club Football Race Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Ship a 2-player browser football race game: room-code multiplayer, curated nation×club picks, 5 rounds of speed-scored player guessing.

**Architecture:** Vite + React client and Fastify + Socket.io server. The server owns rooms, timers, validation, and scores; clients send intents and render snapshots. Pure modules (`scoring`, `dataset`, `guessMatcher`, `matchEngine`) are unit-tested with an injectable clock so tests never wait real 3s/30s.

**Tech Stack:** TypeScript, Vite, React 19, Fastify, Socket.io, Vitest, npm workspaces (`shared`, `server`, `client`)

## Global Constraints

- Stack locked: Vite + React client; Node + Fastify + Socket.io server (per design spec)
- 5 rounds per match; 3s pre-reveal pause; 30s guess window
- Scoring: `points = round(100 × (1 − t / 30))` for correct guesses; both players may score
- Wrong non-empty guess = one-shot lockout for that round
- Curated JSON dataset only for v1 (no live API)
- Pick lists only include entities that participate in ≥1 valid nation×club pair
- Room capacity exactly 2; invite via room code; no matchmaking
- Ask the human before adding or upgrading any npm dependency beyond versions listed in Task 1
- Do not commit secrets; no `.env` with credentials required for local v1

## File Structure

```text
/
  package.json                          # workspaces root
  shared/
    package.json
    src/protocol.ts                     # socket event names + payload types
    src/index.ts
  server/
    package.json
    vitest.config.ts
    tsconfig.json
    data/football.json                  # curated dataset
    src/scoring.ts
    src/normalize.ts
    src/dataset.ts
    src/guessMatcher.ts
    src/roomCode.ts
    src/roomManager.ts
    src/matchEngine.ts
    src/clock.ts                        # injectable Now / Schedule helpers
    src/socketHandlers.ts
    src/index.ts
    src/__tests__/scoring.test.ts
    src/__tests__/normalize.test.ts
    src/__tests__/dataset.test.ts
    src/__tests__/guessMatcher.test.ts
    src/__tests__/roomManager.test.ts
    src/__tests__/matchEngine.test.ts
  client/
    package.json
    vite.config.ts
    tsconfig.json
    index.html
    src/main.tsx
    src/App.tsx
    src/socket.ts
    src/session.ts
    src/types.ts                        # re-export shared protocol types
    src/screens/HomeScreen.tsx
    src/screens/LobbyScreen.tsx
    src/screens/PickScreen.tsx
    src/screens/CountdownScreen.tsx
    src/screens/GuessScreen.tsx
    src/screens/RoundResultScreen.tsx
    src/screens/MatchEndScreen.tsx
    src/screens/ReconnectBanner.tsx
    src/styles.css
  README.md
```

---

### Task 1: Monorepo scaffold

**Files:**
- Create: `package.json`, `shared/package.json`, `shared/src/protocol.ts`, `shared/src/index.ts`, `server/package.json`, `server/tsconfig.json`, `server/vitest.config.ts`, `server/src/index.ts`, `client/package.json`, `client/tsconfig.json`, `client/vite.config.ts`, `client/index.html`, `client/src/main.tsx`, `client/src/App.tsx`, `client/src/styles.css`, `README.md`

**Interfaces:**
- Consumes: nothing
- Produces: npm workspaces runnable; `shared` exports protocol stubs; `server` starts HTTP on `:3001`; `client` Vite on `:5173` proxies `/socket.io` to server

- [ ] **Step 1: Create root workspace `package.json`**

```json
{
  "name": "three-two-one",
  "private": true,
  "workspaces": ["shared", "server", "client"],
  "scripts": {
    "dev": "npm run dev -w server & npm run dev -w client",
    "test": "npm run test -w server",
    "build": "npm run build -w shared && npm run build -w server && npm run build -w client"
  }
}
```

- [ ] **Step 2: Create `shared` package with protocol types**

`shared/package.json`:

```json
{
  "name": "@three-two-one/shared",
  "version": "0.0.1",
  "type": "module",
  "main": "./src/index.ts",
  "types": "./src/index.ts",
  "exports": { ".": "./src/index.ts" }
}
```

`shared/src/protocol.ts` (full event contract used by later tasks):

```ts
export type PlayerId = string;
export type RoomCode = string;
export type Role = "nation" | "club";

export type Phase =
  | "lobby"
  | "picking"
  | "countdown"
  | "guessing"
  | "round_result"
  | "match_end"
  | "paused_reconnect"
  | "abandoned";

export type PublicPlayer = {
  id: PlayerId;
  connected: boolean;
  score: number;
};

export type MatchSnapshot = {
  roomCode: RoomCode;
  phase: Phase;
  players: PublicPlayer[];
  round: number; // 1..5 when in match; 0 in lobby
  totalRounds: 5;
  you?: {
    playerId: PlayerId;
    role?: Role;
    locked?: boolean;
    correct?: boolean;
    roundPoints?: number;
  };
  pickOptions?: { id: string; name: string }[];
  revealed?: { nation: { id: string; name: string }; club: { id: string; name: string } };
  timerEndsAt?: number; // epoch ms
  sampleAnswer?: string;
  winnerId?: PlayerId | null; // null = draw
  message?: string;
};

export const ClientEvents = {
  createRoom: "createRoom",
  joinRoom: "joinRoom",
  reconnect: "reconnectRoom",
  startMatch: "startMatch",
  pickEntity: "pickEntity",
  submitGuess: "submitGuess",
  playAgain: "playAgain",
} as const;

export const ServerEvents = {
  snapshot: "snapshot",
  error: "error",
} as const;

export type CreateRoomAck = { ok: true; roomCode: RoomCode; playerId: PlayerId } | { ok: false; error: string };
export type JoinRoomAck = { ok: true; roomCode: RoomCode; playerId: PlayerId } | { ok: false; error: string };
```

`shared/src/index.ts`:

```ts
export * from "./protocol.js";
```

- [ ] **Step 3: Scaffold `server` with Fastify + Socket.io + Vitest**

Ask before install. Intended deps (exact install command):

```bash
npm init -w server -y
# then in server/package.json set type module and scripts; install:
npm install fastify @fastify/cors socket.io -w server
npm install -D typescript tsx vitest @types/node -w server
```

`server/src/index.ts` (minimal listen):

```ts
import Fastify from "fastify";
import cors from "@fastify/cors";
import { Server } from "socket.io";

const app = Fastify();
await app.register(cors, { origin: true });
app.get("/health", async () => ({ ok: true }));

const port = Number(process.env.PORT ?? 3001);
await app.listen({ port, host: "0.0.0.0" });

const io = new Server(app.server, { cors: { origin: true } });
io.on("connection", (socket) => {
  socket.emit("snapshot", { phase: "lobby", roomCode: "", players: [], round: 0, totalRounds: 5 });
});

console.log(`server on :${port}`);
```

`server/vitest.config.ts`:

```ts
import { defineConfig } from "vitest/config";
export default defineConfig({ test: { environment: "node" } });
```

- [ ] **Step 4: Scaffold `client` with Vite React TS**

Ask before install:

```bash
npm create vite@latest client -- --template react-ts
npm install socket.io-client -w client
```

Point Vite proxy in `client/vite.config.ts`:

```ts
import { defineConfig } from "vite";
import react from "@vitejs/plugin-react";

export default defineConfig({
  plugins: [react()],
  server: {
    port: 5173,
    proxy: { "/socket.io": { target: "http://localhost:3001", ws: true } },
  },
});
```

`client/src/App.tsx` temporary: render `<h1>Nation–Club Football Race</h1>`.

- [ ] **Step 5: Write root `README.md`**

Document: `npm install` at root, `npm run dev -w server` and `npm run dev -w client`, open two browser windows, health check `GET /health`.

- [ ] **Step 6: Verify scaffold**

Run:

```bash
npm install
npm run test -w server   # may be 0 tests — exit 0
npx tsx server/src/index.ts &
curl -s http://localhost:3001/health
```

Expected: `{"ok":true}`

- [ ] **Step 7: Commit**

```bash
git add package.json shared server client README.md package-lock.json
git commit -m "$(cat <<'EOF'
Scaffold monorepo for Nation–Club Football Race.

Add shared protocol types, Fastify+Socket.io server stub, and Vite React client.
EOF
)"
```

---

### Task 2: Scoring + normalize (pure)

**Files:**
- Create: `server/src/scoring.ts`, `server/src/normalize.ts`, `server/src/__tests__/scoring.test.ts`, `server/src/__tests__/normalize.test.ts`

**Interfaces:**
- Consumes: nothing
- Produces:
  - `scoreForElapsed(tSeconds: number, windowSeconds?: number): number`
  - `normalizeGuess(input: string): string`

- [ ] **Step 1: Write failing scoring tests**

```ts
// server/src/__tests__/scoring.test.ts
import { describe, it, expect } from "vitest";
import { scoreForElapsed } from "../scoring.js";

describe("scoreForElapsed", () => {
  it("gives 100 at t=0", () => {
    expect(scoreForElapsed(0)).toBe(100);
  });
  it("gives 50 at t=15", () => {
    expect(scoreForElapsed(15)).toBe(50);
  });
  it("gives 0 at t=30", () => {
    expect(scoreForElapsed(30)).toBe(0);
  });
  it("clamps below 0 to 100 and above window to 0", () => {
    expect(scoreForElapsed(-1)).toBe(100);
    expect(scoreForElapsed(31)).toBe(0);
  });
});
```

- [ ] **Step 2: Run tests — expect FAIL**

```bash
npm run test -w server -- src/__tests__/scoring.test.ts
```

Expected: FAIL (module not found / export missing)

- [ ] **Step 3: Implement scoring**

```ts
// server/src/scoring.ts
export function scoreForElapsed(tSeconds: number, windowSeconds = 30): number {
  const t = Math.min(Math.max(tSeconds, 0), windowSeconds);
  return Math.round(100 * (1 - t / windowSeconds));
}
```

- [ ] **Step 4: Write failing normalize tests + implement**

```ts
// server/src/__tests__/normalize.test.ts
import { describe, it, expect } from "vitest";
import { normalizeGuess } from "../normalize.js";

describe("normalizeGuess", () => {
  it("trims lowercases and strips accents/punctuation", () => {
    expect(normalizeGuess("  Erling Haaland! ")).toBe("erling haaland");
    expect(normalizeGuess("José")).toBe("jose");
  });
});
```

```ts
// server/src/normalize.ts
export function normalizeGuess(input: string): string {
  return input
    .normalize("NFD")
    .replace(/\p{M}/gu, "")
    .toLowerCase()
    .replace(/[^\p{L}\p{N}\s]/gu, "")
    .replace(/\s+/g, " ")
    .trim();
}
```

- [ ] **Step 5: Run tests — expect PASS**

```bash
npm run test -w server -- src/__tests__/scoring.test.ts src/__tests__/normalize.test.ts
```

Expected: all PASS

- [ ] **Step 6: Commit**

```bash
git add server/src/scoring.ts server/src/normalize.ts server/src/__tests__
git commit -m "$(cat <<'EOF'
Add speed scoring and guess normalization.

Pure helpers for the 30s point curve and accent-insensitive matching.
EOF
)"
```

---

### Task 3: Curated dataset

**Files:**
- Create: `server/data/football.json`, `server/src/dataset.ts`, `server/src/__tests__/dataset.test.ts`

**Interfaces:**
- Consumes: JSON shape from spec
- Produces:
  - `loadDataset(path?: string): FootballDataset`
  - `listPickableNations(ds): {id,name}[]`
  - `listPickableClubs(ds): {id,name}[]`
  - `playersForPair(ds, nationId, clubId): PlayerRecord[]`
  - `assertPairInvariant(ds): void` (throws if any exposed nation/club lacks a pair partner with ≥1 player)

- [ ] **Step 1: Write minimal `football.json` (≥6 players spanning overlapping nations/clubs)**

Include at least: Norway + Manchester City → Erling Haaland (aliases include `Haaland`); France + Real Madrid → one player; England + Arsenal → one player; ensure every listed nation/club appears in ≥1 valid pair.

- [ ] **Step 2: Write failing dataset tests**

```ts
import { describe, it, expect } from "vitest";
import { loadDataset, listPickableNations, listPickableClubs, playersForPair, assertPairInvariant } from "../dataset.js";
import path from "node:path";
import { fileURLToPath } from "node:url";

const dataPath = path.join(path.dirname(fileURLToPath(import.meta.url)), "../../data/football.json");

describe("dataset", () => {
  const ds = loadDataset(dataPath);

  it("exposes only nations that participate in a valid pair", () => {
    assertPairInvariant(ds);
    expect(listPickableNations(ds).some((n) => n.id === "nor")).toBe(true);
  });

  it("finds Haaland for Norway × Manchester City", () => {
    const players = playersForPair(ds, "nor", "mci");
    expect(players.map((p) => p.id)).toContain("haaland");
  });
});
```

- [ ] **Step 3: Run — expect FAIL; implement `dataset.ts`; re-run PASS**

Implementation sketch:

```ts
export type PlayerRecord = {
  id: string;
  name: string;
  aliases: string[];
  nationId: string;
  clubIds: string[];
};

export type FootballDataset = {
  nations: { id: string; name: string }[];
  clubs: { id: string; name: string }[];
  players: PlayerRecord[];
};

export function loadDataset(filePath: string): FootballDataset {
  // readFileSync + JSON.parse
}

export function playersForPair(ds: FootballDataset, nationId: string, clubId: string): PlayerRecord[] {
  return ds.players.filter((p) => p.nationId === nationId && p.clubIds.includes(clubId));
}

export function listPickableNations(ds: FootballDataset) {
  return ds.nations.filter((n) =>
    ds.clubs.some((c) => playersForPair(ds, n.id, c.id).length > 0),
  );
}

export function listPickableClubs(ds: FootballDataset) {
  return ds.clubs.filter((c) =>
    ds.nations.some((n) => playersForPair(ds, n.id, c.id).length > 0),
  );
}

export function assertPairInvariant(ds: FootballDataset) {
  for (const n of listPickableNations(ds)) {
    const ok = listPickableClubs(ds).some((c) => playersForPair(ds, n.id, c.id).length > 0);
    if (!ok) throw new Error(`nation ${n.id} has no valid club pair`);
  }
  // mirror check for clubs
}
```

- [ ] **Step 4: Commit**

```bash
git add server/data/football.json server/src/dataset.ts server/src/__tests__/dataset.test.ts
git commit -m "$(cat <<'EOF'
Add curated football dataset with pick-list invariant.

Guarantees every exposed nation/club has at least one linking player.
EOF
)"
```

---

### Task 4: Guess matcher

**Files:**
- Create: `server/src/guessMatcher.ts`, `server/src/__tests__/guessMatcher.test.ts`

**Interfaces:**
- Consumes: `normalizeGuess`, `playersForPair`, `FootballDataset`
- Produces: `isCorrectGuess(ds, nationId, clubId, rawGuess: string): boolean`

- [ ] **Step 1: Write failing tests**

```ts
import { describe, it, expect } from "vitest";
import { loadDataset } from "../dataset.js";
import { isCorrectGuess } from "../guessMatcher.js";
// same dataPath as Task 3

describe("isCorrectGuess", () => {
  const ds = loadDataset(dataPath);
  it("accepts full name and alias for Haaland", () => {
    expect(isCorrectGuess(ds, "nor", "mci", "Erling Haaland")).toBe(true);
    expect(isCorrectGuess(ds, "nor", "mci", "haaland")).toBe(true);
  });
  it("rejects wrong player", () => {
    expect(isCorrectGuess(ds, "nor", "mci", "Messi")).toBe(false);
  });
});
```

- [ ] **Step 2: Implement**

```ts
import { normalizeGuess } from "./normalize.js";
import { FootballDataset, playersForPair } from "./dataset.js";

export function isCorrectGuess(
  ds: FootballDataset,
  nationId: string,
  clubId: string,
  rawGuess: string,
): boolean {
  const guess = normalizeGuess(rawGuess);
  if (!guess) return false;
  return playersForPair(ds, nationId, clubId).some((p) => {
    const names = [p.name, ...p.aliases].map(normalizeGuess);
    return names.includes(guess);
  });
}
```

- [ ] **Step 3: Run tests PASS; commit**

```bash
npm run test -w server -- src/__tests__/guessMatcher.test.ts
git add server/src/guessMatcher.ts server/src/__tests__/guessMatcher.test.ts
git commit -m "$(cat <<'EOF'
Add nation×club guess matcher with alias support.

EOF
)"
```

---

### Task 5: Room manager

**Files:**
- Create: `server/src/roomCode.ts`, `server/src/roomManager.ts`, `server/src/__tests__/roomManager.test.ts`

**Interfaces:**
- Consumes: nothing (in-memory Map)
- Produces:
  - `createRoomCode(): string` — 4 chars from `ABCDEFGHJKLMNPQRSTUVWXYZ23456789`
  - `RoomManager` class:
    - `create(socketId: string): { roomCode, playerId }`
    - `join(roomCode, socketId): { playerId } | throws RoomFullError | RoomNotFoundError`
    - `reconnect(roomCode, playerId, socketId): void`
    - `disconnect(socketId): { roomCode, playerId } | null`
    - `get(roomCode): Room | undefined`
    - `getBySocket(socketId): { room, playerId } | undefined`

```ts
export type Seat = {
  playerId: string;
  socketId: string | null;
  connected: boolean;
  score: number;
};

export type Room = {
  code: string;
  seats: [Seat, Seat | null]; // second null until join
  // matchEngine attached in Task 6 — keep `engine?: MatchEngine` optional for now
};
```

- [ ] **Step 1: Write failing roomManager tests**

Cover: create → join success; third join throws full; unknown code throws; disconnect marks seat disconnected; reconnect same `playerId` restores `socketId`.

- [ ] **Step 2: Implement `roomCode.ts` + `roomManager.ts`; tests PASS**

- [ ] **Step 3: Commit**

```bash
git commit -m "$(cat <<'EOF'
Add in-memory room manager with 2-seat reconnect.

EOF
)"
```

---

### Task 6: Match engine (state machine + injectable clock)

**Files:**
- Create: `server/src/clock.ts`, `server/src/matchEngine.ts`, `server/src/__tests__/matchEngine.test.ts`

**Interfaces:**
- Consumes: `FootballDataset`, `scoreForElapsed`, `isCorrectGuess`, `listPickableNations/Clubs`
- Produces: `MatchEngine` driving phases for one room

```ts
export type Clock = {
  now(): number;
  setTimeout(fn: () => void, ms: number): { clear(): void };
};

export function realClock(): Clock;
export function manualClock(): Clock & { advance(ms: number): void };

export class MatchEngine {
  constructor(opts: {
    dataset: FootballDataset;
    clock: Clock;
    totalRounds?: number; // default 5
    countdownMs?: number; // default 3000
    guessMs?: number; // default 30000
    onChange: (snapshotForBroadcast: Omit<MatchSnapshot, "you">, secrets: Map<PlayerId, Partial<MatchSnapshot["you"]>>) => void;
  });

  start(playerIds: [PlayerId, PlayerId]): void;
  pick(playerId: PlayerId, entityId: string): void;
  submitGuess(playerId: PlayerId, text: string): void;
  playAgain(): void;
  pauseForDisconnect(playerId: PlayerId): void;
  resumeReconnect(playerId: PlayerId): void;
  forfeitIfStillDisconnected(playerId: PlayerId): void; // called after 30s pause
  getPublicState(): Omit<MatchSnapshot, "you">;
  viewFor(playerId: PlayerId): MatchSnapshot;
}
```

**Phase transitions:**

1. `start` → random roles → `picking` (each gets pickOptions for their role only; opponent pick hidden)
2. both picked → `countdown` with `timerEndsAt = now+3000`
3. countdown fires → set `revealed` → `guessing` with `timerEndsAt = now+30000`
4. on guess: empty ignored; wrong → lock; correct → set roundPoints via `scoreForElapsed((now - guessStartedAt)/1000)`, mark correct
5. guessing ends when timer fires **or** both players are terminal (correct or locked) → `round_result` with `sampleAnswer` = first valid player name; after short result delay (use `countdownMs` or 2000ms) → next round or `match_end`
6. `playAgain` → scores 0, round 0→ start again
7. disconnect → `paused_reconnect`; timeout → `abandoned` with `winnerId` = remaining player

- [ ] **Step 1: Write failing tests with `manualClock`**

Minimum cases:
1. Full happy path one round: pick → advance 3s → both guess correct at different times → higher points for earlier → round_result
2. Wrong guess locks; second empty guess ignored
3. After 5 rounds → match_end with winner
4. Disconnect pause then forfeit

Use tiny timings in tests: `countdownMs: 1000`, `guessMs: 5000`, `totalRounds: 2` where useful.

- [ ] **Step 2: Implement `clock.ts` + `matchEngine.ts` until PASS**

Keep file focused: no Socket.io imports inside `matchEngine.ts`.

- [ ] **Step 3: Commit**

```bash
git commit -m "$(cat <<'EOF'
Add match engine with injectable clock and round phases.

EOF
)"
```

---

### Task 7: Socket handlers + server wiring

**Files:**
- Create: `server/src/socketHandlers.ts`
- Modify: `server/src/index.ts`

**Interfaces:**
- Consumes: `RoomManager`, `MatchEngine`, `loadDataset`, shared `ClientEvents` / `ServerEvents`
- Produces: wired `io.on("connection")` handlers that ack create/join and emit personalized `snapshot`

**Handler rules:**
- `createRoom` / `joinRoom` / `reconnectRoom({ roomCode, playerId })` → persist mapping socket↔seat; emit `viewFor(playerId)` to that socket; emit public updates to room
- `startMatch` only if 2 connected seats and phase lobby/match_end/abandoned cleared
- On `disconnect`: `engine.pauseForDisconnect`; schedule 30s forfeit via clock; on reconnect clear forfeit
- Never broadcast secret picks; use `viewFor`

- [ ] **Step 1: Write integration test with `socket.io-client` hitting a test server**

Ask before adding `socket.io-client` as a **server** devDependency if not already present from root.

Test: create + join → 2 players in lobby snapshot → startMatch → both receive `picking` with opposite roles.

- [ ] **Step 2: Implement handlers; test PASS**

- [ ] **Step 3: Commit**

```bash
git commit -m "$(cat <<'EOF'
Wire Socket.io handlers to rooms and match engine.

EOF
)"
```

---

### Task 8: Client session + socket + App shell

**Files:**
- Create: `client/src/socket.ts`, `client/src/session.ts`, `client/src/types.ts`
- Modify: `client/src/App.tsx`, `client/src/main.tsx`, `client/src/styles.css`

**Interfaces:**
- Consumes: shared protocol; `socket.io-client`
- Produces:
  - `saveSession({ roomCode, playerId })` / `loadSession()` / `clearSession()` via `sessionStorage`
  - `connectSocket(): Socket` to `window.location.origin` (Vite proxy)
  - `App` holds `snapshot: MatchSnapshot | null` and routes by `phase`

- [ ] **Step 1: Implement session + socket helpers**

```ts
// client/src/session.ts
const KEY = "ncfr-session";
export type Session = { roomCode: string; playerId: string };
export function saveSession(s: Session) {
  sessionStorage.setItem(KEY, JSON.stringify(s));
}
export function loadSession(): Session | null {
  const raw = sessionStorage.getItem(KEY);
  return raw ? (JSON.parse(raw) as Session) : null;
}
export function clearSession() {
  sessionStorage.removeItem(KEY);
}
```

- [ ] **Step 2: App routes phases to placeholder screens**

```tsx
switch (snapshot?.phase) {
  case undefined:
  case "lobby":
    return players < 2 && !roomCode ? <HomeScreen /> : <LobbyScreen />;
  case "picking": return <PickScreen />;
  case "countdown": return <CountdownScreen />;
  case "guessing": return <GuessScreen />;
  case "round_result": return <RoundResultScreen />;
  case "match_end":
  case "abandoned": return <MatchEndScreen />;
  case "paused_reconnect": return <><ReconnectBanner /><LobbyScreen /></>;
}
```

(Adjust lobby detection using snapshot fields precisely once wired.)

On mount: if `loadSession()`, emit `reconnectRoom`.

- [ ] **Step 3: Basic sports-themed CSS variables (avoid purple/cream AI defaults)** — pitch-green depth gradient, bold display font from Google Fonts (e.g. "Archivo Black" + "Source Sans 3")

- [ ] **Step 4: Manual smoke — two windows see Home; commit**

```bash
git commit -m "$(cat <<'EOF'
Add client socket session and phase-based App shell.

EOF
)"
```

---

### Task 9: Home + Lobby screens

**Files:**
- Create: `client/src/screens/HomeScreen.tsx`, `client/src/screens/LobbyScreen.tsx`
- Modify: `client/src/App.tsx`

**Interfaces:**
- Consumes: socket emitters `createRoom`, `joinRoom`, `startMatch`; snapshot
- Produces: working create/join/start UX

- [ ] **Step 1: HomeScreen** — Create button; Join form with code input (uppercase); on ack success `saveSession` and wait for snapshots

- [ ] **Step 2: LobbyScreen** — show room code prominently, copy helper, player connection dots (1/2), Start enabled when `players.filter(p => p.connected).length === 2`

- [ ] **Step 3: Manual test two browsers; commit**

```bash
git commit -m "$(cat <<'EOF'
Add home and lobby screens for room create/join/start.

EOF
)"
```

---

### Task 10: Pick, Countdown, Guess screens

**Files:**
- Create: `client/src/screens/PickScreen.tsx`, `client/src/screens/CountdownScreen.tsx`, `client/src/screens/GuessScreen.tsx`, `client/src/screens/ReconnectBanner.tsx`

**Interfaces:**
- Consumes: `pickEntity`, `submitGuess`, snapshot fields `you.role`, `pickOptions`, `revealed`, `timerEndsAt`, `you.locked`, `you.correct`

- [ ] **Step 1: PickScreen** — filterable list; on select emit `pickEntity`; show waiting state after local pick (derive from absence of further options or a `you.picked` flag — if missing, add `you.hasPicked?: boolean` to protocol in `shared` and engine)

If protocol needs `hasPicked`, update `shared/src/protocol.ts` + `MatchEngine.viewFor` in this task (keep backward-compatible field optional).

- [ ] **Step 2: CountdownScreen** — display seconds remaining from `timerEndsAt - Date.now()`; no entity names yet

- [ ] **Step 3: GuessScreen** — show revealed nation + club; countdown; text input + submit; disable when locked/correct/timeout; show feedback

- [ ] **Step 4: Manual two-browser round through guessing; commit**

```bash
git commit -m "$(cat <<'EOF'
Add pick, countdown, and guess screens.

EOF
)"
```

---

### Task 11: Round result, match end, play again

**Files:**
- Create: `client/src/screens/RoundResultScreen.tsx`, `client/src/screens/MatchEndScreen.tsx`

**Interfaces:**
- Consumes: round points, totals, `sampleAnswer`, `winnerId`, `playAgain`

- [ ] **Step 1: RoundResultScreen** — this-round points per player, totals, sample answer line

- [ ] **Step 2: MatchEndScreen** — winner / draw / forfeit copy; Play again button emits `playAgain`

- [ ] **Step 3: Full 5-round manual match; commit**

```bash
git commit -m "$(cat <<'EOF'
Add round result and match end with rematch.

EOF
)"
```

---

### Task 12: Disconnect / refresh hardening + README polish

**Files:**
- Modify: `server/src/socketHandlers.ts`, `client/src/App.tsx`, `client/src/screens/ReconnectBanner.tsx`, `README.md`
- Test: extend `server/src/__tests__/matchEngine.test.ts` or socket integration for reconnect

**Interfaces:**
- Consumes: existing reconnect APIs
- Produces: refresh restores seat; opponent sees pause banner; 30s forfeit works

- [ ] **Step 1: Automated test** — disconnect seat → phase `paused_reconnect` → reconnect → resume; separate test advance 30s → `abandoned`

- [ ] **Step 2: Client ReconnectBanner copy: “Opponent reconnecting…”

- [ ] **Step 3: README** — local two-player instructions, architecture blurb, link to design spec

- [ ] **Step 4: Commit**

```bash
git commit -m "$(cat <<'EOF'
Harden reconnect/forfeit flow and document local play.

EOF
)"
```

---

## Spec coverage checklist (self-review)

| Spec requirement | Task |
| --- | --- |
| Online room code pairing, 2-player cap | 5, 7, 9 |
| Random nation/club roles each round | 6 |
| Secret pick → 3s → reveal → 30s guess | 6, 10 |
| Speed curve scoring; both can score | 2, 6 |
| One-shot wrong lockout; empty ignored | 6 |
| Curated dataset + pick invariant | 3 |
| Alias-normalized matching | 2, 4 |
| 5 rounds + winner/draw + play again | 6, 11 |
| Disconnect pause ~30s + forfeit | 6, 12 |
| Refresh reconnect via sessionStorage | 8, 12 |
| React + Fastify + Socket.io | 1, 7, 8 |
| Unit + integration tests | 2–7, 12 |

## Execution Handoff

Plan complete and saved to `docs/superpowers/plans/2026-07-25-nation-club-football-race.md`. Two execution options:

**1. Subagent-Driven (recommended)** — dispatch a fresh subagent per task, review between tasks, fast iteration

**2. Inline Execution** — execute tasks in this session using executing-plans, batch execution with checkpoints

Which approach?
