# Nation–Club Football Race — Design Spec

**Date:** 2026-07-25  
**Status:** Approved for planning  
**Product working title:** Nation–Club Football Race (browser, 2-player)

## Summary

A real-time browser game for two players. Each round, players are randomly assigned Nation or Club roles, secretly pick entities from a curated pool, wait 3 seconds, then race for 30 seconds to name a football player who links that nation and club. Faster correct answers score more. After 5 rounds, highest total score wins.

## Goals

- Fun, fair head-to-head race based on football knowledge and speed
- Works online: each player on their own device, joined via room code/link
- Authoritative server so timers and scores cannot be cheated from the client
- Ship v1 with a curated dataset; keep a clean path to enrich via a football API later

## Non-goals (v1)

- Global random matchmaking / public lobbies
- Peer-to-peer connections (WebRTC)
- Live football API lookups
- Persistence of rooms across server restarts
- More than 2 players, spectators, or ranked accounts
- Mobile-native apps (responsive web is enough)

## Decisions (locked)

| Topic | Choice |
| --- | --- |
| Play mode | Online, friend invite via room code/link |
| Roles | Random Nation vs Club each round |
| Answer source | Hybrid: curated dataset for v1; API later |
| Scoring | Speed curve: both can score if correct; faster = more points |
| Pick pool | Only entities that guarantee ≥1 valid linking player |
| Guess window | 30 seconds |
| Wrong guess | One shot — wrong locks the player out for that round |
| Stack | Vite + React client; Node + Fastify + Socket.io server |

## Architecture

```text
┌─────────────────┐         WebSocket          ┌──────────────────────────┐
│  React (Vite)   │ ───────────────────────────│  Node game server        │
│  Player A       │                            │  Socket.io + room state  │
└─────────────────┘                            │  timers, scoring,        │
                                               │  answer validation       │
┌─────────────────┐         WebSocket          │  curated JSON dataset    │
│  React (Vite)   │ ───────────────────────────│                          │
│  Player B       │                            └──────────────────────────┘
└─────────────────┘
```

Players never connect to each other. Both open an outbound WebSocket to the shared game server (public URL in production; `localhost` in development). The server relays room events and owns all authoritative state.

### Why this shape

- Fair timers and scores require a clock and validator neither client controls
- Socket.io fits room membership, reconnect, and event fan-out
- Separating Vite client from Node (Fastify) keeps real-time hosting simple

## Lobby & pairing

1. Player A **creates** a room → server returns a short `roomCode` (and optionally a shareable URL embedding that code).
2. Player B **joins** with the code.
3. Room capacity is exactly **2**. Extra joins are rejected with a clear error.
4. When both are connected, either player may **Start** the match.
5. Match runs **5 rounds**, then shows the winner.

No matchmaking queue: pairing is invite-only.

## Round loop

Repeated 5 times:

1. **Assign roles (random)** — one player is Nation picker, the other Club picker.
2. **Secret pick** — each selects from their curated list. Choices stay hidden until both have picked.
3. **3-second break** — shared countdown; entities still hidden.
4. **Reveal** — nation and club shown to both.
5. **Guess (30s)** — text input to name a linking player.
   - Correct → award speed-curve points (both players may score in the same round).
   - Wrong (non-empty) → that player is locked out for the round (0 points).
   - Empty submit → ignored (does not lock).
   - Timeout → no further submits; keep whatever points were earned.
6. **Round result** — show this-round points, running totals, optionally one sample valid answer from the dataset; then next round or match end.

### Scoring formula

On a correct guess at elapsed time `t` seconds (0 ≤ t ≤ 30):

```text
points = round(100 × (1 − t / 30))
```

If correct with essentially t = 0 → 100. If correct near t = 30 → approaches 0. Incorrect or timeout without a correct guess → 0 for that player.

Exact tuning (e.g. a small minimum for any in-time correct answer) may change during implementation without changing the “faster correct = more points; both can score” rule.

### Match end

After round 5, the player with the higher total score wins. Ties are allowed (declare a draw). Offer **Play again**: reset scores and round counter in the same room and start a new 5-round match (roles still re-randomized each round).

## Data model (curated v1)

Server-owned JSON (shape illustrative):

```json
{
  "nations": [{ "id": "nor", "name": "Norway" }],
  "clubs": [{ "id": "mci", "name": "Manchester City" }],
  "players": [
    {
      "id": "haaland",
      "name": "Erling Haaland",
      "aliases": ["Haaland", "Erling Braut Haaland"],
      "nationId": "nor",
      "clubIds": ["mci"]
    }
  ]
}
```

**Invariant:** Every nation and club exposed in the pick UI participates in at least one `(nationId, clubId)` pair that has ≥1 player in the dataset. The pick UI only offers those entities, so every completed pair has at least one accepted answer.

**Guess validation (server):**

1. Normalize input: trim, lowercase, strip accents/punctuation.
2. Accept if it matches the display name or any alias of a player linked to **both** the revealed nation and club.
3. First correct for a player awards points once; further guesses from a locked or already-correct player are ignored.

**Future:** Replace or enrich the dataset via a football API behind the same validation interface; clients do not depend on storage details.

## Client screens

1. **Home** — Create room / Join room (code input)
2. **Lobby** — show code, connection status, Start when 2 players present
3. **Role + pick** — role banner + searchable entity list
4. **Countdown (3s)** — shared pause before reveal
5. **Reveal + guess** — entities, 30s timer, guess field, correct/locked feedback
6. **Round result** — points and totals
7. **Match end** — winner, final scores, play again

UI for v1: clear sports theme, readable timers; no heavy animation requirement beyond countdown/timer presence.

## Server responsibilities

- Create/join rooms; enforce 2-player cap
- Assign `playerId` seats; support reconnect with same seat
- Drive round state machine and timers
- Validate picks against curated lists; validate guesses; compute scores
- Broadcast public state; keep secret picks private until reveal

Clients send intents only (`createRoom`, `joinRoom`, `startMatch`, `pickEntity`, `submitGuess`, etc.) and render server snapshots / events.

## Errors, disconnects & edge cases

| Case | Behavior |
| --- | --- |
| Disconnect mid-match | Pause ~30s with “Opponent reconnecting…”. Rejoin via room code + stored `playerId` (e.g. `sessionStorage`). If no return, match abandoned; remaining player wins by forfeit. |
| Refresh | Client restores `roomCode` + `playerId` and reconnects to the same seat. |
| Room full | Reject join with clear error. |
| Both wrong / timeout | End round; show scores; optionally show one sample valid player; continue. |
| Server restart | In-memory rooms lost (acceptable for v1). |

## Testing

- **Unit:** input normalization and alias matching; speed-score formula; dataset invariant (every exposed pair has ≥1 player).
- **Integration:** create/join; full round state machine with two mocked sockets (pick → pause → reveal → guess → result × rounds).
- **Manual:** two browser windows, complete 5-round match including wrong-lock, reconnect, and timeout paths.

## Component boundaries (implementation hint)

| Unit | Responsibility | Depends on |
| --- | --- | --- |
| `dataset` | Load/query nations, clubs, valid players, pair invariant | JSON files |
| `scoring` | Pure speed-curve function | none |
| `guessMatcher` | Normalize + match aliases for a nation×club | `dataset` |
| `roomManager` | Rooms, seats, reconnect | Socket.io |
| `matchEngine` | Round state machine + timers | `roomManager`, `dataset`, `scoring`, `guessMatcher` |
| React screens | Render state; emit intents | Socket client |

## Out of scope follow-ups

- Football API enrichment
- Persistent rooms / database
- Ranked play, accounts, chat
- Random public matchmaking
