# Kill the Prez MVP Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use `superpowers:subagent-driven-development` (recommended) or `superpowers:executing-plans` to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Deliver a locally playable, browser-based 5–10-player Kill the Prez game with an authoritative Colyseus server, private phone controls, and a public shared display.

**Architecture:** Use Colyseus rooms for authoritative live sessions and synchronization, a pure TypeScript game-module contract for rules, and React/Vite clients for the shared display and player phones. Public room state contains only public information; a server-generated private view is sent directly to its entitled player. Store rooms and seat data in memory only; do not add accounts, databases, match history, or public deployment.

**Tech Stack:** Node.js LTS (the current local Node 22 environment), TypeScript, Colyseus 0.18 API family, npm workspaces, React, Vite, `@colyseus/sdk`, `@colyseus/react`, Zod, Vitest, `@colyseus/testing`, React Testing Library, and Playwright. The Colyseus scaffold supports a TypeScript monorepo with `apps/backend` and `apps/frontend`, `netcode none`, and reconnection matchmaking.[3] The browser SDK is `@colyseus/sdk`, and the React hooks package is `@colyseus/react`.[4][5]

Pin generated Colyseus dependencies to the 0.18 API family and confirm them against the migration guide.[11]

**Spec:** `docs/superpowers/specs/2026-11-04-kill-the-prez-design.md`

## Global Constraints

- Support 5–10 players in a roughly 30-minute in-person session.
- Work in ordinary mobile and desktop browsers; no app installation or player account is required.
- One shared display shows public information; each player uses a phone for private role information, voting, and actions.
- The display owner participates from a phone; the display connection is public-only and does not consume a player seat.
- Keep hidden roles, policy hands, ballots before reveal, and investigation results private to the entitled player.
- Use an authoritative server; validate all commands on the server and never trust a client-supplied seat identity.
- No accounts, phone numbers, contacts, persistent profiles/history, public room listings, or in-game chat.
- Keep room and match data in memory only; do not add a database or persistent match history.
- Develop and test locally first. Public hosting, hosting provider selection, and paid hosting are out of scope until local playtesting succeeds.
- For phone testing, expose development servers only on the same trusted Wi-Fi LAN; do not configure port forwarding or change firewall rules without approval.
- Never log reconnection tokens or private roles, factions, policy hands, unrevealed ballots, or investigation results.
- Keep rooms unlisted; only invite-code resolution may return the opaque room ID.
- This is a noncommercial, share-alike adaptation. Include attribution and licensing notices; use original wording and visual treatment, and do not reuse Secret Hitler logos or artwork. Do not submit to an app store without approval.[1]
- Use fictional composite political setting and invented names only; do not use real politicians, current events, or one-to-one real-country stand-ins.
- The Prez is the hidden faction leader and assassination target; Speaker is the rotating elected office, and Cabinet Nominee is the second elected office.
- The sitting Speaker may use a policy-unlocked power regardless of faction. An investigation reveals only faction, never secret role. A successful assassination of The Prez wins for the Opposition.
- Do not add One Night Ultimate Werewolf mechanics, role-swapping, an expansion marketplace, paid content, or generic plugin-marketplace infrastructure in v1. Any future optional modules are free.
- Comment non-obvious logic for hidden-view projection, role knowledge, term limits, chaos, special-election rotation, and reconnect/handoff; avoid comments on obvious code.

---

## Scope and implementation defaults for review

This remains one implementation plan because the authoritative game engine, room lifecycle, and two browser clients must agree on a single action/state protocol; the tasks below each have their own tests and can be reviewed independently.

The design spec leaves several low-risk operational details open. This plan uses adjustable v1 defaults so tests are deterministic:

- Display names: trim whitespace, normalize Unicode to NFC, allow at most 20 Unicode code points, and reject duplicates using `trim().normalize("NFC").toLocaleLowerCase("en-US")` as the comparison key.
- Invite code: six uppercase characters from `23456789ABCDEFGHJKLMNPQRSTUVWXYZ`; it maps to an opaque Colyseus room ID and is not a credential for any personal account.
- Reconnect seat hold: 5 minutes. Host-control handoff: after 120 seconds offline; the original host can still reconnect during the 5-minute window as a normal player. These values live in named constants, not scattered timers. Colyseus supports both automatic and token-based manual reconnection; its guidance describes 5 minutes as a reasonable window for turn-based games.[10]
- Empty lobby expiration: 15 minutes without a player seat. Ended or abandoned rooms are disposed when clients leave; no match data is written to disk.
- Host control belongs to the first player phone seat. The room creator joins from their phone before sharing the displayed invitation; the shared display is a separate public-only connection. This keeps the creator as a player, not a moderator seat.
- A disconnect pauses only when an offline seat is required for the current decision (for example, an uncast ballot or a required officeholder action). No automatic vote or policy choice is made. The host may restart/abandon the room.
- There are no automatic turn deadlines in v1; the table’s face-to-face discussion is not timed by the app.

If Gabriel wants different defaults, adjust this section and the matching tests before implementation; do not silently change these values during coding.

## File map

