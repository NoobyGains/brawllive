# BrawlLive: in-game online matchmaking for Project+ — design spec

Date: 2026-10-15
Status: approved in conversation, awaiting written review
Owner: David (product), Claude (design and orchestration)

## 1. Summary

BrawlLive is an online matchmaking service for Project+ (P+), the community balance patch of Project M for Super Smash Bros. Brawl, played on the official P+ Dolphin netplay build. Today P+ has no matchmaking: players trade 8-character host codes on Discord. BrawlLive gives them a Slippi-style loop instead: launch Dolphin, press a button combo on the pad, get paired, and play. The player never leaves the game window and never creates an account.

The product ships as a drop-in replacement `Dolphin.exe` for the existing P+ 3.2 folder, a matchmaking service, and a small website for leaderboards and downloads.

## 2. Goals and non-goals

Goals (full v1 scope, built in sub-projects):
- Casual and ranked queues, singles and doubles.
- Everything a player does happens inside Dolphin's game window via an overlay and pad combos.
- Zero-signup identity: a device registers itself on first run and uses the P+ netplay nickname.
- Stays on the official P+ netcode and remains netplay-compatible with stock P+ 3.2 players.
- Pairing by region, ping band and, for ranked, rating; automatic host selection, buffer setting and game start.
- Rock-solid failure handling: every failure is a named state with a timeout, a message and a recovery.

Non-goals for v1:
- Rollback netcode. Brawlback and Orca exist; we do not depend on either.
- Modifying P+ game code. A native in-game menu is a later sub-project.
- Modifying Dolphin's emulation core. The patch is frontend-only, enforced by CI.
- Any distribution of game files. The player supplies their own P+ build and Brawl ISO.
- macOS and Linux builds (Windows first; the fork builds for all three, so later is possible).

## 3. Verified facts the design rests on

From REA static analysis of the shipped `Dolphin.exe` (sha256 8206123e…, Ghidra 12.1.4) and from the fork source at tag `v3.2.0` (commit 448d1dbe…), plus cited web research on 2026-10-15:

- The build is "Project+ Dolphin v3.2.0", a fork of mainline Dolphin 2606. Source is public under the GPL at github.com/Project-Plus-Development-Team/Project-Plus-Dolphin. Windows builds use `Source/dolphin-emu.sln` with the VS 2022 v143 toolset and a prebuilt Qt 6.5.1 submodule.
- No matchmaking, no Slippi code, no rollback exists in the build. The P+ FAQ confirms both.
- Dolphin has no command-line netplay. The frontend exposes `MainWindow::NetPlayHost(const GameFile&)`, `MainWindow::NetPlayJoin()`, and `NetPlayServer::RequestStartGame()`. The Discord "Ask to Join" handler is an existing template: set config keys, then call `NetPlayJoin()`.
- Both `NetPlayHost` and `NetPlayJoin` refuse while a game is running. Emulation must be stopped first; `Core::Stop` plus the `EmulationStateChanged(Uninitialized)` signal is the chain point. `ForceStop()` bypasses the "confirm on stop" prompt.
- No auto-start exists. The host's Start button calls `RequestStartGame()`; we can call it ourselves.
- The netplay handshake compares the full 40-character git commit hash. All players must present the same hash.
- Config keys exist for `HostCode`, `Address`, `ConnectPort`, `HostPort`, `ListenPort`, `TraversalServer`, `TraversalPort`, `TraversalPortAlt`, `TraversalChoice`, `Nickname` under `Main.NetPlay`. Default traversal server is `stun.dolphin-emu.org`; the traversal server source is in-tree and self-hostable. Host codes are 8 hex characters.
- The host's traversal host code is available via `Common::g_TraversalClient->GetHostID()`; `NetPlayServer::OnTraversalStateChanged` is the hook point.
- Dolphin's on-screen UI draws ImGui windows each frame from `OnScreenUI::DrawDebugText` (the netplay chat and golf overlays live there). ImGui receives only mouse and keyboard from Qt; no gamepad feed exists.
- Hotkeys bind to ControllerInterface devices. The GameCube adapter is not a ControllerInterface device in this fork. Emulated pad state is readable via `Pad::GetStatus(port)` and `GCAdapter::Input(chan)`.
- Available in-tree libraries: libcurl via `Common::HttpRequest` (synchronous, buffered), ENet, picojson, gtest. No WebSocket client.
- The fork's updater polls the P+ GitHub releases API; a BrawlLive build must redirect it or stock P+ would overwrite our exe.
- The P+ codeset is closed source. A Brawl decompilation exists (about 1%, targets USA Rev 2). No public address list covers stocks, winner or game end. Brawlback runs a shared Ghidra project for Brawl and has a working EXI-device bridge pattern.
- Nobody has shipped an in-game queue for P+ on the official lockstep build. Orca (closed-source rollback fork, alpha) and Brawlback (rollback, no matchmaking) are the adjacent projects.

