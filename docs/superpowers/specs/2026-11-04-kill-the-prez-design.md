# Kill the Prez — Design Specification

**Status:** Draft for user review  
**Date:** 2026-11-04  
**Project:** Kill the Prez

## Summary

Build a browser-based, phone-connected party game in the Secret Hitler social-deduction tradition. The first release is a faithful, noncommercial adaptation of the core rules, re-themed as fictional political satire: the Prez's faction advances voter-suppression policies while the Opposition tries to enact safeguards or stop the Prez. One shared display shows public information; each player uses a phone for private role information, voting, and actions.

The server foundation should also be reusable for future turn-based social-deduction and party games. Use `boardgame.io` as the game-state and multiplayer foundation instead of implementing the networking layer from scratch. Develop and test locally first; public hosting is a later step.

## Goals and success criteria

- Support 5–10 players in a roughly 30-minute in-person session.
- Work in ordinary mobile and desktop browsers; no app installation or player account is required.
- Let players choose their own in-room display names and join a private game by QR code or room code.
- Let the display owner participate as a player from their own phone; the display is not a moderator seat.
- Preserve the base game's election, legislative, policy, executive-power, and win-condition flow.
- Keep hidden roles, policy hands, and investigations private to the entitled player.
- Allow reconnect after a brief disconnect and transfer room controls if the creator leaves.
- Complete a real playtest with 5–10 people on phones plus a shared display, including a reconnect, without hidden-information leakage.

## Game premise and theme

The setting is an unnamed, fictional political republic that feels familiar to American players while using subtle, composite hints at multiple countries. It must not be a direct stand-in for a real country, party, candidate, or current event. The working title is **Kill the Prez**. The fictional assassination is non-graphic and has no real-world target.

- **The Prez** is the hidden leader of the faction seeking to consolidate power.
- The Prez's faction tries to enact **voter-suppression policies**.
- The **Opposition** tries to enact **safeguard laws** and can win by commissioning the fictional assassination of The Prez.
- The rotating elected office is **Speaker**. The Speaker nominates a **Cabinet Nominee**, and the table votes on the proposed government. This is intentionally satirical, not a civics simulation.
- Faction labels, candidate names, policy wording, symbols, and visual identity will be original; no Secret Hitler logo or artwork is to be reused.

## Base rules and win conditions

The base game follows the attached Secret Hitler rulebook's core rules and player-count setup. The political wording changes, but the underlying thresholds and sequence remain unchanged.[2]

- The Prez's faction wins by enacting six voter-suppression policies, or by electing The Prez as Cabinet Nominee after at least three voter-suppression policies have been enacted.
- The Opposition wins by enacting five safeguard laws or by assassinating The Prez.
- Elections retain term limits, simultaneous public ballot reveal, election-tracker/chaos rules, and the policy-deck reshuffle rules.
- Legislative sessions retain secret policy selection: the Speaker privately sees three policies and discards one; the Cabinet Nominee privately sees two and enacts one. Discarded policies remain hidden.
- The veto power activates at the same policy-track threshold as the original rules.
- When policies unlock executive powers, the sitting Speaker may use the available power regardless of faction. The power to assassinate is fictional and non-graphic; if its target is The Prez, the Opposition wins. If another player is targeted, do not reveal that player's faction.
- An investigation reveals only the target's party/faction affiliation to the Speaker, not the target's secret role or whether they are The Prez. It is private; a player may share or lie about the result, and the same player cannot be investigated twice.[2]
- Guided setup replaces the physical pass-the-envelope and eyes-closed role-recognition process. The official role distribution and private information rules are retained.

## Player experience and screens

### Lobby

1. A player creates a private room from the shared-display view.
2. The display shows a QR code and room code. Players join from phone browsers, choose their own display names, and take a seat.
3. The room accepts 5–10 players. The display owner also joins as a player by phone.
4. The room creator can start the game when the group is ready.

### Shared display

Show public information only: room code and lobby, player names, current offices, publicly revealed votes, policy tracks, election tracker, public actions, and game progress. Do not show roles, unannounced vote choices, secret policy hands, discarded policies, or private investigation results.

### Player phones

Each phone shows only that player's permitted information: role/faction, role-specific starting knowledge, ballot submission, private policy choices for the current officeholder, private power results, and available actions. Ballots are submitted privately and revealed together after all players have voted. In-game chat is out of scope; discussion remains face-to-face.

### Investigation

When the Speaker investigates a player, the Speaker's phone privately displays the target's faction only. The public display and target's phone receive no investigation result. The Speaker can disclose or misrepresent the finding verbally, matching the source rules.[2]

## Reusable server architecture

Use `boardgame.io` as the shared turn-based game server foundation:

- One server registers multiple game definitions; **Kill the Prez** is the first game module.[12][18]
- Each game module owns its setup, role distribution, phases, legal moves, policy transitions, powers, and win conditions.
- The shared web application owns the reusable room-creation/join flow, friendly room-code/QR experience, display-name entry, shared-display shell, and phone-controller shell.
- Game state is authoritative on the server. Player-specific views strip hidden information before state is sent to a client.[11][12]
- Use multi-player stages to collect simultaneous ballots and reveal them only after all ballots are locked.[17]
- Use generated, private seat credentials for accountless move authorization and reconnect. Do not put secrets in room codes or public URLs.
- Keep game-specific rules modular, but do not build an expansion marketplace or a generic plugin system in v1.

The framework's default storage is in memory. This matches the no-history MVP; a server process restart ends active rooms.[18] The exact human-readable room-code and reconnect adapter must be validated during implementation rather than assumed to be supplied turnkey by the framework.

## Privacy, room lifecycle, and failures

- No accounts, phone numbers, contacts, or persistent player profiles.
- Player-selected names are session-only and visible to the table. Enforce a reasonable display-name length and prevent duplicate names within a room or disambiguate them in the UI.
- Rooms are private and unlisted; joining requires a room code or QR invitation. There is no public room browser.
- Keep live game state in memory only. Closing or abandoning a room removes it; do not save match history.
- On a player disconnect, pause the game at the current required decision and let the same seat reconnect with its private credential. The host may abandon/restart if the player does not return.
- If the room creator disconnects and does not return after a short grace period, transfer room controls to another connected player. The original creator may reconnect as a normal player.
- A server restart ends active rooms; the MVP does not recover in-progress games.
- Invalid or expired room codes and rejected moves must produce clear, non-sensitive error messages.

## Scope boundaries

### Included in v1

- Faithful base rules for 5–10 players, with political re-theming.
- Private room creation/join by QR or code; custom player display names.
- Shared public display and individual phone interfaces.
- Guided setup, private role assignment, elections, policy selection, policy tracks, executive powers, veto, win conditions, reconnect, and host handoff.
- Local multi-device testing before public deployment.

### Deferred or excluded

- One Night Ultimate Werewolf-specific role names, wording, artwork, or exact role mechanics.
- Role-swapping, one-night/no-elimination mode, or other optional variants in the base game.
- Future optional modules are free and are considered only after the base game is playable.
- Accounts, phone-number/contact access, public room listings, persistent profiles/history, in-game chat, paid content, and app-store distribution.
- Public hosting, provider selection, and any paid hosting decision until local playtesting succeeds.

## Licensing and attribution

This is a direct, noncommercial adaptation of Secret Hitler. The official site identifies the game as CC BY-NC-SA 4.0 and requires attribution, noncommercial use, and share-alike for adaptations; it also says app-store submission requires approval.[1] Before public distribution, include appropriate attribution and licensing notices and confirm the permitted scope of the software adaptation. This specification is not legal advice.

## Verification plan

- Unit-test role distribution, legal phase transitions, term limits, policy deck reshuffles, election tracker/chaos, veto threshold, executive powers, and win conditions.
- Test the hidden-information boundary for every client view: public display, each player, Speaker, Cabinet Nominee, investigated player, and spectator.
- Test investigation specifically: faction revealed only to Speaker; secret role never revealed; repeat investigation rejected.
- Test simultaneous ballots: no individual vote is revealed before all ballots are locked; results then become public.
- Integration-test private room creation, QR/code join, custom names, 5–10 player capacity, start/restart, reconnect credentials, creator handoff, and room cleanup.
- Manually exercise a complete game with real phone browsers and a shared display before claiming the MVP works.

## Open details for later design/implementation

- Final faction names, fictional-country cues, policy labels, character names, and original art.
- Name-length limit and duplicate-name behavior (reject or add a disambiguating suffix).
- Exact reconnect grace period and idle-room expiration.
- Hosting provider and cost, selected only after local testing.
- Confirm `boardgame.io`'s lobby behavior is sufficient for the desired room-code flow; add a thin adapter if needed.

## Sources

[1] Secret Hitler official site and licensing FAQ: https://www.secrethitler.com/  
[2] Secret Hitler official rulebook: https://www.secrethitler.com/assets/Secret_Hitler_Rules.pdf  
[11] boardgame.io secret state: https://boardgame.io/documentation/secret-state.html  
[12] boardgame.io multiplayer and multiple game definitions: https://boardgame.io/documentation/multiplayer  
[17] boardgame.io stages and multiple active players: https://boardgame.io/documentation/stages.html  
[18] boardgame.io server, multiple game definitions, and in-memory default storage: https://boardgame.io/documentation/api/Server.html