- `package.json` — npm workspace root and common `dev`, `test`, `typecheck`, `build`, and `test:e2e` commands.
- `packages/protocol/src/index.ts` — wire-level IDs, public snapshots, private player snapshots, client commands, server messages, and user-safe error codes shared by browser and server.
- `apps/backend/src/game/GameModule.ts` — the small reusable game-module contract, with no Colyseus imports.
- `apps/backend/src/game/registry.ts` — registers `kill-the-prez` as the first game module and rejects unknown module IDs.
- `apps/backend/src/games/kill-the-prez/model.ts` — domain types, phase union, roles, faction labels, policies, powers, and state invariants.
- `apps/backend/src/games/kill-the-prez/setup.ts` — deterministic role/policy setup and count-specific starting knowledge.
- `apps/backend/src/games/kill-the-prez/transitions.ts` — pure command validation and state transitions for elections, legislation, powers, and end states.
- `apps/backend/src/games/kill-the-prez/projections.ts` — public and per-player projections; no Colyseus serialization logic.
- `apps/backend/src/games/kill-the-prez/*.test.ts` — Vitest unit tests for setup, elections, legislation, powers, and projections.
- `apps/backend/src/rooms/PublicRoomState.ts` — Colyseus Schema containing only public room/game fields.
- `apps/backend/src/rooms/GameRoom.ts` — room lifecycle, seat/session mapping, command validation, public publication, targeted private messages, pause logic, and host controls.
- `apps/backend/src/rooms/InviteCodeRegistry.ts` — in-memory invite-code-to-room-ID mapping and collision checks.
- `apps/backend/src/rooms/lifecycleConfig.ts` — reconnect, host-handoff, and idle-lobby timeout constants.
- `apps/backend/src/rooms/messages.ts` — Zod-validated, typed Colyseus messages that translate wire commands into game-module actions.
- `apps/backend/src/http/roomCodeRoutes.ts` — type-safe HTTP endpoint resolving a six-character invite code to a room ID after validating the code.
- `apps/backend/src/app.config.ts` — Colyseus room registration and invite lookup route registration.
- `apps/backend/src/rooms/GameRoom.test.ts` — integration tests using simulated Colyseus clients.
- `apps/frontend/src/App.tsx` — route selection and shared app shell.
- `apps/frontend/src/views/JoinView.tsx` — invite-code and display-name entry.
- `apps/frontend/src/views/PublicDisplayView.tsx` — public-only room display, QR invitation, lobby, and board.
- `apps/frontend/src/views/PlayerView.tsx` — private role information, ballots, policy choices, powers, reconnect state, and host controls.
- `apps/frontend/src/components/` — focused lobby, role briefing, election, legislative, power, and endgame components.
- `apps/frontend/src/room/connection.ts` — Colyseus SDK creation, invite resolution, reconnection-token persistence, and message subscriptions.
- `apps/frontend/src/styles.css` — responsive, accessible, original CSS-based visual treatment.
- `apps/frontend/src/**/*.test.tsx` — React Testing Library component/flow tests.
- `apps/frontend/vitest.config.ts` — jsdom environment and React Testing Library setup.
- `apps/frontend/e2e/kill-the-prez.spec.ts` — multi-client browser acceptance tests.
- `playwright.config.ts` — starts local server and Vite webServer commands and selects Chromium for E2E tests.
- `docs/ATTRIBUTION.md` — noncommercial/share-alike attribution and source link; legal scope is not represented as legal advice.
- `README.md` — local setup, run, test, and data-handling instructions.

## Protocol and game-module interfaces

`packages/protocol/src/index.ts` defines shared transport types. Keep names and wire fields stable across tasks:

```ts
export type SeatId = string;
export type RoomCode = string;
export type Faction = "opposition" | "prez";
export type Role = "opposition" | "prez_member" | "the_prez";
export type Vote = "yes" | "no";

export interface PublicPlayer {
  seatId: SeatId;
  displayName: string;
  connected: boolean;
  alive: boolean;
  isHost: boolean;
}

export interface PublicGameView {
  phase: "role_briefing" | "nomination" | "vote" | "legislative" | "executive_power" | "finished";
  nominationSpeakerSeatId: SeatId | null;
  nomineeCandidateSeatId: SeatId | null;
  electedSpeakerSeatId: SeatId | null;
  electedNomineeSeatId: SeatId | null;
  ballotsReceived: number;
  ballotCount: number;
  roleBriefingReadyCount: number;
  roleBriefingCount: number;
  revealedBallots: Array<{ seatId: SeatId; vote: Vote }> | null;
  oppositionPolicies: number;
  prezPolicies: number;
  electionTracker: 0 | 1 | 2;
  vetoUnlocked: boolean;
  pendingPower: "investigate" | "special_election" | "policy_peek" | "assassinate" | null;
  gameWinner: Faction | null;
  publicNotice: string | null;
}

export interface PublicRoomSnapshot {
  roomPhase: "lobby" | "playing" | "finished";
  roomStatus: "active" | "paused";
  pauseMessage: string | null;
  players: PublicPlayer[];
  game: PublicGameView | null;
}

export interface PrivatePlayerView {
  seatId: SeatId;
  role: Role;
  faction: Faction;
  roleBriefingAcknowledged: boolean;
  availableActions: GameAction["type"][];
  knownPrezFactionSeatIds: SeatId[];
  policyHand: Array<{ policyId: string; kind: "opposition" | "prez" }> | null;
  investigationResult: { targetSeatId: SeatId; faction: Faction } | null;
  policyPeek: Array<"opposition" | "prez"> | null;
}

export interface RoomInfoMessage { roomCode: RoomCode }
export interface ActionErrorMessage { code: string; message: string }

export type GameAction =
  | { type: "acknowledge_role" }
  | { type: "nominate"; nomineeSeatId: SeatId }
  | { type: "cast_ballot"; vote: Vote }
  | { type: "discard_policy"; policyId: string }
  | { type: "enact_policy"; policyId: string }
  | { type: "request_veto" }
  | { type: "resolve_veto"; consent: boolean }
  | { type: "use_power"; targetSeatId?: SeatId };
```

`PublicGameView` contains only game-projected public fields; `PublicRoomSnapshot` wraps the lobby/game view with connection status and seats. Neither contains a role, faction, unannounced ballot choice, policy hand, deck order, investigation result, or private role-recognition list. `PrivatePlayerView` is constructed only for its `seatId`; `action_error` and `room_info` are separate targeted server messages. A display connection receives the public room snapshot and no player-private view.

`apps/backend/src/game/GameModule.ts` defines the minimal reusable boundary:

```ts
export interface RandomSource {
  nextInt(maxExclusive: number): number;
}

export type ActionResult<State> =
  | { ok: true; state: State; publicNotice?: string }
  | { ok: false; code: string; message: string };

export interface GameModule<State, Action, PublicView, PrivateView> {
  readonly id: string;
  createGame(players: readonly { seatId: SeatId; displayName: string }[], random: RandomSource): State;
  applyAction(state: State, actorSeatId: SeatId, action: Action): ActionResult<State>;
  getPublicView(state: State): PublicView;
  getPrivateView(state: State, viewerSeatId: SeatId): PrivateView;
  requiredActors(state: State): readonly SeatId[];
}
```

Production `RandomSource` uses Node’s cryptographic random integer API; tests inject a seeded deterministic source. The game module never imports Colyseus or React. The room never mutates internal game fields directly; every state change goes through `applyAction`, and every client projection goes through the two view functions.

## Tasks

### Task 1: Scaffold a local TypeScript Colyseus/browser workspace

**Files:**
- Modify/create: root `package.json`, `.gitignore`, `README.md`
- Create: `packages/protocol/package.json`, `packages/protocol/tsconfig.json`, `packages/protocol/src/index.ts`
- Modify: generated `apps/backend/package.json`, `apps/frontend/package.json`, and workspace TypeScript configs