## 4. Decisions made

| Decision | Choice | Why |
|---|---|---|
| Client footprint | Drop-in patched `Dolphin.exe` | Only path to a one-button queue |
| Where the UI lives | In-game overlay (v1), native P+ menu (later) | "Online matching via the game" builds community |
| Agent transport | HTTP long-poll via in-tree libcurl | No new dependencies, works through any NAT |
| Netplay identity | Send the upstream P+ commit hash | Keeps BrawlLive users compatible with stock P+ lobbies |
| Core code rule | Patch never touches `Source/Core/Core` except new files under `Source/Core/Core/Config` | Identical emulation makes the identity choice safe; CI enforces |
| Identity | Zero-signup device identity, optional Discord link later | Lowest friction; ranked may require linking later |
| Match transition | Stop current game, host/join, boot in sync | Lockstep netplay must boot together; 10–20 s restart accepted |
| Scope | Casual + ranked, singles + doubles, built in six sub-projects | David's call; decomposition keeps each spec and plan small |

## 5. Architecture

```
+------------------------------+        HTTPS long-poll / events        +--------------------------+
| BrawlLive Dolphin build      | <------------------------------------> | Matchmaking service      |
|  - Agent (UICommon)          |                                        |  - Agent API             |
|  - Overlay (VideoCommon)     |      Dolphin NetPlay (ENet, UDP)       |  - Queues + pairing      |
|  - Pad combo poller          | <---- traversal server (ours) ------>  |  - Match state machine   |
|  - MainWindow glue (Qt)      |                                        |  - Ratings, admin        |
+------------------------------+                                        |  - PostgreSQL            |
                                                                        +--------------------------+
                                                                                  |
                                                                        +--------------------------+
                                                                        | Website (leaderboards,   |
                                                                        | profiles, download)      |
                                                                        +--------------------------+
```

Repositories:
- `brawllive` (this folder): service, website, installer, docs, simulated agent, ops.
- `brawllive-dolphin`: GitHub fork of Project-Plus-Dolphin with the patch, CI and releases. The `upstream/` clone in this folder is a read-only reference and is git-ignored.

## 6. Components

### 6.1 BrawlLive Dolphin build (sub-project 1)

Patch surface, all frontend:
- `Source/Core/UICommon/BrawlLive/`: `Agent` (background thread, long-poll loop, event queue, backoff), `Protocol` (picojson encode/decode, versioned), `MatchState` (client-side state machine), `PadCombo` (input poller and combo detector), `Log` (rotating local log in the user directory).
- `Source/Core/VideoCommon/BrawlLiveUI.{h,cpp}`: ImGui overlay drawn from `OnScreenUI::DrawDebugText`, reading a snapshot the agent publishes under a mutex. Global pointer pattern identical to the golf overlay.
- `Source/Core/DolphinQt/`: public slots on `MainWindow` for `BrawlLiveHost(match)`, `BrawlLiveJoin(match)`, `BrawlLiveStart()`, `BrawlLiveLeave()`; a post-stop continuation hooked to `EmulationStateChanged`; a "BrawlLive" menu with Enable, Nickname, Open log, About; updater redirected to our releases.
- `Source/Core/Core/Config/BrawlLiveSettings.{h,cpp}`: `Enabled`, `ServerURL`, `DeviceToken`, `Region`, `MaxPingMs`, `ComboOpen`, `OverlayScale`. The only files allowed under `Source/Core/Core`.
- `Source/Core/Common/make_scmrev.h.js` and CMake equivalent: honour `BRAWLLIVE_NETPLAY_IDENTITY` so `SCM_REV_STR` equals the upstream P+ commit while `SCM_DESC_STR` reads `v3.2.0-brawllive.N`.