**Interfaces:**
- Produces workspace package names `@killtheprez/server`, `@killtheprez/web`, and `@killtheprez/protocol`.
- Produces root commands `npm run dev:server`, `npm run dev:web`, `npm test`, `npm run typecheck`, `npm run build`, and `npm run test:e2e`.
- Consumes the official Colyseus scaffold’s `apps/backend` and `apps/frontend` layout. The generator can be run into the existing project root with `--overwrite`; preserve `.git/` and `docs/`, and replace the one-line README with project instructions.[3]

- [ ] **Step 1: Generate the scaffold**

Run from the repository root:

```bash
npm create colyseus-app@latest . -- --ts --layout monorepo --netcode none --matchmaking reconnection --auth none --database none --yes --overwrite
```

`netcode none` avoids adding realtime prediction to a turn-based game; reconnection matchmaking is included, while authentication and database features are explicitly disabled.[3]

- [ ] **Step 2: Create a failing protocol test**

Create `packages/protocol/package.json`, `packages/protocol/tsconfig.json`, `packages/protocol/src/index.ts`, and `packages/protocol/src/index.test.ts`. Give the package a `test` script of `vitest run`; leave `PROTOCOL_VERSION` unexported so the first run fails for the missing symbol, not because the workspace is missing.

```ts
import { expect, test } from "vitest";
import { PROTOCOL_VERSION } from "./index";

test("protocol package exposes the v1 wire contract", () => {
  expect(PROTOCOL_VERSION).toBe(1);
});
```

Run: `npm test --workspace @killtheprez/protocol`
Expected: FAIL with the missing `PROTOCOL_VERSION` export.

- [ ] **Step 3: Wire workspace scripts and protocol package**

Add `packages/*` to the root npm workspaces, set the three package names above, and set the generated backend test script to `vitest run` (retain the scaffold’s tested `dev` command). Define the root scripts exactly:

```json
{
  "workspaces": ["apps/*", "packages/*"],
  "scripts": {
    "dev:server": "npm run dev --workspace @killtheprez/server",
    "dev:web": "npm run dev --workspace @killtheprez/web",
    "test": "npm run test --workspaces --if-present",
    "typecheck": "npm run typecheck --workspaces --if-present",
    "build": "npm run build --workspaces --if-present",
    "test:e2e": "playwright test --config playwright.config.ts"
  }
}
```

Install Vitest in `apps/backend` and `packages/protocol`, set each `test` script to `vitest run`, export `const PROTOCOL_VERSION = 1 as const`, and ensure the frontend package includes React, React DOM, Vite, `@vitejs/plugin-react`, `react-router-dom`, `qrcode.react`, `@colyseus/sdk`, and `@colyseus/react` even if the generated frontend is framework-neutral. Keep the backend `dev` command generated by the scaffold; set `apps/frontend` `dev` to `vite --host 0.0.0.0`. Set `typecheck` to `tsc --noEmit` in all three workspaces; use `tsc` for backend build, `vite build` for frontend build, and `tsc --noEmit` for the type-only protocol build. Configure the backend to bind on the trusted LAN only for phone playtesting; do not expose it publicly. Colyseus’ TypeScript SDK includes its own TypeScript definitions, and its React package provides room/state hooks.[4][5]

- [ ] **Step 4: Verify clean scaffold commands**

Run: `npm install && npm run typecheck && npm test && npm run build`
Expected: all commands exit 0; no database, OAuth, authentication, production hosting, or secrets are configured.

- [ ] **Step 5: Document local commands and commit**

Update `README.md` with Node LTS prerequisite, install/run/test commands, and the policy that no accounts or match history are stored. Add `docs/ATTRIBUTION.md` naming creators Mike Boxleiter, Tommy Maranges, and Mac Schubert; linking Secret Hitler’s official site/rulebook and the CC BY-NC-SA 4.0 license; and explaining that Kill the Prez is a fictional political retheming. Do not copy its art or logo.[1]

Run: `git diff --check && git status --short`
Commit: `git add package.json package-lock.json .gitignore README.md apps packages docs/ATTRIBUTION.md && git commit -m "chore: scaffold local Colyseus game workspace"`

### Task 2: Define the game module and test role setup/private starting knowledge

**Files:**
- Create: `apps/backend/src/game/GameModule.ts`
- Create: `apps/backend/src/games/kill-the-prez/model.ts`, `setup.ts`, `projections.ts`
- Create: `apps/backend/src/games/kill-the-prez/setup.test.ts`, `projections.test.ts`
- Modify: `packages/protocol/src/index.ts`

**Interfaces:**
- Produces the `GameModule`, `RandomSource`, `ActionResult`, `PublicGameView`, and `PrivatePlayerView` contracts above.
- Produces `createKillThePrezGame(players, random): KillThePrezState`, `getPublicView(state): PublicGameView`, and `getPrivateView(state, seatId): PrivatePlayerView`.
- Starts in `role_briefing`; `acknowledge_role` is required from each seated player before the initial Speaker nomination begins. The public view exposes only the ready count.
- `Role` values are `opposition`, `prez_member`, and `the_prez`; `Faction` values are `opposition` and `prez`. The Prez role belongs to the Prez faction but is distinct from ordinary Prez members.

Use these domain types in `model.ts`; later tasks extend `GamePhase` only if a named phase is needed for a specific decision:

```ts
export type PolicyKind = "opposition" | "prez";
export type ExecutivePower = "investigate" | "special_election" | "policy_peek" | "assassinate";
export type GamePhase =
  | "role_briefing" | "nomination" | "vote"
  | "legislative_speaker" | "legislative_nominee" | "veto_response"
  | "executive_power" | "finished";

export interface KillThePrezPlayer {
  seatId: SeatId;
  displayName: string;
  role: Role;
  faction: Faction;
  alive: boolean;
  roleAcknowledged: boolean;
}

export interface PolicyCard { policyId: string; kind: PolicyKind }

export interface KillThePrezState {
  players: KillThePrezPlayer[];
  startingPlayerCount: 5 | 6 | 7 | 8 | 9 | 10;
  phase: GamePhase;
  nominationSpeakerSeatId: SeatId;
  nomineeCandidateSeatId: SeatId | null;
  electedSpeakerSeatId: SeatId | null;
  electedNomineeSeatId: SeatId | null;
  ballots: Partial<Record<SeatId, Vote>>;
  policyDeck: PolicyCard[];
  discardPile: PolicyCard[];
  policyHands: Partial<Record<SeatId, PolicyCard[]>>;
  oppositionPolicies: number;
  prezPolicies: number;
  electionTracker: 0 | 1 | 2;
  vetoUnlocked: boolean;
  lastElectedPair: { speakerSeatId: SeatId; nomineeSeatId: SeatId } | null;
  specialElectionReturnSeatId: SeatId | null;
  investigatedSeatIds: SeatId[];
  privateResults: Partial<Record<SeatId, {
    investigationResult: { targetSeatId: SeatId; faction: Faction } | null;
    policyPeek: PolicyKind[] | null;
  }>>;
  pendingPower: ExecutivePower | null;
  winner: Faction | null;
}
```

- [ ] **Step 1: Write role-distribution and knowledge tests**

Use the exact table below as `it.each` input; faction totals include The Prez. The official rules specify one special leader, the ordinary-faction counts, and the different starting-knowledge procedure for 5–6 versus 7–10 players.[1][2][13]

| Seats | Opposition | Ordinary Prez members | The Prez | Starting knowledge |
|---:|---:|---:|---:|---|
| 5 | 3 | 1 | 1 | Both Prez-faction players know each other. |
| 6 | 4 | 1 | 1 | Both Prez-faction players know each other. |
| 7 | 4 | 2 | 1 | Ordinary Prez members know each other and The Prez; The Prez knows no other seat. |
| 8 | 5 | 2 | 1 | Ordinary Prez members know each other and The Prez; The Prez knows no other seat. |
| 9 | 5 | 3 | 1 | Ordinary Prez members know each other and The Prez; The Prez knows no other seat. |
| 10 | 6 | 3 | 1 | Ordinary Prez members know each other and The Prez; The Prez knows no other seat. |

For each row, assert exact role counts, all seats receive exactly one role, every role/faction pairing is valid, starting knowledge matches the table, and no Opposition player’s private view lists a Prez-faction seat. Also assert 4 and 11 seats are rejected.

Use this table as the production distribution constant as well as the test input:

```ts
const ROLE_COUNTS: Record<number, { opposition: number; prezMembers: number }> = {
  5: { opposition: 3, prezMembers: 1 },
  6: { opposition: 4, prezMembers: 1 },
  7: { opposition: 4, prezMembers: 2 },
  8: { opposition: 5, prezMembers: 2 },
  9: { opposition: 5, prezMembers: 3 },
  10: { opposition: 6, prezMembers: 3 },
};
```

Run: `npm test --workspace @killtheprez/server -- setup.test.ts projections.test.ts`
Expected: FAIL because setup and projection functions do not exist.

- [ ] **Step 2: Implement deterministic setup and view projections**

Create the roster from the table, shuffle it with the injected `RandomSource`, choose the initial Speaker candidate from the shuffled living seats, and create six Opposition policy cards plus eleven Prez policy cards. Store role, faction, policy deck, discard pile, and pending private knowledge only in server-owned domain state. Start the game in `role_briefing`; after every player acknowledges their phone briefing, transition to `nomination`. Ensure `getPublicView()` never returns secret values.

Keep shuffling injectable and uniform; the production source is cryptographic and tests use a seeded source:

```ts
function shuffle<T>(items: T[], random: RandomSource): T[] {
  for (let i = items.length - 1; i > 0; i -= 1) {
    const j = random.nextInt(i + 1);
    [items[i], items[j]] = [items[j], items[i]];
  }
  return items;
}

const counts = ROLE_COUNTS[players.length];
if (!counts) throw new RangeError("A game requires 5 to 10 players.");
const roles: Role[] = [
  ...Array.from({ length: counts.opposition }, () => "opposition" as const),
  ...Array.from({ length: counts.prezMembers }, () => "prez_member" as const),
  "the_prez",
];
shuffle(roles, random);
const assigned = players.map((player, index) => ({
  ...player,
  role: roles[index],
  faction: roles[index] === "opposition" ? "opposition" as const : "prez" as const,
  alive: true,
  roleAcknowledged: false,
}));
```

Use `assigned` to initialize the `KillThePrezState` fields above, set `startingPlayerCount` to `players.length`, and create a shuffled 6+11 card policy deck; initialize ballots/hands/discard/results empty, policy tracks/tracker at zero, and choose a random living `nominationSpeakerSeatId`. The `getPrivateView` projection derives starting knowledge from the count table: at 5–6 players, The Prez and the ordinary Prez member recognize each other; at 7–10, ordinary Prez members recognize each other and The Prez while The Prez recognizes no other seat.

- [ ] **Step 3: Add projection leak tests**

Assert that JSON serialization of `getPublicView()` contains none of: `role`, `faction`, `knownPrezFactionSeatIds`, `policyHand`, `investigationResult`, `policyPeek`, or policy-deck order. For each seat, assert `getPrivateView()` includes that seat’s role/faction, permitted starting knowledge, role-briefing status, available actions, and only its own current policy hand/private results.

Run: `npm test --workspace @killtheprez/server -- setup.test.ts projections.test.ts`
Expected: PASS for all six player counts and privacy assertions.

- [ ] **Step 4: Commit setup and module contract**

Run: `npm run typecheck --workspace @killtheprez/server && npm test --workspace @killtheprez/server`
Commit: `git add apps/backend/src/game apps/backend/src/games/kill-the-prez packages/protocol/src/index.ts && git commit -m "feat: add Kill the Prez game module and setup"`

### Task 3: Implement nominations, ballots, term limits, and election chaos

**Files:**
- Create: `apps/backend/src/game/registry.ts`, `apps/backend/src/games/kill-the-prez/transitions.ts`
- Modify: `apps/backend/src/games/kill-the-prez/model.ts`, `projections.ts`
- Create: `apps/backend/src/games/kill-the-prez/elections.test.ts`
- Modify: `packages/protocol/src/index.ts`

**Interfaces:**
- Completes `applyAction(state, actorSeatId, action): ActionResult<KillThePrezState>` for `acknowledge_role`, `nominate`, and `cast_ballot` actions, then registers the completed module as `kill-the-prez`.
- Maps internal `legislative_speaker`, `legislative_nominee`, and `veto_response` phases to public phase `legislative`; only a completed ballot set is public.
- `requiredActors(state)` returns all unacknowledged seats in `role_briefing`, the Speaker candidate in `nomination`, every living uncast voter in `vote`, the acting officeholder during legislation/powers, and no seats in `finished`.