Game selection: the host command names the game-list entry "Project+ Netplay Launcher"; the agent resolves it from the game list model by name and path under `User/Launcher/`. Clients resolve by Dolphin's sync identifier as today.

Transition sequence (host): countdown ends → `ForceStop()` → wait for `Uninitialized` → set traversal config to ours → `NetPlayHost(game)` → on host code, report `hosting` → on expected players present, report `lobby_ready` → on `start` command, set pad mapping and buffer, call `RequestStartGame()` → on boot, report `game_started`. Joiner: countdown ends → `ForceStop()` → wait → set host code and traversal → `NetPlayJoin()` → report `joined` → boot → `game_started`.

Buffer rule: frames = max(P+ minimum from the build's default, ceil(peer ping ms / 8)). The service can override via `set_buffer`.

### 6.2 Matchmaking service (sub-project 2)

Node.js with TypeScript, PostgreSQL. Processes: API (agent long-poll and events, website API, admin), matcher (runs pairing on a tick), reaper (expires stale queue entries and stuck matches). All state in PostgreSQL; a restart loses nothing.

Core tables: `devices`, `players` (device-backed, optional Discord link), `queue_entries`, `matches`, `match_players`, `match_events`, `ratings`, `blocks`, `reports`, `pair_history` (measured ping per pair), `disputes`.

### 6.3 Traversal server

Dolphin's `traversal_server` target built from the fork, run as a container on the VPS, ports 6262 and 6226 UDP. BrawlLive sessions set `TraversalServer` to ours for the session only and restore the stock default afterwards.

### 6.4 Website (sub-project 3)

Download page, install guide, leaderboards, player pages, match history, status page. Later: Discord link and dispute view. Same repository as the service.

### 6.5 Installer (sub-project 1)

PowerShell script v1: locate the P+ folder, verify the stock `Dolphin.exe` hash matches a known P+ release, back it up as `Dolphin.exe.stock`, copy the BrawlLive exe in, print next steps. Uninstall restores the backup. A signed installer comes later.

## 7. Agent protocol v1

Base URL from `BrawlLive.ServerURL` (build-time default). All requests over TLS with `Authorization: Bearer <device_token>`, `X-BrawlLive-Build: <desc string>`, `X-BrawlLive-Identity: <netplay hash>`.

Registration (first run, no token yet):
- `POST /v1/devices` `{nickname, build, identity, platform}` → `{device_id, device_token}`. Token is 256 random bits, stored hashed server-side, written to `BrawlLive.DeviceToken`.

Commands (server → agent):
- `GET /v1/agent/commands?after=<seq>&wait=25` → `200 {commands:[{seq, id, type, payload}], now}` or `204` when the wait elapses. The agent passes the highest seq it has processed; the server returns only newer, unexpired commands. Reconnect with `after=0` re-delivers pending commands.
- Types: `queue_status {position, searching, est_wait_s}`, `match_found {match_id, mode, role, players[{slot, nickname, rating?}], ping_estimate_ms, countdown_s}`, `host {match_id, traversal{host, port, alt_port}, expected_players, pad_map[{slot, port}], buffer{min, divisor}}`, `join {match_id, host_code, traversal}`, `start {match_id, buffer_frames?}`, `set_buffer {match_id, frames}`, `cancel {match_id, reason}`, `notice {text, level}`.

Events (agent → server):
- `POST /v1/agent/events` `{events:[{client_seq, at, type, payload}]}` → `{accepted_through}`. Duplicates by `client_seq` are ignored.
- Types: `online {build, identity, pplus_assets_hash, controller_kind}` where `pplus_assets_hash` combines Dolphin's existing SD card and Brawl file sync identifiers, `queue_join {mode, region, max_ping_ms}`, `queue_leave`, `accept {match_id}`, `decline {match_id}`, `hosting {match_id, host_code}`, `joined {match_id}`, `player_joined {match_id, nickname, ping_ms}`, `lobby_ready {match_id, players[]}`, `game_started {match_id}`, `game_ended {match_id, reason, duration_s}`, `left {match_id}`, `result_claim {match_id, winner_slot, games[{winner_slot, stage?}]}`, `error {match_id?, code, detail}`.

Rules: commands are idempotent and carry explicit `match_id`; the agent ignores commands for a match it has left; events are retried with backoff until accepted; the agent logs every command and event locally.

## 8. Match lifecycle

States on the service: `queued → matched → accepting → hosting → joining → lobby → starting → in_game → ended | failed`. Ranked adds `result_pending → resolved | disputed`.

Timeouts and recoveries:
- `accepting`: both players accept within the countdown (5 s auto-accept unless cancelled) or back to `queued`.
- `hosting`: `hosting` event within 20 s, else swap host roles once, then `failed(host_timeout)` and requeue both.
- `joining`: `player_joined` from host within 30 s, else the joiner retries on the alternate traversal port once, then `failed(join_timeout)`.
- `lobby`: all expected players present; the service sends `start`.
- `starting`: `game_started` from every player within 20 s, else `failed(start_timeout)`.
- `in_game`: ends on `game_ended` from any player. Casual: offer rematch or requeue. Ranked: a game shorter than 60 s counts as void.
- Any `failed` returns the players to `queued` automatically unless they cancelled. A device with three failures in ten minutes gets a NAT hint and lower host preference.

Host selection: the player with the better hosting success rate and the lower NAT failure count; ties random; swap once on failure.

Regions are a fixed list (EU-West, EU-East, NA-East, NA-West, South America, Asia, Oceania) with an adjacency table in service config; a player picks one and the service verifies it loosely against IP geolocation.

Pairing (matcher tick every 2 s): candidates share a queue (mode and bracket), region equal or adjacent, ping band satisfied using `pair_history` when known and region distance otherwise; the band widens every 15 s of waiting; blocked pairs never match; no immediate rematch with the same opponent unless both chose rematch. Ranked adds a rating window that widens with wait time. Doubles fills four slots with two teams; team rating uses a team-capable system chosen in sub-project 4.

## 9. In-game UI and input

Overlay: a compact ImGui panel in a corner, scaled by display size and `OverlayScale`, hideable. Lines: connection state, queue state with wait time, match found with opponent and ping, countdown, last error with code, hints for the active combos. Drawn only while a game runs because the overlay lives in the presenter; before any game runs, the Qt main window shows the same state in its status bar.

Input: a frontend thread polls emulated pad state at 60 Hz for ports 1–4, reading `GCAdapter::Input` when the port's SI device is the adapter and `Pad::GetStatus` otherwise. Combos default to: hold L + R + Start for one second to open or close the menu; D-pad up and down to move; A to confirm; B to back; hold the open combo for three seconds in a netplay session to leave the match. Combos are configurable; keyboard hotkeys mirror them. During a netplay session every combo except leave is disabled.

Flow: open menu → choose queue (Casual singles, Ranked singles, Casual doubles, Ranked doubles) → searching, player may keep playing offline → match found with countdown and cancel → restart into the session → after the match, rematch or requeue.

## 10. Error handling

- Agent network calls have connect and total timeouts; retries use exponential backoff with jitter capped at 30 s; the overlay shows "offline" after two failures and recovers silently.
- Every failure has a code (`E_HOST_TIMEOUT`, `E_JOIN_REJECTED`, `E_TRAVERSAL_TIMEOUT`, `E_VERSION_MISMATCH`, `E_GAME_NOT_FOUND`, `E_START_TIMEOUT`, `E_DISCONNECT`, `E_DESYNC`) shown in the overlay and written to the local log with context.
- Version drift: the agent reports build, identity and a hash of the P+ assets it will sync; the service pairs only identical values.
- Netplay stop reasons from Dolphin map to `game_ended.reason`.
- Service: all state transitions in transactions; the reaper expires queue entries without a poll for 60 s and matches stuck in any pre-game state past its timeout.

## 11. Security and abuse

- Device tokens: random, hashed at rest, rotatable, revocable by admin. TLS only. No secrets in logs.
- Host codes are sent only to matched peers and expire with the match.
- Rate limits per device and per IP on registration, queueing and events.
- Block and report from the overlay; blocked pairs never match; reports queue for admin review.
- Ranked results require agreement of all players; disagreement opens a dispute; repeated disputes lower trust.
- No game files, saves or ISOs ever touch our infrastructure. The fork source is public with build instructions, satisfying the GPL.
- Data minimisation: nickname, region, ping measurements, match outcomes. IP addresses only for rate limiting and NAT diagnostics, retained briefly.
- Admin surface behind an allowlist of linked Discord accounts.

## 12. Testing

- Service: unit tests for pairing, the state machine, timeouts and ratings; API contract tests driven by a simulated agent in TypeScript that speaks protocol v1, so CI plays full matches without Dolphin.
- Dolphin patch: gtest units for protocol parsing, the client state machine and the combo detector; a build-time path guard that fails if `Source/Core/Core` changes outside `Config/BrawlLiveSettings.*`.
- Milestone test on David's PC: a PowerShell harness launches two Dolphin instances with separate `--user` folders, the P+ 3.2 build and David's Rev 1 ISO, against a local service, enqueues both devices and asserts both report `game_started` within 90 s. Repeat with roles swapped and with an induced host failure.
- Compatibility test: the BrawlLive build joins a stock P+ 3.2 host by code and plays a match.
- Beta: a closed EU group before public release; failure codes and ping data reviewed before widening.

## 13. Operations and release

- One EU VPS with Docker Compose: API, matcher and reaper, PostgreSQL, traversal server, TLS reverse proxy. Nightly offsite backups, uptime checks, structured logs, basic metrics (queue depth, time to match, failure rate by code).
- GitHub Actions on `brawllive-dolphin`: Windows 2022 runner, submodules, `msbuild` Release x64, env `DOLPHIN_DISTRIBUTOR=BrawlLive` and `BRAWLLIVE_NETPLAY_IDENTITY=<upstream hash>`, path guard, artifact with hashes, release on tag.
- The BrawlLive build's updater points at our releases. Each P+ release gets a matching BrawlLive build within days; the service refuses pairs on stale identities with a clear message.
- Domain, hosting account and TLS are chosen in sub-project 6.

## 14. Sub-projects and order

1. **Dolphin agent patch and build pipeline.** Everything in 6.1 and 6.5, protocol v1 client side, the simulated service stub needed for the milestone test, CI with path guard. Exit: the two-instance milestone test passes on David's PC.
2. **Matchmaking service core.** 6.2 and 6.3 for casual singles, protocol v1 server side, simulated-agent contract tests, local Compose. Exit: two real agents matched through the service.
3. **Website.** Download, guide, status, leaderboards shell, player pages.
4. **Ranked.** Ratings, result claims and disputes, trust rules; then result capture v2 by reading game state inside Dolphin, which starts with a reverse-engineering spike using Dolphin Memory Engine, the Brawl decompilation and Brawlback's Ghidra project.
5. **Doubles.** Four-slot matches, pad map and team assignment, team rating.
6. **Operations and public launch.** Domain, VPS, monitoring, installer polish, beta, launch.

Later, outside v1: native P+ "Online" menu via a custom EXI device and injected codes alongside the closed P+ codeset, following the Brawlback bridge pattern; Discord account linking; macOS and Linux builds.

## 15. Risks and mitigations

- P+ releases change netplay code: the patch is small and isolated; the fork tracks tags; REA diffs of the stock exe flag netplay changes.
- Traversal and NAT failures: our own traversal server, alternate port retry, host swap, NAT hints, pair history steering.
- Restart UX feels slow: countdown with clear messaging; later a boot-to-character-select code once scene IDs are verified for P+.
- Identity spoofing risk from sending the upstream hash: the core-path CI guard and a release checklist item that diffs `Source/Core/Core` against the upstream tag.
- Smurfing with zero-signup identity: trust scores, optional Discord link, possible requirement for ranked later.
- Legal: no game files distributed, GPL compliance, no Nintendo branding. The P+ team is informed before public launch.
- Two-instance testing on one PC needs two controller configurations; the harness assigns keyboard to one instance.

## 16. Glossary

- P+: Project+, the Brawl mod. Launcher: the `.dol` entry Dolphin boots to start P+.
- Host code: 8-character traversal ID identifying a netplay host.
- Lockstep netplay: Dolphin's input-delay netcode; all players boot together and run in sync.
- Agent: the headless BrawlLive client inside Dolphin.
- Overlay: the ImGui panel BrawlLive draws over the game.
- Identity: the commit hash Dolphin sends during the netplay handshake.