- [ ] **Step 1: Write failing election tests**

Add these named tests with the stated outcomes:

- `rejectsNominationFromNonSpeaker`: only the current Speaker candidate may nominate.
- `rejectsIneligibleNominee`: the current Speaker cannot nominate themselves; an ineligible Cabinet Nominee candidate is rejected.
- `termLimitsUseLastElectedGovernment`: a failed nomination/vote does not create term limits; the last elected Speaker and Nominee do.
- `fiveOrFewerLivingPlayersOnlyBlockLastNominee`: when at most five players remain, the last elected Nominee is blocked but the last elected Speaker may be nominated.
- `keepsIndividualBallotsPrivateUntilEveryoneVotes`: before the last ballot, public view has `revealedBallots === null`; after the final ballot, it contains every living voter’s choice together.
- `strictMajorityPassesAndTieFails`: a strict majority of yes votes passes; a tie fails.
- `failedElectionAdvancesCandidateAndTracker`: failed vote increments tracker and advances to the next living Speaker candidate.
- `thirdFailureEnactsTopPolicyWithoutPower`: third consecutive failure reshuffles first if fewer than three policy cards remain, enacts the top card, resets tracker and term limits, and does not enqueue an executive power.
- `policyEnactmentResetsElectionTracker`: any enacted policy resets the tracker to zero.
- `rotationSkipsEliminatedSeats`: turn order advances to the next living seat only.
- `electedGovernmentAdvancesNormalRotation`: after the legislative session, ordinary rotation advances clockwise from the Speaker candidate.
- `roleBriefingWaitsForEveryPlayer`: the first nomination cannot begin until every seated player acknowledges their private briefing.
- `nonPrezNomineeAfterThreePrezPoliciesIsPubliclyCleared`: after the Prez policy threshold, an elected nominee who is not The Prez is publicly confirmed not to be The Prez; the role remains private otherwise.

Use explicit fixtures that set current phase, last-elected pair, player count, tracker, and policy deck so each assertion is deterministic.

Run: `npm test --workspace @killtheprez/server -- elections.test.ts`
Expected: FAIL on missing transitions and projection fields.

- [ ] **Step 2: Implement election transitions**

Implement strict-majority counting over living seats; store ballot choices only in server-owned state; publish none of them until all living seats have submitted. On the third failed election, if fewer than three cards remain, shuffle the remaining draw and discard piles before revealing the top card; enact it, do not trigger a power, clear term limits, and reset the tracker. Add comments to explain the “last elected pair” and chaos-order rules.

- [ ] **Step 3: Verify election and type checks**

Run: `npm test --workspace @killtheprez/server -- elections.test.ts && npm run typecheck --workspace @killtheprez/server`
Expected: all named tests PASS and TypeScript exits 0.

- [ ] **Step 4: Commit election rules**

Commit: `git add apps/backend/src/games/kill-the-prez packages/protocol/src/index.ts && git commit -m "feat: implement elections and chaos rules"`

### Task 4: Implement legislation, policy tracks, veto, executive powers, and win conditions

**Files:**
- Modify: `apps/backend/src/games/kill-the-prez/model.ts`, `transitions.ts`, `projections.ts`
- Create: `apps/backend/src/games/kill-the-prez/legislation.test.ts`, `powers.test.ts`
- Modify: `packages/protocol/src/index.ts`

**Interfaces:**
- Adds `discard_policy`, `enact_policy`, `request_veto`, `resolve_veto`, and `use_power` actions to `GameAction`.
- Speaker’s hand is private to Speaker; the two remaining policies are private to Cabinet Nominee; the enacted policy and track counts are public.
- `pendingPower` is derived from the Prez track for `state.startingPlayerCount`, fixed when the game is created; later eliminations never change the track. The client cannot choose a different power.

- [ ] **Step 1: Write failing legislation and track tests**

Add tests asserting: the Speaker draws exactly three policies; only the Speaker can discard one; the Nominee receives exactly the other two; only one remaining policy can be enacted; neither discarded policy is disclosed; after a legislative session (including veto), fewer than three remaining cards causes the discard and draw piles to be shuffled together before any executive power; and before chaos, fewer than three cards causes a reshuffle before the top-deck. Test the deck starts with exactly 6 Opposition and 11 Prez policies.

Add this explicit Prez-track table to a constant and tests; the values are transcribed from the rulebook’s player-count tracks and cross-checked against a text rules reference.[1][13]

| Prez policy slot | 5–6 players | 7–8 players | 9–10 players |
|---:|---|---|---|
| 1 | None | None | Investigate |
| 2 | None | Investigate | Investigate |
| 3 | Policy peek | Special election | Special election |
| 4 | Assassinate | Assassinate | Assassinate |
| 5 | Assassinate | Assassinate | Assassinate; veto is available to later sessions |
| 6 | Prez faction wins | Prez faction wins | Prez faction wins |

Additional required assertions:

- `chaosPolicyDoesNotTriggerTrackPower`: a chaos-enacted Prez policy advances its track and may end the game, but does not trigger its slot power.
- `reshuffleBeforePolicyPeek`: a session-ending reshuffle occurs before Policy Peek, and the peek returns the reshuffled deck’s top three in order without changing the deck.
- `vetoUnlocksAfterFifthPrezPolicyForSubsequentSessions`: the fifth policy’s normal power still resolves; veto can be proposed in later legislative sessions.
- `acceptedVetoDiscardsBothRemainingPoliciesAndAdvancesTracker`: the government’s original Speaker discard plus the two vetoed policies all enter the discard pile; the tracker increments; the third failure invokes chaos.
- `rejectedVetoRequiresNomineeToEnact`: the Speaker’s refusal leaves the Nominee required to choose one of the two policies.
- `policyPeekIsPrivateAndPreservesOrder`: only the Speaker receives the top three card kinds, in their existing order; the deck is unchanged.
- `investigationRevealsFactionNotRoleAndCannotRepeatTarget`: only the Speaker receives the target faction; no secret role or public result is sent; a target can be investigated at most once.
- `specialElectionAllowsTermLimitedSpeakerCandidate`: the Speaker may choose any other living seat as the next candidate, even if that seat was term-limited as Nominee; ordinary rotation resumes after the Speaker who called it.
- `assassinationOfNonPrezEliminatesWithoutRevealingFaction`: target is removed from voting/nominations; public state does not reveal faction.
- `assassinationOfPrezWinsForOpposition`: game ends immediately for Opposition.
- `fiveSafeguardPoliciesAndSixSuppressionPoliciesEndGame`: the applicable track threshold wins immediately.
- `PrezElectedNomineeAfterThreeSuppressionPoliciesWins`: a successful election of The Prez as Cabinet Nominee after three Prez policies ends immediately for the Prez faction.

Run: `npm test --workspace @killtheprez/server -- legislation.test.ts powers.test.ts`
Expected: FAIL on missing legislative transitions and track table.

- [ ] **Step 2: Implement the policy deck and legislative sequence**

Use a cryptographically backed production `RandomSource`; use the injected deterministic source in tests. Draw three, discard one secretly, pass two privately, enact one publicly, and keep the other secret in the discard pile. At the end of every legislative session, including a successful veto, if fewer than three cards remain, shuffle the discard pile with the remaining draw pile before any executive power; before chaos, apply the same fewer-than-three rule before the top-deck. Store policy identifiers server-side; accept only an ID currently in the acting player’s private hand.

- [ ] **Step 3: Implement veto and executive powers**

Implement the table above as a pure function `powerForPrezPolicySlot(startingPlayerCount, slot)`. Apply a single pending power after an elected government enacts the relevant policy; the Speaker’s faction does not affect eligibility. Policy peek, investigation, special election, and assassination are the only powers. An exhausted target set is handled as a legal skipped power with a public notice and no fabricated result.

- [ ] **Step 4: Verify wins, hidden results, and all track variants**

Run: `npm test --workspace @killtheprez/server -- legislation.test.ts powers.test.ts && npm run typecheck --workspace @killtheprez/server`
Expected: PASS for 5, 6, 7, 8, 9, and 10 players; all private results are absent from public projections.

- [ ] **Step 5: Commit legislation and powers**

Commit: `git add apps/backend/src/games/kill-the-prez packages/protocol/src/index.ts && git commit -m "feat: implement policies powers and win conditions"`

### Task 5: Connect the pure module to an authoritative Colyseus room

**Files:**
- Create: `apps/backend/src/rooms/PublicRoomState.ts`, `GameRoom.ts`, `messages.ts`, `GameRoom.test.ts`
- Modify: `apps/backend/src/app.config.ts`, `apps/backend/src/game/registry.ts`
- Modify: `packages/protocol/src/index.ts`

**Interfaces:**
- Room name: `game_room`; registered module ID: `kill-the-prez`.
- Public synchronization uses `PublicRoomState` only. Private data uses targeted `client.send("private_view", view)` messages; no role, faction, hand, deck order, investigation, or ballot choice before reveal is ever put into the shared Schema. Colyseus sends all unfiltered Schema fields to every client by default; `StateView` is available for per-client filtering, but this plan chooses direct private messages to keep the shared public schema structurally free of secrets.[8][9]
- Each player `Client` receives a server-assigned `seatId` in `client.userData`; the room ignores any client-supplied actor/seat ID.
- Each accepted command flows through `GameModule.applyAction`; rejected commands return a targeted `action_error` with a safe code/message. Register the room as private/unlisted; invite-code resolution is the only intended join path.

- [ ] **Step 1: Write failing room isolation tests**

Using `@colyseus/testing`, boot `app.config.ts`, create one room, and connect a public-display client plus two player clients. Assert that all clients see the same public Schema, no role/faction/private hand fields exist in that Schema, `private_view` reaches only its owner, and the display client never receives any private message. The Colyseus test package supports simulated client connections and message exchange for room tests.[7]

Add `rejectsForgedSeatId`: send a command payload containing another seat’s ID; assert the server uses the sender’s assigned seat and rejects the unauthorized action.

Run: `npm test --workspace @killtheprez/server -- GameRoom.test.ts`
Expected: FAIL because room and message handlers do not exist.

- [ ] **Step 2: Add public-only room schema and typed messages**

`PublicRoomState` serializes to the `PublicRoomSnapshot` contract. Keep nested game/player values as Colyseus `Schema` classes or schema arrays, not untyped plain objects; publish `game.revealedBallots` only after every living seat has voted. No serialized room field may contain a role, faction, ballot map, per-player “has voted” flag, policy hand, deck order, or investigation result.

Define Zod-validated messages with typed handlers for `set_display_name`, `start_game`, `acknowledge_role`, `nominate`, `cast_ballot`, `discard_policy`, `enact_policy`, `request_veto`, `resolve_veto`, `use_power`, `restart_game`, and `abandon_room`. Colyseus message composability supports typed handlers and Zod validation; use regular functions only where handler `this` is required.[9]

- [ ] **Step 3: Implement `GameRoom` command and publication flow**

On a valid player command, derive actor seat from `client.userData`, call the game module, and on success publish the new public projection and send refreshed private projections only to player clients. On rejection, send the error only to the actor. The display client receives public state only. Accept player joins only in `lobby`; start only when 5–10 player seats are present and connected. The display connection does not count toward that limit.

- [ ] **Step 4: Run room integration tests**

Run: `npm test --workspace @killtheprez/server -- GameRoom.test.ts && npm run typecheck --workspace @killtheprez/server`
Expected: PASS for private-message isolation, public-only Schema, forged seat rejection, and valid/invalid command routing.

- [ ] **Step 5: Commit the authoritative room boundary**

Commit: `git add apps/backend/src/rooms apps/backend/src/app.config.ts apps/backend/src/game/registry.ts packages/protocol/src/index.ts && git commit -m "feat: add authoritative Colyseus game room"`

### Task 6: Add invite-code lobby, display pairing, reconnect, host handoff, and cleanup

**Files:**
- Create: `apps/backend/src/rooms/InviteCodeRegistry.ts`, `apps/backend/src/rooms/lifecycleConfig.ts`, `apps/backend/src/http/roomCodeRoutes.ts`, `apps/backend/src/rooms/lifecycle.test.ts`
- Modify: `apps/backend/src/rooms/GameRoom.ts`, `apps/backend/src/app.config.ts`, `apps/backend/src/rooms/PublicRoomState.ts`
- Create: `apps/frontend/src/room/connection.ts`

**Interfaces:**
- `POST /api/rooms/resolve` accepts `{ code: RoomCode }` and returns `{ roomId: string }` only for a live valid code; invalid/expired codes return a user-safe 404. Register it as a typed Colyseus API route with Zod validation and keep the room out of public matchmaking listings.[12]
- The room creation client uses `client.create("game_room", { connectionKind: "display" })`; a player resolves the displayed code and calls `client.joinById(roomId, { connectionKind: "player", displayName })`. A display never consumes a seat.
- `InviteCodeRegistry` exposes `register(roomId): RoomCode`, `resolve(code): string | undefined`, and `remove(roomId): void`; it holds no persistent state. Collision checks follow the Colyseus custom-room-ID pattern of checking live room identifiers before registering a human-friendly code.[6]
- `room.onDrop()` marks the seat disconnected and calls `allowReconnection(client, 300)`; the SDK supports storing the refreshed `reconnectionToken` and later calling `client.reconnect(token)` after a reload.[10]

- [ ] **Step 1: Write failing lifecycle and code tests**

Add tests asserting: generated code is six characters from the chosen alphabet; active code collisions are retried; disposing a room removes its code; a fresh in-memory registry after server restart cannot resolve an old code; expired codes cannot resolve; 4 and 11 player seats cannot start; ten player seats plus one display connection are accepted, while an 11th player seat is rejected; first player seat becomes host; display connection never counts toward 5–10 players; a 21-code-point name is rejected; duplicate names are rejected using the normalized lowercase comparison key; a reconnect restores the same `seatId` and private view; the display never receives a seat; host handoff occurs after 120 seconds offline; reconnecting former host after handoff does not reclaim control; host restart clears all game secrets and returns to lobby; host abandon disposes the room; idle empty lobby expires after 15 minutes. Inject the lifecycle clock/configuration in tests and advance a fake clock rather than waiting real minutes.

Run: `npm test --workspace @killtheprez/server -- lifecycle.test.ts`
Expected: FAIL on missing registry/routes/lifecycle methods.

- [ ] **Step 2: Implement invite registry and resolve endpoint**

Generate codes using `crypto.randomInt` with the fixed alphabet `23456789ABCDEFGHJKLMNPQRSTUVWXYZ`, check the in-memory registry for collision before returning a code, and store only `code → opaque roomId` plus the reverse lookup needed for cleanup. Send the code to the display connection in a `room_info` message; do not render a room ID or reconnection token in the UI or invitation URL. Use a URL containing only the invite code, such as `/join/ABC234`.

- [ ] **Step 3: Implement host assignment and room lifecycle**

The first player phone seat is host; the screen creator joins from their phone first, then shares the code. Set `GameRoom.maxClients = 11` for at most ten player phones plus one display. Add `HOST_HANDOFF_MS = 120_000`, `RECONNECT_WINDOW_SECONDS = 300`, and `IDLE_LOBBY_MS = 900_000` in `lifecycleConfig.ts`. If a required actor is offline, keep the game’s logical `phase` unchanged, set the independent `roomStatus` to `paused`, publish a non-sensitive `pauseMessage`, and reject commands that would advance it. On reconnect, restore that seat, clear the pause if all required actors are connected, and send the fresh private view. If the host remains offline for 120 seconds, transfer controls to the next connected living seat clockwise; preserve the original host’s seat for token-based reconnect during the remaining grace window. If reconnection expires, keep the seat offline and never auto-advance; the host may restart (return connected seats to the lobby and drop expired seats) or abandon. Host restart clears hidden game state; abandon closes the room and unregisters the invite code.

- [ ] **Step 4: Add client token lifecycle**

In `connection.ts`, keep the reconnection token only in `sessionStorage` under a key scoped to the room code; write it after join and after every successful reconnect, remove it when the room ends or reconnect is rejected, and never place it in a URL, QR code, public state, or logs. Use `client.reconnect(token)` before fresh join after a page reload; after reconnection, attach new state/message callbacks to the new `Room` instance.[10]

- [ ] **Step 5: Run lifecycle integration tests**

Run: `npm test --workspace @killtheprez/server -- lifecycle.test.ts && npm run typecheck`
Expected: PASS for room-code lifecycle, all capacity edges, display/player separation, pause, same-seat reconnect, host handoff, reset, and cleanup.

- [ ] **Step 6: Commit lobby and reconnect support**

Commit: `git add apps/backend/src/rooms apps/backend/src/http apps/backend/src/app.config.ts apps/frontend/src/room/connection.ts packages/protocol/src/index.ts && git commit -m "feat: add private rooms reconnect and host handoff"`

### Task 7: Build the shared display and phone player interface

**Files:**
- Create/modify: `apps/frontend/src/App.tsx`, `views/JoinView.tsx`, `views/PublicDisplayView.tsx`, `views/PlayerView.tsx`
- Create: `apps/frontend/src/components/LobbyPanel.tsx`, `RoleBriefing.tsx`, `ElectionControls.tsx`, `LegislativeControls.tsx`, `PowerControls.tsx`, `GameOverView.tsx`
- Create/modify: `apps/frontend/src/room/connection.ts`, `apps/frontend/src/styles.css`, `apps/frontend/vitest.config.ts`, component tests under `apps/frontend/src/`
- Modify: `apps/frontend/package.json`, root `package.json`

**Interfaces:**
- Both client views consume `PublicRoomSnapshot`/`PublicGameView`; only `PlayerView` subscribes to the current device’s `private_view` message.
- `JoinView` accepts a code and display name; `PublicDisplayView` creates the display connection and renders only public state; `PlayerView` joins by code and sends protocol commands.
- No client directly changes game state. Every action is a server command; errors and connection status are displayed without exposing internal state.

- [ ] **Step 1: Write failing UI tests for room mode separation**

Install Vitest, jsdom, `@testing-library/react`, and `@testing-library/jest-dom` in `apps/frontend`; set its `test` script to `vitest run` and configure jsdom in `vitest.config.ts`. Add tests asserting: the join form rejects empty code/name; display view renders only public seats/track/votes and never role/faction/private policy data; player view renders the current player’s role and only their allowed action; an investigation result appears only in the Speaker’s private panel and is absent from the target phone and display; no chat control exists; the host start/restart controls appear only on the host player phone, not the public display.

Run: `npm test --workspace @killtheprez/web`
Expected: FAIL because the views/components do not exist.

- [ ] **Step 2: Implement join/lobby/display views**

Add `/join/:code`, `/display`, and `/play/:code` routes. Public display shows QR/code, names, connection state, public offices, election tracker, policy tracks, a count of received ballots without identifying who has voted, and public actions. It never renders player-private messages. The host joins on their phone before sharing the code; the public display stays a spectator connection.

- [ ] **Step 3: Implement private player decision panels**

Add guided role briefing and acknowledgement, simultaneous ballot buttons, Speaker’s three-policy discard panel, Nominee’s two-policy enact panel, veto confirmation, investigation/policy-peek/assassination/special-election target selection, pause/reconnect state, and phone-only host controls. Keep private information in component memory; clear it on logout/room end. Do not add a game chat, accounts, personal data fields, or persistent history.

- [ ] **Step 4: Add original responsive styling**

Use original CSS shapes, iconography, and wording for the fictional setting. Make touch targets comfortable on phones, preserve keyboard focus and screen-reader labels, and meet WCAG AA contrast for text. Use no Secret Hitler logo, art, or copied card illustrations. The final faction/policy copy stays within the spec’s fictional composite setting; do not introduce real politicians or current events.

- [ ] **Step 5: Verify UI tests and production builds**

Run: `npm test --workspace @killtheprez/web && npm run typecheck && npm run build`
Expected: PASS; the frontend and backend builds use the protocol package and no secret data is present in public markup or public snapshots.

- [ ] **Step 6: Commit the browser interfaces**

Commit: `git add apps/frontend package.json package-lock.json && git commit -m "feat: add public display and phone controls"`

### Task 8: Exercise complete local games and document the verified run

**Files:**
- Create: `apps/frontend/e2e/kill-the-prez.spec.ts`, root `playwright.config.ts`
- Modify: `package.json`, `README.md`, `docs/ATTRIBUTION.md`
- Modify tests only where a verified behavior exposes a defect

**Interfaces:**
- Acceptance tests run against a local Colyseus server and Vite client; no hosted service or paid account is used.
- A deterministic test-only random source can be injected through test configuration, but production rooms always use cryptographic randomness.
- The manual acceptance flow uses one shared-display browser and at least five real phone browsers; the 10-player capacity is also exercised with simulated clients.

- [ ] **Step 1: Add browser acceptance tests**

Install `@playwright/test` at the workspace root and install Chromium with `npx playwright install chromium`. Configure `playwright.config.ts` to start the server and Vite with `webServer`. Cover: create room and display code/QR; join five player seats; separately fill a room to ten seats and reject an 11th player; complete private role briefing; vote with all five clients and verify no choice is revealed early; resolve one legislative session; trigger one executive power; disconnect a required voter, verify the game pauses without auto-voting, reconnect the same browser token, verify the same seat/private role returns, and complete the vote; transfer host control after the configured offline delay; finish a scripted win condition; verify ended/abandoned room cleanup and token removal.

Run: `npm run test:e2e`
Expected: PASS in headless Chromium against locally started services.

- [ ] **Step 2: Run complete automated checks**

Run:

```bash
npm test
npm run typecheck
npm run build
npm run test:e2e
```

Expected: all commands exit 0; `git diff --check` reports no whitespace errors.

- [ ] **Step 3: Perform the real-device playtest**

Run `npm run dev:server` and `npm run dev:web` in separate terminals. For the phone test, configure the Vite client with `VITE_COLYSEUS_URL=ws://<LAN-IP>:2567` and `VITE_PUBLIC_BASE_URL=http://<LAN-IP>:5173`; bind development servers only to the trusted Wi-Fi LAN, and do not configure router port forwarding or change firewall rules without approval. Put the public display on a separate desktop/tablet browser, then run one complete game with at least five participants on phones. Include one deliberate disconnect during a required decision and same-seat reconnect. Verify the display owner uses their phone as a player; public screen never shows a role, private hand, unrevealed ballot, or investigation result; disconnected decisions never auto-advance; host can restart; no accounts or personal phone data are requested; and room state disappears after the room ends. Capture only non-sensitive test notes; do not record real roles or credentials in logs/screenshots.

- [ ] **Step 4: Confirm attribution and local-only scope**

Read `README.md` and `docs/ATTRIBUTION.md`; verify attribution to Mike Boxleiter, Tommy Maranges, and Mac Schubert, the source rulebook/site link, CC BY-NC-SA 4.0 terms, and this project’s changes. Verify no deployment, paid service, app-store submission, or real-world political content was added.[1]

- [ ] **Step 5: Final repository check and commit**

Run: `git diff --check && git status --short && npm test && npm run typecheck && npm run build && npm run test:e2e`
Commit: `git add apps packages README.md docs/ATTRIBUTION.md package.json package-lock.json && git commit -m "test: verify local Kill the Prez MVP"`

## Definition of done

- The game is playable locally in ordinary desktop/mobile browsers with 5–10 player seats, a separate public display connection, and no account.
- The rule engine passes deterministic tests for setup, every election/election-tracker rule, the policy deck, all player-count power tracks, veto, investigation, special election, assassination, and all win conditions.
- Integration tests demonstrate that the public schema contains no hidden information and targeted private messages reach only the correct player.
- A required disconnected action pauses the game; accountless token reconnection restores the same seat; host controls hand off and can restart/abandon.
- Browser tests pass, and at least one complete 5+ person phone playtest with a shared display and reconnect has been completed.
- Room state is memory-only, cleared at room end, and never persisted as profiles/history; no deployment or paid service has been introduced.
- Attribution and noncommercial/share-alike notices are present; all artwork and wording are original.

## Sources

[1] https://www.secrethitler.com/assets/Secret_Hitler_Rules.pdf — Secret Hitler official rulebook
[2] https://secret-hitler.app/rules — Secret Hitler rules reference
[3] https://docs.colyseus.io/getting-started/server — Colyseus server setup
[4] https://docs.colyseus.io/getting-started/typescript — Colyseus TypeScript SDK
[5] https://docs.colyseus.io/getting-started/react — Colyseus React SDK
[6] https://docs.colyseus.io/recipes/custom-room-id — Colyseus custom room ID
[7] https://docs.colyseus.io/tools/unit-testing — Colyseus unit testing
[8] https://docs.colyseus.io/state/view — Colyseus StateView
[9] https://docs.colyseus.io/room/messages — Colyseus message composability
[10] https://docs.colyseus.io/room/reconnection — Colyseus reconnection
[11] https://docs.colyseus.io/migrating/0.18 — Colyseus 0.18 migration guide
[12] https://docs.colyseus.io/server/http-routes — Colyseus HTTP routes
[13] https://raw.githubusercontent.com/TimothyHadfield/secret-hitler-companion/main/SECRET_HITLER_RULES.md — Secret Hitler companion rules reference
