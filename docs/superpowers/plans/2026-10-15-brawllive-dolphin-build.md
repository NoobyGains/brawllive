# BrawlLive Dolphin Build (Sub-project 1) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** A drop-in `Dolphin.exe` for the Project+ 3.2 folder that, driven by a matchmaking server, stops the current game, hosts or joins a netplay session, auto-starts it, and reports events; plus the installer, a stub server and harness that prove it on David's PC, and CI that builds it with the core-path guard.

**Architecture:** Fork of Project-Plus-Dolphin at tag `v3.2.0` with a frontend-only patch: pure logic modules in `UICommon/BrawlLive` (protocol, state machine, pad-combo detector, agent thread over libcurl), an ImGui overlay in `VideoCommon`, and a Qt controller in `DolphinQt` that executes host/join/start through the existing `MainWindow` and `NetPlayServer` entry points. A zero-dependency Node stub plays the server role for local tests.

**Tech Stack:** C++20 (MSVC v143, Dolphin code style), Qt 6.5.1 (prebuilt submodule), picojson, libcurl via `Common::HttpRequest`, gtest (in-tree), Node.js 24 TypeScript via native type stripping with `node --test`, PowerShell 7, GitHub Actions `windows-2022`.

**Spec:** `D:\Projects\Games\BRAWLLIVE\docs\superpowers\specs\2026-10-15-brawllive-matchmaking-design.md` (sections 6.1, 6.5, 7, 8, 9, 10, 12, 13, 14.1)

## Global Constraints

- Fork source: `https://github.com/Project-Plus-Development-Team/Project-Plus-Dolphin`, tag `v3.2.0` = commit `448d1dbee83429f6a64f269a3511dc9a5fb11a98`. Work on branch `brawllive` in David's fork `brawllive-dolphin`, cloned to `D:\Projects\Games\brawllive-dolphin` (sibling of the `brawllive` repo, never inside it).
- Netplay identity: the built exe must embed `SCM_REV_STR "448d1dbee83429f6a64f269a3511dc9a5fb11a98"` (upstream hash) while `SCM_DESC_STR` is our tag `v3.2.0-brawllive.N`.
- Core rule: no file under `Source/Core/Core/` may change except new files `Source/Core/Core/Config/BrawlLiveSettings.h` and `.cpp`. CI and a local script enforce it.
- Windows build: `Source\dolphin-emu.sln`, `msbuild /p:Configuration=Release /p:Platform=x64`, VS 2022 Build Tools at `G:\BuildTools` (toolset v143), Windows SDK 10.0.26100. Output `Binary\x64\Dolphin.exe`. Unit tests: `Build\x64\Release\UnitTests\bin\UnitTests.exe` (verify in Task 1).
- New C++ files carry the Dolphin SPDX header: `// Copyright 2026 BrawlLive Contributors` / `// SPDX-License-Identifier: GPL-2.0-or-later`. 2-space indent, `u8/u16/u32/u64` from `Common/CommonTypes.h`, `std::string` not `QString` outside DolphinQt.
- Every new source file is registered in three places: `Source/Core/DolphinLib.props` (Windows), the matching `CMakeLists.txt` (other platforms), and for tests `Source/UnitTests/UnitTests.vcxproj` plus `Source/UnitTests/<dir>/CMakeLists.txt`.
- Node tooling: Node 24.19 on David's PC; TypeScript files run natively (type stripping is default and stable since 24.12). Use only erasable syntax: no `enum`, no `namespace`, no parameter properties. Tests use `node --test`. Zero npm dependencies.
- CI runner: `windows-2022` (still ships Visual Studio 2022; `windows-latest` is Server 2025 with VS 2026).
- Game files: never commit, upload or copy into any repo or artifact. The Brawl ISO lives only at `D:\Projects\Games\BRAWLLIVE\Super Smash Bros. Brawl (USA) (Rev 1).7z` and its extraction.
- The `brawllive` repo (`D:\Projects\Games\BRAWLLIVE`) holds `tools/stub-service`, `tools/installer`, `tools/milestone` and docs. Commits end with `Co-Authored-By: Claude Fable 5.1 <noreply@anthropic.com>`.

## Review Focus

1. Malformed, truncated or huge JSON from the server must be rejected without crashing or acting. Test in Task 3 (`Protocol, RejectsMalformedAndOversized`).
2. A controller unplugging mid-hold, or a port reporting not connected, must reset the combo and never fire an action. Test in Task 4 (`PadCombo, DisconnectResetsHold`).
3. A late command for a match the client already left or that was cancelled must be ignored. Test in Task 5 (`MatchState, IgnoresStaleMatchCommands`).
4. Server unreachable for minutes at startup: backoff caps at 30 s, status shows offline, and the first successful poll recovers silently. Test in Task 6 (`Agent, BackoffCapsAndRecovers`).
5. The P+ launcher missing from the game list (player opened the wrong folder) must fail the match with `E_GAME_NOT_FOUND` and return to idle, not hang in Preparing. Test in Task 5 (`MatchState, GameNotFoundFailsToIdle`) and manual check in Task 9.

---

## File Structure

Fork `brawllive-dolphin` (new or modified):

| Path | Responsibility |
|---|---|
| `Source/Core/Core/Config/BrawlLiveSettings.{h,cpp}` | Config keys under `[BrawlLive]` |
| `Source/Core/UICommon/BrawlLive/Protocol.{h,cpp}` | JSON command/event types, parse and serialize |
| `Source/Core/UICommon/BrawlLive/PadCombo.{h,cpp}` | Pure combo detector over pad button masks |
| `Source/Core/UICommon/BrawlLive/MatchState.{h,cpp}` | Client-side match state machine, pending actions |
| `Source/Core/UICommon/BrawlLive/Transport.{h,cpp}` | `ITransport` interface and libcurl implementation |
| `Source/Core/UICommon/BrawlLive/Agent.{h,cpp}` | Background thread: registration, long-poll, event delivery |
| `Source/Core/VideoCommon/BrawlLiveUI.{h,cpp}` | Overlay snapshot type, line formatter, ImGui draw |
| `Source/Core/VideoCommon/OnScreenUI.cpp` | Two-line hook calling the overlay |
| `Source/Core/DolphinQt/BrawlLive/BrawlLiveController.{h,cpp}` | Qt glue: timers, pad polling, host/join/start, snapshot publishing |
| `Source/Core/DolphinQt/MainWindow.{h,cpp}` | `friend class BrawlLiveController;` construction, updater URL |
| `Source/Core/DolphinQt/MenuBar.{h,cpp}` | "BrawlLive" top-level menu |
| `Source/Core/Common/make_scmrev.h.js`, `CMake/ScmRevGen.cmake` | Identity override via env var |
| `Source/Core/DolphinLib.props`, `*/CMakeLists.txt`, `DolphinQt.vcxproj` | Registration |
| `Source/UnitTests/UICommon/BrawlLive/*Test.cpp`, `Source/UnitTests/VideoCommon/BrawlLiveUITest.cpp` | gtest |
| `build.ps1` | One-command Windows build with identity env |
| `tools/core-path-guard.ps1` | Fails if forbidden core paths changed |
| `.github/workflows/brawllive.yml` | Guard, build, tests, artifact, release |
| `BRAWLLIVE.md` | Build and patch notes for the fork |

Repo `brawllive`:

| Path | Responsibility |
|---|---|
| `tools/stub-service/server.ts`, `tools/stub-service/server.test.ts`, `tools/stub-service/package.json` | Protocol v1 stub server and tests |
| `tools/installer/Install-BrawlLive.ps1`, `Uninstall-BrawlLive.ps1`, `Install-BrawlLive.Tests.ps1` | Installer |
| `tools/milestone/Run-TwoInstance.ps1` | Exit-criterion harness |

---

### Task 1: Fork, baseline build, test run, baseline runtime check

**Files:**
- Create: `D:\Projects\Games\brawllive-dolphin\BRAWLLIVE.md`
- Create: `D:\Projects\Games\brawllive-dolphin\build.ps1`

**Interfaces:**
- Produces: a working local build pipeline (`build.ps1 -Configuration Release`) that later tasks run after every change, and the verified path of `UnitTests.exe`.

- [ ] **Step 1: Create the fork and clone it with submodules**

Run (requires `gh auth status` to show David's account; if not logged in, stop and ask David to run `gh auth login`):

```powershell
gh repo fork Project-Plus-Development-Team/Project-Plus-Dolphin --fork-name brawllive-dolphin --clone=false
git clone --branch v3.2.0 --recurse-submodules --shallow-submodules https://github.com/NoobyGains/brawllive-dolphin.git D:\Projects\Games\brawllive-dolphin
cd D:\Projects\Games\brawllive-dolphin
git switch -c brawllive
git log -1 --format=%H
```
Expected: `448d1dbee83429f6a64f269a3511dc9a5fb11a98`. `NoobyGains` is David's GitHub login as printed by `gh api user --jq .login` on 2026-10-15.

- [ ] **Step 2: Write `build.ps1`**

```powershell
#Requires -Version 7
<#
.SYNOPSIS  Build BrawlLive Dolphin (Windows x64) with the netplay identity override.
.PARAMETER Configuration  Release (default) or Debug
.PARAMETER Tests          Also build UnitTests and run them
#>
param(
  [ValidateSet('Release','Debug')] [string] $Configuration = 'Release',
  [switch] $Tests
)
$ErrorActionPreference = 'Stop'
$root = $PSScriptRoot
$vsDevShell = 'G:\BuildTools\Common7\Tools\Launch-VsDevShell.ps1'
if (-not (Test-Path $vsDevShell)) { throw "VS 2022 Build Tools not found at $vsDevShell" }
& $vsDevShell -Arch amd64 -SkipAutomaticLocation | Out-Null

# Netplay identity: upstream P+ v3.2.0 commit. Our frontend-only patch keeps emulation identical.
$env:BRAWLLIVE_NETPLAY_IDENTITY = '448d1dbee83429f6a64f269a3511dc9a5fb11a98'
$env:DOLPHIN_DISTRIBUTOR = 'BrawlLive'

$targets = 'DolphinQt'
if ($Tests) { $targets = 'DolphinQt;UnitTests' }
msbuild "$root\Source\dolphin-emu.sln" /m /verbosity:minimal /t:$targets `
  /p:Configuration=$Configuration /p:Platform=x64
if ($LASTEXITCODE -ne 0) { throw "msbuild failed ($LASTEXITCODE)" }

$exe = Join-Path $root "Binary\x64\Dolphin.exe"
if (-not (Test-Path $exe)) { throw "Build produced no $exe" }
Write-Host "Built $exe"

if ($Tests) {
  $test = Get-ChildItem -Recurse "$root\Build\x64\$Configuration" -Filter 'UnitTests*.exe' | Select-Object -First 1
  if (-not $test) { throw 'UnitTests exe not found under Build\x64' }
  Write-Host "Running $($test.FullName)"
  & $test.FullName --gtest_brief=1
  if ($LASTEXITCODE -ne 0) { throw "Unit tests failed ($LASTEXITCODE)" }
}
```

- [ ] **Step 3: Run the baseline build with tests**

Run: `pwsh -File .\build.ps1 -Tests`
Expected: `Built D:\Projects\Games\brawllive-dolphin\Binary\x64\Dolphin.exe`, then gtest output ending in `[  PASSED  ]`. Note the printed `UnitTests*.exe` path and write it into `BRAWLLIVE.md` (Step 5). A cold build takes 30–60 minutes. If `make_scmrev.h.js` fails, ensure `git` is on PATH inside the dev shell.

- [ ] **Step 4: Baseline runtime check with the stock P+ assets**

```powershell
$work = 'D:\Projects\Games\BRAWLLIVE\work\pplus-baseline'
robocopy 'D:\Projects\Games\BRAWLLIVE\Project+.v3.2.Netplay.Windows' $work /E /NFL /NDL /NJH /NJS | Out-Null
Copy-Item 'D:\Projects\Games\brawllive-dolphin\Binary\x64\Dolphin.exe' "$work\Dolphin.exe" -Force
Rename-Item "$work\user\Cache (rename folder to just Cache to use)" "$work\user\Cache" -ErrorAction SilentlyContinue
Start-Process "$work\Dolphin.exe"
```
Expected: Dolphin opens with the P+ theme and the two launcher entries in the game list. Open Help → About: it must read "Project+ Dolphin v3.2.0" and show revision `448d1dbe…`. Close Dolphin. (The ISO is not configured yet; booting is checked in Task 13.) Add `work/` to `D:\Projects\Games\BRAWLLIVE\.gitignore`.

- [ ] **Step 5: Write `BRAWLLIVE.md`**

```markdown
# BrawlLive Dolphin

Fork of Project+ Dolphin v3.2.0 (commit 448d1dbe) with a frontend-only patch
that adds in-game online matchmaking. See the BrawlLive spec in the `brawllive`
repo (`docs/superpowers/specs/2026-10-15-brawllive-matchmaking-design.md`).

## Build (Windows)
- VS 2022 Build Tools (v143), Windows SDK 10.0.26100, git on PATH.
- `pwsh -File build.ps1 -Tests` builds `Binary\x64\Dolphin.exe` and runs
  `<path printed in Task 1 Step 3>`.

## Rules
- Never change files under `Source/Core/Core/` except
  `Source/Core/Core/Config/BrawlLiveSettings.{h,cpp}`. `tools/core-path-guard.ps1`
  and CI enforce this. Reason: the build sends the upstream commit hash as its
  netplay identity, which is only honest while emulation code is identical.
- Register every new source in `Source/Core/DolphinLib.props` and the CMakeLists.

## Patch map
Documented per task in the plan; summary kept current here.
```

- [ ] **Step 6: Commit**

```powershell
git add BRAWLLIVE.md build.ps1
git commit -m "chore: BrawlLive build script and fork notes

Co-Authored-By: Claude Fable 5.1 <noreply@anthropic.com>"
```

### Task 2: Config section, build registration pattern, first unit test

**Files:**
- Create: `Source/Core/Core/Config/BrawlLiveSettings.h`, `Source/Core/Core/Config/BrawlLiveSettings.cpp`
- Create: `Source/UnitTests/UICommon/BrawlLive/BrawlLiveSettingsTest.cpp`, `Source/UnitTests/UICommon/CMakeLists.txt`, `Source/UnitTests/UICommon/BrawlLive/CMakeLists.txt`
- Modify: `Source/Core/DolphinLib.props` (near lines 206 and 917), `Source/Core/Core/CMakeLists.txt` (next to `Config/NetplaySettings.cpp`), `Source/UnitTests/UnitTests.vcxproj` (ClCompile list), `Source/UnitTests/CMakeLists.txt` (add `add_subdirectory(UICommon)`)
- Create: `tools/core-path-guard.ps1`

**Interfaces:**
- Produces: `Config::BRAWLLIVE_ENABLED` (bool, default true), `BRAWLLIVE_SERVER_URL` (string, default from macro `BRAWLLIVE_DEFAULT_SERVER_URL` or `""`), `BRAWLLIVE_DEVICE_TOKEN` (string, `""`), `BRAWLLIVE_REGION` (string, `"EU-West"`), `BRAWLLIVE_MAX_PING_MS` (u32, 120), `BRAWLLIVE_OPEN_HOLD_MS` (u32, 1000), `BRAWLLIVE_OVERLAY_SCALE` (float, 1.0f), `BRAWLLIVE_AUTO_QUEUE` (string, `""`; test hook: queue this mode once online).

- [ ] **Step 1: Write the failing test**

`Source/UnitTests/UICommon/BrawlLive/BrawlLiveSettingsTest.cpp`:
```cpp
// Copyright 2026 BrawlLive Contributors
// SPDX-License-Identifier: GPL-2.0-or-later

#include <gtest/gtest.h>

#include "Common/Config/Config.h"
#include "Core/Config/BrawlLiveSettings.h"

TEST(BrawlLiveSettings, Defaults)
{
  EXPECT_TRUE(Config::Get(Config::BRAWLLIVE_ENABLED));
  EXPECT_EQ(Config::Get(Config::BRAWLLIVE_DEVICE_TOKEN), "");
  EXPECT_EQ(Config::Get(Config::BRAWLLIVE_REGION), "EU-West");
  EXPECT_EQ(Config::Get(Config::BRAWLLIVE_MAX_PING_MS), 120u);
  EXPECT_EQ(Config::Get(Config::BRAWLLIVE_OPEN_HOLD_MS), 1000u);
  EXPECT_FLOAT_EQ(Config::Get(Config::BRAWLLIVE_OVERLAY_SCALE), 1.0f);
  EXPECT_EQ(Config::Get(Config::BRAWLLIVE_AUTO_QUEUE), "");
}

TEST(BrawlLiveSettings, KeysLiveInBrawlLiveSection)
{
  EXPECT_EQ(Config::BRAWLLIVE_SERVER_URL.GetLocation().section, "BrawlLive");
  EXPECT_EQ(Config::BRAWLLIVE_SERVER_URL.GetLocation().key, "ServerURL");
  EXPECT_EQ(Config::BRAWLLIVE_SERVER_URL.GetLocation().system, Config::System::Main);
}
```
Register it: in `Source/UnitTests/UnitTests.vcxproj` add `<ClCompile Include="UICommon\BrawlLive\BrawlLiveSettingsTest.cpp" />` after the last `Common\...Test.cpp` line. Create `Source/UnitTests/UICommon/CMakeLists.txt` with `add_subdirectory(BrawlLive)` and `Source/UnitTests/UICommon/BrawlLive/CMakeLists.txt` with `add_dolphin_test(BrawlLiveSettingsTest BrawlLiveSettingsTest.cpp)`; add `add_subdirectory(UICommon)` to `Source/UnitTests/CMakeLists.txt` next to the existing `add_subdirectory(Core)`.

- [ ] **Step 2: Run the test to verify it fails**

Run: `pwsh -File .\build.ps1 -Tests`
Expected: compile error `Core/Config/BrawlLiveSettings.h: No such file`.

- [ ] **Step 3: Write the config module**

`Source/Core/Core/Config/BrawlLiveSettings.h`:
```cpp
// Copyright 2026 BrawlLive Contributors
// SPDX-License-Identifier: GPL-2.0-or-later

#pragma once

#include <string>

#include "Common/CommonTypes.h"
#include "Common/Config/Config.h"

namespace Config
{
// [BrawlLive] section of Dolphin.ini. Frontend-only feature; see BRAWLLIVE.md.
extern const Info<bool> BRAWLLIVE_ENABLED;
extern const Info<std::string> BRAWLLIVE_SERVER_URL;
extern const Info<std::string> BRAWLLIVE_DEVICE_TOKEN;
extern const Info<std::string> BRAWLLIVE_REGION;
extern const Info<u32> BRAWLLIVE_MAX_PING_MS;
extern const Info<u32> BRAWLLIVE_OPEN_HOLD_MS;
extern const Info<float> BRAWLLIVE_OVERLAY_SCALE;
extern const Info<std::string> BRAWLLIVE_AUTO_QUEUE;
}  // namespace Config
```

`Source/Core/Core/Config/BrawlLiveSettings.cpp`:
```cpp
// Copyright 2026 BrawlLive Contributors
// SPDX-License-Identifier: GPL-2.0-or-later

#include "Core/Config/BrawlLiveSettings.h"

#ifndef BRAWLLIVE_DEFAULT_SERVER_URL
#define BRAWLLIVE_DEFAULT_SERVER_URL ""
#endif

namespace Config
{
const Info<bool> BRAWLLIVE_ENABLED{{System::Main, "BrawlLive", "Enabled"}, true};
const Info<std::string> BRAWLLIVE_SERVER_URL{{System::Main, "BrawlLive", "ServerURL"},
                                             BRAWLLIVE_DEFAULT_SERVER_URL};
const Info<std::string> BRAWLLIVE_DEVICE_TOKEN{{System::Main, "BrawlLive", "DeviceToken"}, ""};
const Info<std::string> BRAWLLIVE_REGION{{System::Main, "BrawlLive", "Region"}, "EU-West"};
const Info<u32> BRAWLLIVE_MAX_PING_MS{{System::Main, "BrawlLive", "MaxPingMs"}, 120};
const Info<u32> BRAWLLIVE_OPEN_HOLD_MS{{System::Main, "BrawlLive", "OpenHoldMs"}, 1000};
const Info<float> BRAWLLIVE_OVERLAY_SCALE{{System::Main, "BrawlLive", "OverlayScale"}, 1.0f};
const Info<std::string> BRAWLLIVE_AUTO_QUEUE{{System::Main, "BrawlLive", "AutoQueue"}, ""};
}  // namespace Config
```
Register: `Source/Core/DolphinLib.props` add `<ClInclude Include="Core\Config\BrawlLiveSettings.h" />` next to line 206 and `<ClCompile Include="Core\Config\BrawlLiveSettings.cpp" />` next to line 917. `Source/Core/Core/CMakeLists.txt`: add `Config/BrawlLiveSettings.cpp` and `Config/BrawlLiveSettings.h` beside the `Config/NetplaySettings.*` entries. (This CMakeLists edit is a registration line in `Source/Core/Core/CMakeLists.txt`; the guard in Step 5 allows that single file.)

- [ ] **Step 4: Run the tests to verify they pass**

Run: `pwsh -File .\build.ps1 -Tests`
Expected: `[  PASSED  ]` including `BrawlLiveSettings.Defaults` and `BrawlLiveSettings.KeysLiveInBrawlLiveSection`.

- [ ] **Step 5: Write the core-path guard**

`tools/core-path-guard.ps1`:
```powershell
#Requires -Version 7
<# Fails (exit 1) if any path under Source/Core/Core changed since the upstream tag,
   except the BrawlLive config pair and the Core CMakeLists registration line. #>
param([string] $Base = 'v3.2.0', [string] $Head = 'HEAD')
$ErrorActionPreference = 'Stop'
$allowed = @(
  'Source/Core/Core/Config/BrawlLiveSettings.h',
  'Source/Core/Core/Config/BrawlLiveSettings.cpp',
  'Source/Core/Core/CMakeLists.txt'
)
$changed = git diff --name-only "$Base...$Head" | Where-Object { $_ -like 'Source/Core/Core/*' }
$violations = $changed | Where-Object { $allowed -notcontains $_ }
if ($violations) {
  Write-Host 'Forbidden changes under Source/Core/Core:' -ForegroundColor Red
  $violations | ForEach-Object { Write-Host "  $_" }
  exit 1
}
# The CMakeLists change must be exactly the two registration lines.
if ($changed -contains 'Source/Core/Core/CMakeLists.txt') {
  $added = git diff "$Base...$Head" -- Source/Core/Core/CMakeLists.txt |
    Where-Object { $_ -match '^\+[^+]' } | ForEach-Object { $_.Substring(1).Trim() }
  $bad = $added | Where-Object { $_ -notmatch '^Config/BrawlLiveSettings\.(h|cpp)$' }
  if ($bad) { Write-Host "Unexpected CMakeLists additions: $bad" -ForegroundColor Red; exit 1 }
}
Write-Host 'core-path-guard: OK'
exit 0
```
Run: `pwsh -File tools\core-path-guard.ps1`
Expected: `core-path-guard: OK`.

- [ ] **Step 6: Commit**

```powershell
git add Source/Core/Core/Config/BrawlLiveSettings.h Source/Core/Core/Config/BrawlLiveSettings.cpp Source/Core/Core/CMakeLists.txt Source/Core/DolphinLib.props Source/UnitTests tools/core-path-guard.ps1
git commit -m "feat(brawllive): config section and core-path guard

Co-Authored-By: Claude Fable 5.1 <noreply@anthropic.com>"
```

---

### Task 3: Protocol v1 types, parse and serialize

**Files:**
- Create: `Source/Core/UICommon/BrawlLive/Protocol.h`, `Source/Core/UICommon/BrawlLive/Protocol.cpp`
- Create: `Source/UnitTests/UICommon/BrawlLive/ProtocolTest.cpp`
- Modify: `Source/Core/DolphinLib.props` (near lines 593 and 1294), `Source/Core/UICommon/CMakeLists.txt` (after `NetPlayIndex.h`), `Source/UnitTests/UnitTests.vcxproj`, `Source/UnitTests/UICommon/BrawlLive/CMakeLists.txt`

**Interfaces:**
- Produces (namespace `BrawlLive`):
  - `enum class CommandType { QueueStatus, MatchFound, Host, Join, Start, SetBuffer, Cancel, Notice, Unknown };`
  - `struct TraversalInfo { std::string host; u16 port = 6262; u16 alt_port = 6226; };`
  - `struct PlayerInfo { int slot = 0; std::string nickname; std::optional<int> rating; };`
  - `struct Command { u64 seq; std::string id; CommandType type; std::string match_id; int position; bool searching; int est_wait_s; std::string mode, role; std::vector<PlayerInfo> players; int ping_estimate_ms; int countdown_s; std::optional<TraversalInfo> traversal; int expected_players; std::array<int, 4> pad_map; int buffer_min; int buffer_divisor; std::string game_name; std::string host_code; std::string address; u16 port; std::optional<int> buffer_frames; std::string reason; std::string text; std::string level; }` (all numeric fields default 0, strings empty, `pad_map` `{1,2,0,0}`, `buffer_min` 3, `buffer_divisor` 8).
  - `struct CommandBatch { std::vector<Command> commands; std::string now; };`
  - `std::optional<CommandBatch> ParseCommands(std::string_view json);` returns nullopt on malformed input or input over 256 KiB; unknown command types are kept with `type == Unknown`.
  - `enum class EventType { Online, QueueJoin, QueueLeave, Accept, Decline, Hosting, Joined, PlayerJoined, LobbyReady, GameStarted, GameEnded, Left, Error };`
  - `struct Event { u64 client_seq = 0; std::string at; EventType type; std::string match_id; picojson::object payload; };`
  - `std::string SerializeEvents(const std::vector<Event>& events);` → `{"events":[{client_seq,at,type,match_id?,payload}]}`
  - `std::optional<u64> ParseAcceptedThrough(std::string_view json);`
  - `struct Registration { std::string device_id, device_token; }; std::optional<Registration> ParseRegistration(std::string_view json); std::string SerializeRegistration(const std::string& nickname, const std::string& build, const std::string& identity);`
  - `const char* EventTypeName(EventType)`; `CommandType CommandTypeFromName(std::string_view)`.

- [ ] **Step 1: Write the failing tests**

`Source/UnitTests/UICommon/BrawlLive/ProtocolTest.cpp`:
```cpp
// Copyright 2026 BrawlLive Contributors
// SPDX-License-Identifier: GPL-2.0-or-later

#include <gtest/gtest.h>

#include <string>

#include "UICommon/BrawlLive/Protocol.h"

using namespace BrawlLive;

static const char* kBatch = R"({"now":"2026-10-15T20:00:00Z","commands":[
 {"seq":12,"id":"cmd_a","type":"match_found","payload":{"match_id":"m1","mode":"casual_singles",
  "role":"host","players":[{"slot":0,"nickname":"Dave"},{"slot":1,"nickname":"Zed","rating":1500}],
  "ping_estimate_ms":38,"countdown_s":5}},
 {"seq":13,"id":"cmd_b","type":"host","payload":{"match_id":"m1","traversal":{"host":"t.example","port":6262,"alt_port":6226},
  "expected_players":2,"pad_map":[1,2,0,0],"buffer":{"min":3,"divisor":8},"game_name":"Project+ Netplay Launcher"}},
 {"seq":14,"id":"cmd_c","type":"join","payload":{"match_id":"m1","host_code":"a1b2c3d4"}},
 {"seq":15,"id":"cmd_d","type":"join","payload":{"match_id":"m2","address":"127.0.0.1","port":2626}},
 {"seq":16,"id":"cmd_e","type":"start","payload":{"match_id":"m1","buffer_frames":5}},
 {"seq":17,"id":"cmd_f","type":"wibble","payload":{}}
]})";

TEST(Protocol, ParsesCommandBatch)
{
  const auto batch = ParseCommands(kBatch);
  ASSERT_TRUE(batch.has_value());
  EXPECT_EQ(batch->now, "2026-10-15T20:00:00Z");
  ASSERT_EQ(batch->commands.size(), 6u);

  const Command& found = batch->commands[0];
  EXPECT_EQ(found.seq, 12u);
  EXPECT_EQ(found.type, CommandType::MatchFound);
  EXPECT_EQ(found.match_id, "m1");
  EXPECT_EQ(found.role, "host");
  ASSERT_EQ(found.players.size(), 2u);
  EXPECT_EQ(found.players[1].nickname, "Zed");
  EXPECT_EQ(found.players[1].rating.value_or(-1), 1500);
  EXPECT_EQ(found.countdown_s, 5);

  const Command& host = batch->commands[1];
  EXPECT_EQ(host.type, CommandType::Host);
  ASSERT_TRUE(host.traversal.has_value());
  EXPECT_EQ(host.traversal->host, "t.example");
  EXPECT_EQ(host.traversal->alt_port, 6226);
  EXPECT_EQ(host.expected_players, 2);
  EXPECT_EQ(host.pad_map, (std::array<int, 4>{1, 2, 0, 0}));
  EXPECT_EQ(host.buffer_min, 3);
  EXPECT_EQ(host.buffer_divisor, 8);
  EXPECT_EQ(host.game_name, "Project+ Netplay Launcher");

  EXPECT_EQ(batch->commands[2].host_code, "a1b2c3d4");
  EXPECT_FALSE(batch->commands[2].traversal.has_value());
  EXPECT_EQ(batch->commands[3].address, "127.0.0.1");
  EXPECT_EQ(batch->commands[3].port, 2626);
  EXPECT_EQ(batch->commands[4].buffer_frames.value_or(0), 5);
  EXPECT_EQ(batch->commands[5].type, CommandType::Unknown);
}

TEST(Protocol, RejectsMalformedAndOversized)
{
  EXPECT_FALSE(ParseCommands("{not json").has_value());
  EXPECT_FALSE(ParseCommands(R"({"commands":"nope"})").has_value());
  EXPECT_FALSE(ParseCommands(R"({"commands":[{"seq":"x","type":"start"}]})").has_value());
  const std::string huge(256 * 1024 + 1, ' ');
  EXPECT_FALSE(ParseCommands(huge).has_value());
  EXPECT_TRUE(ParseCommands(R"({"commands":[]})").has_value());
}

TEST(Protocol, SerializesEvents)
{
  Event e;
  e.client_seq = 7;
  e.at = "2026-10-15T20:00:01Z";
  e.type = EventType::Hosting;
  e.match_id = "m1";
  e.payload["host_code"] = picojson::value("a1b2c3d4");
  const std::string json = SerializeEvents({e});
  EXPECT_EQ(json, R"({"events":[{"at":"2026-10-15T20:00:01Z","client_seq":7,"match_id":"m1",)"
                  R"("payload":{"host_code":"a1b2c3d4"},"type":"hosting"}]})");
}

TEST(Protocol, ParsesAcceptedThroughAndRegistration)
{
  EXPECT_EQ(ParseAcceptedThrough(R"({"accepted_through":9})").value_or(0), 9u);
  EXPECT_FALSE(ParseAcceptedThrough(R"({"nope":1})").has_value());
  const auto reg = ParseRegistration(R"({"device_id":"dev_1","device_token":"tok"})");
  ASSERT_TRUE(reg.has_value());
  EXPECT_EQ(reg->device_token, "tok");
  EXPECT_EQ(SerializeRegistration("Dave", "v3.2.0-brawllive.1", "448d1dbe"),
            R"({"build":"v3.2.0-brawllive.1","identity":"448d1dbe","nickname":"Dave","platform":"windows"})");
}
```
Register the test in `UnitTests.vcxproj` and `UICommon/BrawlLive/CMakeLists.txt` (`add_dolphin_test(ProtocolTest ProtocolTest.cpp)`).

- [ ] **Step 2: Run to verify failure**

Run: `pwsh -File .\build.ps1 -Tests`
Expected: compile error, `UICommon/BrawlLive/Protocol.h` missing.

- [ ] **Step 3: Implement the protocol module**

`Source/Core/UICommon/BrawlLive/Protocol.h`:
```cpp
// Copyright 2026 BrawlLive Contributors
// SPDX-License-Identifier: GPL-2.0-or-later

#pragma once

#include <array>
#include <optional>
#include <string>
#include <string_view>
#include <vector>

#include <picojson.h>

#include "Common/CommonTypes.h"

namespace BrawlLive
{
constexpr size_t MAX_MESSAGE_BYTES = 256 * 1024;

enum class CommandType
{
  QueueStatus,
  MatchFound,
  Host,
  Join,
  Start,
  SetBuffer,
  Cancel,
  Notice,
  Unknown
};

struct TraversalInfo
{
  std::string host;
  u16 port = 6262;
  u16 alt_port = 6226;
};

struct PlayerInfo
{
  int slot = 0;
  std::string nickname;
  std::optional<int> rating;
};

struct Command
{
  u64 seq = 0;
  std::string id;
  CommandType type = CommandType::Unknown;
  std::string match_id;
  // queue_status
  int position = 0;
  bool searching = false;
  int est_wait_s = 0;
  // match_found
  std::string mode;
  std::string role;
  std::vector<PlayerInfo> players;
  int ping_estimate_ms = 0;
  int countdown_s = 0;
  // host
  std::optional<TraversalInfo> traversal;
  int expected_players = 2;
  std::array<int, 4> pad_map{1, 2, 0, 0};
  int buffer_min = 3;
  int buffer_divisor = 8;
  std::string game_name;
  // join
  std::string host_code;
  std::string address;
  u16 port = 0;
  // start / set_buffer
  std::optional<int> buffer_frames;
  // cancel / notice
  std::string reason;
  std::string text;
  std::string level;
};

struct CommandBatch
{
  std::vector<Command> commands;
  std::string now;
};

enum class EventType
{
  Online,
  QueueJoin,
  QueueLeave,
  Accept,
  Decline,
  Hosting,
  Joined,
  PlayerJoined,
  LobbyReady,
  GameStarted,
  GameEnded,
  Left,
  Error
};

struct Event
{
  u64 client_seq = 0;
  std::string at;
  EventType type = EventType::Error;
  std::string match_id;
  picojson::object payload;
};

struct Registration
{
  std::string device_id;
  std::string device_token;
};

const char* EventTypeName(EventType type);
CommandType CommandTypeFromName(std::string_view name);

std::optional<CommandBatch> ParseCommands(std::string_view json);
std::string SerializeEvents(const std::vector<Event>& events);
std::optional<u64> ParseAcceptedThrough(std::string_view json);
std::optional<Registration> ParseRegistration(std::string_view json);
std::string SerializeRegistration(const std::string& nickname, const std::string& build,
                                  const std::string& identity);
}  // namespace BrawlLive
```

`Source/Core/UICommon/BrawlLive/Protocol.cpp`:
```cpp
// Copyright 2026 BrawlLive Contributors
// SPDX-License-Identifier: GPL-2.0-or-later

#include "UICommon/BrawlLive/Protocol.h"

#include <cmath>
#include <map>

namespace BrawlLive
{
namespace
{
// Strict accessors: every getter returns nullopt when the key is missing or mistyped.
std::optional<picojson::object> GetObject(const picojson::object& o, const char* key)
{
  const auto it = o.find(key);
  if (it == o.end() || !it->second.is<picojson::object>())
    return std::nullopt;
  return it->second.get<picojson::object>();
}
std::optional<std::string> GetString(const picojson::object& o, const char* key)
{
  const auto it = o.find(key);
  if (it == o.end() || !it->second.is<std::string>())
    return std::nullopt;
  return it->second.get<std::string>();
}
std::optional<double> GetNumber(const picojson::object& o, const char* key)
{
  const auto it = o.find(key);
  if (it == o.end() || !it->second.is<double>())
    return std::nullopt;
  return it->second.get<double>();
}
std::optional<bool> GetBool(const picojson::object& o, const char* key)
{
  const auto it = o.find(key);
  if (it == o.end() || !it->second.is<bool>())
    return std::nullopt;
  return it->second.get<bool>();
}
int ToInt(std::optional<double> v, int fallback)
{
  return v ? static_cast<int>(std::lround(*v)) : fallback;
}

bool ParseOne(const picojson::object& o, Command* cmd)
{
  const auto seq = GetNumber(o, "seq");
  const auto type = GetString(o, "type");
  if (!seq || !type || *seq < 0)
    return false;
  cmd->seq = static_cast<u64>(*seq);
  cmd->id = GetString(o, "id").value_or("");
  cmd->type = CommandTypeFromName(*type);
  const auto payload = GetObject(o, "payload").value_or(picojson::object{});
  cmd->match_id = GetString(payload, "match_id").value_or("");
  cmd->position = ToInt(GetNumber(payload, "position"), 0);
  cmd->searching = GetBool(payload, "searching").value_or(false);
  cmd->est_wait_s = ToInt(GetNumber(payload, "est_wait_s"), 0);
  cmd->mode = GetString(payload, "mode").value_or("");
  cmd->role = GetString(payload, "role").value_or("");
  if (const auto it = payload.find("players"); it != payload.end() && it->second.is<picojson::array>())
  {
    for (const auto& pv : it->second.get<picojson::array>())
    {
      if (!pv.is<picojson::object>())
        return false;
      const auto& po = pv.get<picojson::object>();
      PlayerInfo p;
      p.slot = ToInt(GetNumber(po, "slot"), 0);
      p.nickname = GetString(po, "nickname").value_or("");
      if (const auto r = GetNumber(po, "rating"))
        p.rating = ToInt(r, 0);
      cmd->players.push_back(std::move(p));
    }
  }
  cmd->ping_estimate_ms = ToInt(GetNumber(payload, "ping_estimate_ms"), 0);
  cmd->countdown_s = ToInt(GetNumber(payload, "countdown_s"), 0);
  if (const auto t = GetObject(payload, "traversal"))
  {
    TraversalInfo info;
    info.host = GetString(*t, "host").value_or("");
    info.port = static_cast<u16>(ToInt(GetNumber(*t, "port"), 6262));
    info.alt_port = static_cast<u16>(ToInt(GetNumber(*t, "alt_port"), 6226));
    cmd->traversal = info;
  }
  cmd->expected_players = ToInt(GetNumber(payload, "expected_players"), 2);
  if (const auto it = payload.find("pad_map"); it != payload.end() && it->second.is<picojson::array>())
  {
    const auto& arr = it->second.get<picojson::array>();
    for (size_t i = 0; i < arr.size() && i < cmd->pad_map.size(); ++i)
      cmd->pad_map[i] = arr[i].is<double>() ? static_cast<int>(arr[i].get<double>()) : 0;
  }
  if (const auto b = GetObject(payload, "buffer"))
  {
    cmd->buffer_min = ToInt(GetNumber(*b, "min"), 3);
    cmd->buffer_divisor = ToInt(GetNumber(*b, "divisor"), 8);
  }
  cmd->game_name = GetString(payload, "game_name").value_or("");
  cmd->host_code = GetString(payload, "host_code").value_or("");
  cmd->address = GetString(payload, "address").value_or("");
  cmd->port = static_cast<u16>(ToInt(GetNumber(payload, "port"), 0));
  if (const auto f = GetNumber(payload, "buffer_frames"))
    cmd->buffer_frames = ToInt(f, 0);
  cmd->reason = GetString(payload, "reason").value_or("");
  cmd->text = GetString(payload, "text").value_or("");
  cmd->level = GetString(payload, "level").value_or("");
  return true;
}
}  // namespace

const char* EventTypeName(EventType type)
{
  switch (type)
  {
  case EventType::Online: return "online";
  case EventType::QueueJoin: return "queue_join";
  case EventType::QueueLeave: return "queue_leave";
  case EventType::Accept: return "accept";
  case EventType::Decline: return "decline";
  case EventType::Hosting: return "hosting";
  case EventType::Joined: return "joined";
  case EventType::PlayerJoined: return "player_joined";
  case EventType::LobbyReady: return "lobby_ready";
  case EventType::GameStarted: return "game_started";
  case EventType::GameEnded: return "game_ended";
  case EventType::Left: return "left";
  case EventType::Error: return "error";
  }
  return "error";
}

CommandType CommandTypeFromName(std::string_view name)
{
  static const std::map<std::string_view, CommandType> names{
      {"queue_status", CommandType::QueueStatus}, {"match_found", CommandType::MatchFound},
      {"host", CommandType::Host},               {"join", CommandType::Join},
      {"start", CommandType::Start},             {"set_buffer", CommandType::SetBuffer},
      {"cancel", CommandType::Cancel},           {"notice", CommandType::Notice},
  };
  const auto it = names.find(name);
  return it == names.end() ? CommandType::Unknown : it->second;
}

std::optional<CommandBatch> ParseCommands(std::string_view json)
{
  if (json.size() > MAX_MESSAGE_BYTES)
    return std::nullopt;
  picojson::value root;
  const std::string err = picojson::parse(root, json.begin(), json.end());
  if (!err.empty() || !root.is<picojson::object>())
    return std::nullopt;
  const auto& obj = root.get<picojson::object>();
  const auto it = obj.find("commands");
  if (it == obj.end() || !it->second.is<picojson::array>())
    return std::nullopt;
  CommandBatch batch;
  batch.now = GetString(obj, "now").value_or("");
  for (const auto& cv : it->second.get<picojson::array>())
  {
    if (!cv.is<picojson::object>())
      return std::nullopt;
    Command cmd;
    if (!ParseOne(cv.get<picojson::object>(), &cmd))
      return std::nullopt;
    batch.commands.push_back(std::move(cmd));
  }
  return batch;
}

std::string SerializeEvents(const std::vector<Event>& events)
{
  picojson::array arr;
  for (const Event& e : events)
  {
    picojson::object o;
    o["client_seq"] = picojson::value(static_cast<double>(e.client_seq));
    o["at"] = picojson::value(e.at);
    o["type"] = picojson::value(std::string(EventTypeName(e.type)));
    if (!e.match_id.empty())
      o["match_id"] = picojson::value(e.match_id);
    o["payload"] = picojson::value(e.payload);
    arr.emplace_back(std::move(o));
  }
  picojson::object root;
  root["events"] = picojson::value(arr);
  return picojson::value(root).serialize();
}

std::optional<u64> ParseAcceptedThrough(std::string_view json)
{
  picojson::value root;
  if (!picojson::parse(root, json.begin(), json.end()).empty() || !root.is<picojson::object>())
    return std::nullopt;
  const auto v = GetNumber(root.get<picojson::object>(), "accepted_through");
  if (!v || *v < 0)
    return std::nullopt;
  return static_cast<u64>(*v);
}

std::optional<Registration> ParseRegistration(std::string_view json)
{
  picojson::value root;
  if (!picojson::parse(root, json.begin(), json.end()).empty() || !root.is<picojson::object>())
    return std::nullopt;
  const auto& o = root.get<picojson::object>();
  const auto id = GetString(o, "device_id");
  const auto token = GetString(o, "device_token");
  if (!id || !token || token->empty())
    return std::nullopt;
  return Registration{*id, *token};
}

std::string SerializeRegistration(const std::string& nickname, const std::string& build,
                                  const std::string& identity)
{
  picojson::object o;
  o["nickname"] = picojson::value(nickname);
  o["build"] = picojson::value(build);
  o["identity"] = picojson::value(identity);
  o["platform"] = picojson::value(std::string("windows"));
  return picojson::value(o).serialize();
}
}  // namespace BrawlLive
```
Note: picojson serializes object keys in sorted order and integers without a fractional part, which the test strings rely on. If `client_seq` serializes as `7.0`, wrap with `picojson::value(static_cast<int64_t>(...))` only if the in-tree picojson is built with `PICOJSON_USE_INT64` (check `Externals/picojson/picojson.h`); otherwise keep `double`, which prints `7`.

Register: `DolphinLib.props` `<ClInclude Include="UICommon\BrawlLive\Protocol.h" />` near line 593 and `<ClCompile Include="UICommon\BrawlLive\Protocol.cpp" />` near line 1294; `Source/Core/UICommon/CMakeLists.txt` add `BrawlLive/Protocol.cpp` and `BrawlLive/Protocol.h` after `NetPlayIndex.h`. Confirm `picojson` is on the include path for UICommon (NetPlayIndex.cpp already includes `<picojson.h>`).

- [ ] **Step 4: Run tests to verify they pass**

Run: `pwsh -File .\build.ps1 -Tests`
Expected: `Protocol.*` four tests PASS.

- [ ] **Step 5: Commit**

```powershell
git add Source/Core/UICommon/BrawlLive Source/Core/UICommon/CMakeLists.txt Source/Core/DolphinLib.props Source/UnitTests
git commit -m "feat(brawllive): protocol v1 parse and serialize

Co-Authored-By: Claude Fable 5.1 <noreply@anthropic.com>"
```

### Task 4: Pad combo detector

**Files:**
- Create: `Source/Core/UICommon/BrawlLive/PadCombo.h`, `Source/Core/UICommon/BrawlLive/PadCombo.cpp`
- Create: `Source/UnitTests/UICommon/BrawlLive/PadComboTest.cpp`
- Modify: `Source/Core/DolphinLib.props`, `Source/Core/UICommon/CMakeLists.txt`, `Source/UnitTests/UnitTests.vcxproj`, `Source/UnitTests/UICommon/BrawlLive/CMakeLists.txt`

**Interfaces:**
- Produces (namespace `BrawlLive`):
  - `struct ComboConfig { u16 open_mask = PAD_TRIGGER_L | PAD_TRIGGER_R | PAD_BUTTON_START; u32 open_hold_ms = 1000; u32 leave_hold_ms = 3000; };`
  - `enum class ComboAction { None, OpenMenu, Up, Down, Confirm, Back, Leave };`
  - `class ComboDetector { explicit ComboDetector(ComboConfig); void SetMenuOpen(bool); void SetInSession(bool); ComboAction Feed(u16 buttons, bool connected, u64 now_ms); }` — one `Feed` per poll per port; the controller feeds the first connected port.

- [ ] **Step 1: Write the failing tests**

`Source/UnitTests/UICommon/BrawlLive/PadComboTest.cpp`:
```cpp
// Copyright 2026 BrawlLive Contributors
// SPDX-License-Identifier: GPL-2.0-or-later

#include <gtest/gtest.h>

#include "InputCommon/GCPadStatus.h"
#include "UICommon/BrawlLive/PadCombo.h"

using namespace BrawlLive;

constexpr u16 kOpen = PAD_TRIGGER_L | PAD_TRIGGER_R | PAD_BUTTON_START;

TEST(PadCombo, HoldOpensMenuAfterThresholdOnce)
{
  ComboDetector d(ComboConfig{});
  EXPECT_EQ(d.Feed(kOpen, true, 0), ComboAction::None);
  EXPECT_EQ(d.Feed(kOpen, true, 500), ComboAction::None);
  EXPECT_EQ(d.Feed(kOpen, true, 999), ComboAction::None);
  EXPECT_EQ(d.Feed(kOpen, true, 1000), ComboAction::OpenMenu);
  EXPECT_EQ(d.Feed(kOpen, true, 1100), ComboAction::None);
  EXPECT_EQ(d.Feed(0, true, 1200), ComboAction::None);
  EXPECT_EQ(d.Feed(kOpen, true, 1300), ComboAction::None);
  EXPECT_EQ(d.Feed(kOpen, true, 2300), ComboAction::OpenMenu);
}

TEST(PadCombo, PartialComboNeverFires)
{
  ComboDetector d(ComboConfig{});
  EXPECT_EQ(d.Feed(PAD_TRIGGER_L | PAD_TRIGGER_R, true, 0), ComboAction::None);
  EXPECT_EQ(d.Feed(PAD_TRIGGER_L | PAD_TRIGGER_R, true, 5000), ComboAction::None);
}

TEST(PadCombo, DisconnectResetsHold)
{
  ComboDetector d(ComboConfig{});
  EXPECT_EQ(d.Feed(kOpen, true, 0), ComboAction::None);
  EXPECT_EQ(d.Feed(kOpen, true, 600), ComboAction::None);
  EXPECT_EQ(d.Feed(kOpen, false, 700), ComboAction::None);
  EXPECT_EQ(d.Feed(kOpen, true, 800), ComboAction::None);
  EXPECT_EQ(d.Feed(kOpen, true, 1799), ComboAction::None);
  EXPECT_EQ(d.Feed(kOpen, true, 1800), ComboAction::OpenMenu);
}

TEST(PadCombo, NavigationIsEdgeTriggeredOnlyWhenMenuOpen)
{
  ComboDetector d(ComboConfig{});
  EXPECT_EQ(d.Feed(PAD_BUTTON_A, true, 0), ComboAction::None);
  d.SetMenuOpen(true);
  EXPECT_EQ(d.Feed(PAD_BUTTON_UP, true, 10), ComboAction::Up);
  EXPECT_EQ(d.Feed(PAD_BUTTON_UP, true, 20), ComboAction::None);
  EXPECT_EQ(d.Feed(0, true, 30), ComboAction::None);
  EXPECT_EQ(d.Feed(PAD_BUTTON_DOWN, true, 40), ComboAction::Down);
  EXPECT_EQ(d.Feed(PAD_BUTTON_A, true, 50), ComboAction::Confirm);
  EXPECT_EQ(d.Feed(PAD_BUTTON_B, true, 60), ComboAction::Back);
  d.SetMenuOpen(false);
  EXPECT_EQ(d.Feed(0, true, 70), ComboAction::None);
  EXPECT_EQ(d.Feed(PAD_BUTTON_B, true, 80), ComboAction::None);
}

TEST(PadCombo, InSessionOnlyLeaveAfterLongHold)
{
  ComboDetector d(ComboConfig{});
  d.SetInSession(true);
  d.SetMenuOpen(true);
  EXPECT_EQ(d.Feed(PAD_BUTTON_A, true, 0), ComboAction::None);
  EXPECT_EQ(d.Feed(kOpen, true, 100), ComboAction::None);
  EXPECT_EQ(d.Feed(kOpen, true, 1100), ComboAction::None);
  EXPECT_EQ(d.Feed(kOpen, true, 3099), ComboAction::None);
  EXPECT_EQ(d.Feed(kOpen, true, 3100), ComboAction::Leave);
  EXPECT_EQ(d.Feed(kOpen, true, 4000), ComboAction::None);
}
```
Register in `UnitTests.vcxproj` and `UICommon/BrawlLive/CMakeLists.txt` (`add_dolphin_test(PadComboTest PadComboTest.cpp)`).

- [ ] **Step 2: Run to verify failure**

Run: `pwsh -File .\build.ps1 -Tests`
Expected: compile error, `UICommon/BrawlLive/PadCombo.h` missing.

- [ ] **Step 3: Implement the detector**

`Source/Core/UICommon/BrawlLive/PadCombo.h`:
```cpp
// Copyright 2026 BrawlLive Contributors
// SPDX-License-Identifier: GPL-2.0-or-later

#pragma once

#include "Common/CommonTypes.h"
#include "InputCommon/GCPadStatus.h"

namespace BrawlLive
{
struct ComboConfig
{
  u16 open_mask = PAD_TRIGGER_L | PAD_TRIGGER_R | PAD_BUTTON_START;
  u32 open_hold_ms = 1000;
  u32 leave_hold_ms = 3000;
};

enum class ComboAction
{
  None,
  OpenMenu,
  Up,
  Down,
  Confirm,
  Back,
  Leave
};

// Pure, single-port edge/hold detector. Feed once per poll with the port's button mask.
class ComboDetector
{
public:
  explicit ComboDetector(ComboConfig config);
  void SetMenuOpen(bool open) { m_menu_open = open; }
  void SetInSession(bool in_session) { m_in_session = in_session; }
  ComboAction Feed(u16 buttons, bool connected, u64 now_ms);

private:
  ComboConfig m_config;
  bool m_menu_open = false;
  bool m_in_session = false;
  bool m_holding = false;
  bool m_fired = false;
  u64 m_hold_started_ms = 0;
  u16 m_prev_buttons = 0;
};
}  // namespace BrawlLive
```

`Source/Core/UICommon/BrawlLive/PadCombo.cpp`:
```cpp
// Copyright 2026 BrawlLive Contributors
// SPDX-License-Identifier: GPL-2.0-or-later

#include "UICommon/BrawlLive/PadCombo.h"

namespace BrawlLive
{
ComboDetector::ComboDetector(ComboConfig config) : m_config(config)
{
}

ComboAction ComboDetector::Feed(u16 buttons, bool connected, u64 now_ms)
{
  if (!connected)
  {
    m_holding = false;
    m_fired = false;
    m_prev_buttons = 0;
    return ComboAction::None;
  }

  const bool mask_down = (buttons & m_config.open_mask) == m_config.open_mask;
  ComboAction action = ComboAction::None;

  if (mask_down)
  {
    if (!m_holding)
    {
      m_holding = true;
      m_fired = false;
      m_hold_started_ms = now_ms;
    }
    else if (!m_fired)
    {
      const u64 threshold = m_in_session ? m_config.leave_hold_ms : m_config.open_hold_ms;
      if (now_ms - m_hold_started_ms >= threshold)
      {
        m_fired = true;
        action = m_in_session ? ComboAction::Leave : ComboAction::OpenMenu;
      }
    }
  }
  else
  {
    m_holding = false;
    m_fired = false;
    if (m_menu_open && !m_in_session)
    {
      const u16 pressed = static_cast<u16>(buttons & ~m_prev_buttons);
      if (pressed & PAD_BUTTON_A)
        action = ComboAction::Confirm;
      else if (pressed & PAD_BUTTON_B)
        action = ComboAction::Back;
      else if (pressed & PAD_BUTTON_UP)
        action = ComboAction::Up;
      else if (pressed & PAD_BUTTON_DOWN)
        action = ComboAction::Down;
    }
  }

  m_prev_buttons = buttons;
  return action;
}
}  // namespace BrawlLive
```
Register in `DolphinLib.props` (`UICommon\BrawlLive\PadCombo.h/.cpp`) and `Source/Core/UICommon/CMakeLists.txt`.

- [ ] **Step 4: Run tests to verify they pass**

Run: `pwsh -File .\build.ps1 -Tests`
Expected: all five `PadCombo.*` tests PASS.

- [ ] **Step 5: Commit**

```powershell
git add Source/Core/UICommon/BrawlLive/PadCombo.* Source/Core/UICommon/CMakeLists.txt Source/Core/DolphinLib.props Source/UnitTests
git commit -m "feat(brawllive): pad combo detector

Co-Authored-By: Claude Fable 5.1 <noreply@anthropic.com>"
```

---

### Task 5: Client match state machine

**Files:**
- Create: `Source/Core/UICommon/BrawlLive/MatchState.h`, `Source/Core/UICommon/BrawlLive/MatchState.cpp`
- Create: `Source/UnitTests/UICommon/BrawlLive/MatchStateTest.cpp`
- Modify: registration files as in Task 4

**Interfaces:**
- Consumes: `Command`, `Event`, `EventType`, `TraversalInfo`, `PlayerInfo` from Task 3.
- Produces (namespace `BrawlLive`):
  - `enum class Phase { Idle, Queued, MatchFound, Preparing, Hosting, Joining, Lobby, Starting, InGame, Ended, Failed };`
  - `enum class Role { None, Host, Joiner };`
  - `enum class PendingAction { None, StopEmulation, HostSession, JoinSession, StartGame, LeaveSession };`
  - `struct LocalInput { enum class Kind { QueueJoin, QueueLeave, CancelMatch, ActionDone, ActionFailed, HostCode, PlayersUpdate, EmulationRunning, EmulationStopped, Disconnected, Tick }; Kind kind; std::string mode; PendingAction action = PendingAction::None; std::string error_code, error_detail; std::string host_code; int player_count = 0; std::vector<int> peer_pings; };`
  - `struct Status { Phase phase = Phase::Idle; Role role = Role::None; PendingAction pending = PendingAction::None; std::string match_id, mode; std::vector<PlayerInfo> players; int countdown_s = 0; bool accepted = false; int queue_position = 0; int est_wait_s = 0; bool searching = false; std::string error_code, error_detail; std::string host_code; std::optional<TraversalInfo> traversal; std::string address; u16 port = 0; std::array<int, 4> pad_map{1, 2, 0, 0}; int expected_players = 2; int buffer_min = 3; int buffer_divisor = 8; std::optional<int> buffer_frames; std::string game_name; int present_players = 0; std::vector<int> peer_pings; std::string notice; u64 phase_since_ms = 0; u64 game_started_ms = 0; };`
  - `class MatchState { std::vector<Event> OnCommand(const Command&, u64 now_ms); std::vector<Event> OnLocal(const LocalInput&, u64 now_ms); const Status& GetStatus() const; }` — events come back with `client_seq` assigned and `at` empty (the Agent stamps `at`).
  - Local timeouts: Preparing 15 s → `E_STOP_TIMEOUT`; Hosting without host code 20 s → `E_HOST_TIMEOUT`; Joining 30 s → `E_JOIN_TIMEOUT`; Starting 20 s → `E_START_TIMEOUT`.
  - The controller contract: whenever `GetStatus().pending != None` after a call, the controller performs that action and reports `ActionDone`/`ActionFailed` with the same `action`.

- [ ] **Step 1: Write the failing tests**

`Source/UnitTests/UICommon/BrawlLive/MatchStateTest.cpp`:
```cpp
// Copyright 2026 BrawlLive Contributors
// SPDX-License-Identifier: GPL-2.0-or-later

#include <gtest/gtest.h>

#include <vector>

#include "UICommon/BrawlLive/MatchState.h"

using namespace BrawlLive;

namespace
{
Command Cmd(CommandType type, const std::string& match_id, u64 seq)
{
  Command c;
  c.type = type;
  c.match_id = match_id;
  c.seq = seq;
  return c;
}
LocalInput Local(LocalInput::Kind kind)
{
  LocalInput in;
  in.kind = kind;
  return in;
}
LocalInput Done(PendingAction action)
{
  LocalInput in = Local(LocalInput::Kind::ActionDone);
  in.action = action;
  return in;
}
std::vector<EventType> Types(const std::vector<Event>& events)
{
  std::vector<EventType> out;
  for (const auto& e : events)
    out.push_back(e.type);
  return out;
}
}  // namespace

TEST(MatchState, HostHappyPath)
{
  MatchState s;
  LocalInput join = Local(LocalInput::Kind::QueueJoin);
  join.mode = "casual_singles";
  auto ev = s.OnLocal(join, 0);
  EXPECT_EQ(Types(ev), (std::vector<EventType>{EventType::QueueJoin}));
  EXPECT_EQ(s.GetStatus().phase, Phase::Queued);

  Command found = Cmd(CommandType::MatchFound, "m1", 1);
  found.role = "host";
  found.countdown_s = 5;
  found.players = {{0, "Dave", std::nullopt}, {1, "Zed", 1500}};
  ev = s.OnCommand(found, 1000);
  EXPECT_TRUE(ev.empty());
  EXPECT_EQ(s.GetStatus().phase, Phase::MatchFound);
  EXPECT_EQ(s.GetStatus().role, Role::Host);
  EXPECT_EQ(s.GetStatus().countdown_s, 5);

  ev = s.OnLocal(Local(LocalInput::Kind::Tick), 3000);
  EXPECT_TRUE(ev.empty());
  EXPECT_EQ(s.GetStatus().countdown_s, 3);
  ev = s.OnLocal(Local(LocalInput::Kind::Tick), 6000);
  EXPECT_EQ(Types(ev), (std::vector<EventType>{EventType::Accept}));
  EXPECT_TRUE(s.GetStatus().accepted);

  Command host = Cmd(CommandType::Host, "m1", 2);
  host.traversal = TraversalInfo{"t.example", 6262, 6226};
  host.expected_players = 2;
  host.game_name = "Project+ Netplay Launcher";
  ev = s.OnCommand(host, 6100);
  EXPECT_EQ(s.GetStatus().phase, Phase::Preparing);
  EXPECT_EQ(s.GetStatus().pending, PendingAction::StopEmulation);

  ev = s.OnLocal(Done(PendingAction::StopEmulation), 6500);
  EXPECT_EQ(s.GetStatus().pending, PendingAction::HostSession);
  ev = s.OnLocal(Done(PendingAction::HostSession), 7000);
  EXPECT_EQ(s.GetStatus().phase, Phase::Hosting);
  EXPECT_EQ(s.GetStatus().pending, PendingAction::None);

  LocalInput code = Local(LocalInput::Kind::HostCode);
  code.host_code = "a1b2c3d4";
  ev = s.OnLocal(code, 8000);
  ASSERT_EQ(Types(ev), (std::vector<EventType>{EventType::Hosting}));
  EXPECT_EQ(ev[0].payload.at("host_code").get<std::string>(), "a1b2c3d4");
  EXPECT_EQ(ev[0].match_id, "m1");

  LocalInput players = Local(LocalInput::Kind::PlayersUpdate);
  players.player_count = 2;
  players.peer_pings = {41};
  ev = s.OnLocal(players, 9000);
  EXPECT_EQ(Types(ev), (std::vector<EventType>{EventType::PlayerJoined, EventType::LobbyReady}));
  EXPECT_EQ(s.GetStatus().phase, Phase::Lobby);

  Command start = Cmd(CommandType::Start, "m1", 3);
  ev = s.OnCommand(start, 9500);
  EXPECT_EQ(s.GetStatus().phase, Phase::Starting);
  EXPECT_EQ(s.GetStatus().pending, PendingAction::StartGame);
  EXPECT_EQ(s.GetStatus().buffer_frames.value_or(0), 6);  // ceil(41/8)=6 >= min 3

  ev = s.OnLocal(Done(PendingAction::StartGame), 9600);
  ev = s.OnLocal(Local(LocalInput::Kind::EmulationRunning), 15000);
  EXPECT_EQ(Types(ev), (std::vector<EventType>{EventType::GameStarted}));
  EXPECT_EQ(s.GetStatus().phase, Phase::InGame);

  ev = s.OnLocal(Local(LocalInput::Kind::EmulationStopped), 315000);
  ASSERT_EQ(Types(ev), (std::vector<EventType>{EventType::GameEnded}));
  EXPECT_EQ(ev[0].payload.at("duration_s").get<double>(), 300.0);
  EXPECT_EQ(s.GetStatus().phase, Phase::Ended);
  EXPECT_EQ(s.GetStatus().pending, PendingAction::LeaveSession);
  ev = s.OnLocal(Done(PendingAction::LeaveSession), 316000);
  EXPECT_EQ(s.GetStatus().phase, Phase::Ended);
  EXPECT_EQ(s.GetStatus().pending, PendingAction::None);
}

TEST(MatchState, JoinerHappyPath)
{
  MatchState s;
  LocalInput join = Local(LocalInput::Kind::QueueJoin);
  join.mode = "casual_singles";
  s.OnLocal(join, 0);
  Command found = Cmd(CommandType::MatchFound, "m1", 1);
  found.role = "join";
  found.countdown_s = 0;
  auto ev = s.OnCommand(found, 100);
  EXPECT_EQ(Types(ev), (std::vector<EventType>{EventType::Accept}));
  Command jn = Cmd(CommandType::Join, "m1", 2);
  jn.host_code = "a1b2c3d4";
  s.OnCommand(jn, 200);
  EXPECT_EQ(s.GetStatus().pending, PendingAction::StopEmulation);
  s.OnLocal(Done(PendingAction::StopEmulation), 300);
  EXPECT_EQ(s.GetStatus().pending, PendingAction::JoinSession);
  ev = s.OnLocal(Done(PendingAction::JoinSession), 400);
  EXPECT_EQ(Types(ev), (std::vector<EventType>{EventType::Joined}));
  EXPECT_EQ(s.GetStatus().phase, Phase::Lobby);
  ev = s.OnLocal(Local(LocalInput::Kind::EmulationRunning), 5000);
  EXPECT_EQ(Types(ev), (std::vector<EventType>{EventType::GameStarted}));
  EXPECT_EQ(s.GetStatus().phase, Phase::InGame);
}

TEST(MatchState, IgnoresStaleMatchCommands)
{
  MatchState s;
  LocalInput join = Local(LocalInput::Kind::QueueJoin);
  join.mode = "casual_singles";
  s.OnLocal(join, 0);
  Command found = Cmd(CommandType::MatchFound, "m1", 1);
  found.role = "host";
  s.OnCommand(found, 100);
  Command cancel = Cmd(CommandType::Cancel, "m1", 2);
  cancel.reason = "opponent_declined";
  auto ev = s.OnCommand(cancel, 200);
  EXPECT_TRUE(ev.empty());
  EXPECT_EQ(s.GetStatus().phase, Phase::Queued);  // back to queue after a pre-session cancel

  ev = s.OnCommand(Cmd(CommandType::Start, "m1", 3), 300);
  EXPECT_TRUE(ev.empty());
  EXPECT_EQ(s.GetStatus().phase, Phase::Queued);
  EXPECT_EQ(s.GetStatus().pending, PendingAction::None);

  ev = s.OnCommand(Cmd(CommandType::Unknown, "", 4), 400);
  EXPECT_TRUE(ev.empty());
}

TEST(MatchState, GameNotFoundFailsToIdle)
{
  MatchState s;
  LocalInput join = Local(LocalInput::Kind::QueueJoin);
  join.mode = "casual_singles";
  s.OnLocal(join, 0);
  Command found = Cmd(CommandType::MatchFound, "m1", 1);
  found.role = "host";
  s.OnCommand(found, 100);
  s.OnLocal(Local(LocalInput::Kind::Tick), 6000);
  s.OnCommand(Cmd(CommandType::Host, "m1", 2), 6100);
  s.OnLocal(Done(PendingAction::StopEmulation), 6200);
  LocalInput failed = Local(LocalInput::Kind::ActionFailed);
  failed.action = PendingAction::HostSession;
  failed.error_code = "E_GAME_NOT_FOUND";
  failed.error_detail = "Project+ Netplay Launcher not in game list";
  auto ev = s.OnLocal(failed, 6300);
  ASSERT_EQ(Types(ev), (std::vector<EventType>{EventType::Error}));
  EXPECT_EQ(ev[0].payload.at("code").get<std::string>(), "E_GAME_NOT_FOUND");
  EXPECT_EQ(s.GetStatus().phase, Phase::Failed);
  EXPECT_EQ(s.GetStatus().pending, PendingAction::LeaveSession);
  s.OnLocal(Done(PendingAction::LeaveSession), 6400);
  EXPECT_EQ(s.GetStatus().phase, Phase::Failed);
  s.OnLocal(Local(LocalInput::Kind::Tick), 12000);  // failure banner clears after 5 s
  EXPECT_EQ(s.GetStatus().phase, Phase::Idle);
}

TEST(MatchState, HostCodeTimeoutFails)
{
  MatchState s;
  LocalInput join = Local(LocalInput::Kind::QueueJoin);
  join.mode = "casual_singles";
  s.OnLocal(join, 0);
  Command found = Cmd(CommandType::MatchFound, "m1", 1);
  found.role = "host";
  s.OnCommand(found, 100);
  s.OnLocal(Local(LocalInput::Kind::Tick), 6000);
  s.OnCommand(Cmd(CommandType::Host, "m1", 2), 6100);
  s.OnLocal(Done(PendingAction::StopEmulation), 6200);
  s.OnLocal(Done(PendingAction::HostSession), 6300);
  auto ev = s.OnLocal(Local(LocalInput::Kind::Tick), 26000);
  EXPECT_TRUE(ev.empty());
  ev = s.OnLocal(Local(LocalInput::Kind::Tick), 26400);
  ASSERT_EQ(Types(ev), (std::vector<EventType>{EventType::Error}));
  EXPECT_EQ(ev[0].payload.at("code").get<std::string>(), "E_HOST_TIMEOUT");
  EXPECT_EQ(s.GetStatus().pending, PendingAction::LeaveSession);
}
```
Register in `UnitTests.vcxproj` and `UICommon/BrawlLive/CMakeLists.txt`.

- [ ] **Step 2: Run to verify failure**

Run: `pwsh -File .\build.ps1 -Tests`
Expected: compile error, `UICommon/BrawlLive/MatchState.h` missing.

- [ ] **Step 3: Implement the state machine**

`Source/Core/UICommon/BrawlLive/MatchState.h`:
```cpp
// Copyright 2026 BrawlLive Contributors
// SPDX-License-Identifier: GPL-2.0-or-later

#pragma once

#include <array>
#include <optional>
#include <string>
#include <vector>

#include "Common/CommonTypes.h"
#include "UICommon/BrawlLive/Protocol.h"

namespace BrawlLive
{
enum class Phase
{
  Idle,
  Queued,
  MatchFound,
  Preparing,
  Hosting,
  Joining,
  Lobby,
  Starting,
  InGame,
  Ended,
  Failed
};

enum class Role
{
  None,
  Host,
  Joiner
};

enum class PendingAction
{
  None,
  StopEmulation,
  HostSession,
  JoinSession,
  StartGame,
  LeaveSession
};

struct LocalInput
{
  enum class Kind
  {
    QueueJoin,
    QueueLeave,
    CancelMatch,
    ActionDone,
    ActionFailed,
    HostCode,
    PlayersUpdate,
    EmulationRunning,
    EmulationStopped,
    Disconnected,
    Tick
  };
  Kind kind = Kind::Tick;
  std::string mode;
  PendingAction action = PendingAction::None;
  std::string error_code;
  std::string error_detail;
  std::string host_code;
  int player_count = 0;
  std::vector<int> peer_pings;
};

struct Status
{
  Phase phase = Phase::Idle;
  Role role = Role::None;
  PendingAction pending = PendingAction::None;
  std::string match_id;
  std::string mode;
  std::vector<PlayerInfo> players;
  int countdown_s = 0;
  bool accepted = false;
  int queue_position = 0;
  int est_wait_s = 0;
  bool searching = false;
  std::string error_code;
  std::string error_detail;
  std::string host_code;
  std::optional<TraversalInfo> traversal;
  std::string address;
  u16 port = 0;
  std::array<int, 4> pad_map{1, 2, 0, 0};
  int expected_players = 2;
  int buffer_min = 3;
  int buffer_divisor = 8;
  std::optional<int> buffer_frames;
  std::string game_name;
  int present_players = 0;
  std::vector<int> peer_pings;
  std::string notice;
  u64 phase_since_ms = 0;
  u64 game_started_ms = 0;
};

// Pure client-side state machine. Thread-agnostic; the controller serialises calls.
class MatchState
{
public:
  std::vector<Event> OnCommand(const Command& cmd, u64 now_ms);
  std::vector<Event> OnLocal(const LocalInput& in, u64 now_ms);
  const Status& GetStatus() const { return m_status; }

  static constexpr u64 PREPARING_TIMEOUT_MS = 15000;
  static constexpr u64 HOSTING_TIMEOUT_MS = 20000;
  static constexpr u64 JOINING_TIMEOUT_MS = 30000;
  static constexpr u64 STARTING_TIMEOUT_MS = 20000;
  static constexpr u64 FAILED_BANNER_MS = 5000;

private:
  Event MakeEvent(EventType type);
  void SetPhase(Phase phase, u64 now_ms);
  void Fail(const std::string& code, const std::string& detail, u64 now_ms, std::vector<Event>* out);
  void ClearMatch();
  bool InSession() const;

  Status m_status;
  u64 m_next_client_seq = 1;
  u64 m_countdown_started_ms = 0;
  int m_countdown_total_s = 0;
};
}  // namespace BrawlLive
```

`Source/Core/UICommon/BrawlLive/MatchState.cpp`:
```cpp
// Copyright 2026 BrawlLive Contributors
// SPDX-License-Identifier: GPL-2.0-or-later

#include "UICommon/BrawlLive/MatchState.h"

#include <algorithm>

namespace BrawlLive
{
Event MatchState::MakeEvent(EventType type)
{
  Event e;
  e.client_seq = m_next_client_seq++;
  e.type = type;
  e.match_id = m_status.match_id;
  return e;
}

void MatchState::SetPhase(Phase phase, u64 now_ms)
{
  m_status.phase = phase;
  m_status.phase_since_ms = now_ms;
}

bool MatchState::InSession() const
{
  switch (m_status.phase)
  {
  case Phase::Hosting:
  case Phase::Joining:
  case Phase::Lobby:
  case Phase::Starting:
  case Phase::InGame:
    return true;
  default:
    return false;
  }
}

void MatchState::ClearMatch()
{
  const std::string mode = m_status.mode;
  m_status = Status{};
  m_status.mode = mode;
}

void MatchState::Fail(const std::string& code, const std::string& detail, u64 now_ms,
                      std::vector<Event>* out)
{
  Event e = MakeEvent(EventType::Error);
  e.payload["code"] = picojson::value(code);
  e.payload["detail"] = picojson::value(detail);
  out->push_back(std::move(e));
  m_status.error_code = code;
  m_status.error_detail = detail;
  // Leave the session if one may exist; otherwise nothing to clean up.
  m_status.pending = InSession() || m_status.phase == Phase::Preparing ? PendingAction::LeaveSession :
                                                                         PendingAction::None;
  SetPhase(Phase::Failed, now_ms);
}

std::vector<Event> MatchState::OnCommand(const Command& cmd, u64 now_ms)
{
  std::vector<Event> out;
  if (cmd.type == CommandType::Unknown)
    return out;
  const bool has_match = !m_status.match_id.empty();
  const bool stale = !cmd.match_id.empty() && has_match && cmd.match_id != m_status.match_id;
  if (stale)
    return out;
  // Commands that name a match while we have none are stale too, except match_found.
  if (!cmd.match_id.empty() && !has_match && cmd.type != CommandType::MatchFound)
    return out;

  switch (cmd.type)
  {
  case CommandType::QueueStatus:
    if (m_status.phase == Phase::Queued)
    {
      m_status.queue_position = cmd.position;
      m_status.est_wait_s = cmd.est_wait_s;
      m_status.searching = cmd.searching;
    }
    break;
  case CommandType::MatchFound:
    if (m_status.phase != Phase::Queued)
      break;
    m_status.match_id = cmd.match_id;
    m_status.role = cmd.role == "host" ? Role::Host : Role::Joiner;
    m_status.players = cmd.players;
    m_status.countdown_s = cmd.countdown_s;
    m_status.accepted = false;
    m_countdown_started_ms = now_ms;
    m_countdown_total_s = cmd.countdown_s;
    SetPhase(Phase::MatchFound, now_ms);
    if (cmd.countdown_s <= 0)
    {
      m_status.accepted = true;
      out.push_back(MakeEvent(EventType::Accept));
    }
    break;
  case CommandType::Host:
    if (m_status.phase != Phase::MatchFound || m_status.role != Role::Host)
      break;
    m_status.traversal = cmd.traversal;
    m_status.expected_players = std::max(2, cmd.expected_players);
    m_status.pad_map = cmd.pad_map;
    m_status.buffer_min = cmd.buffer_min;
    m_status.buffer_divisor = std::max(1, cmd.buffer_divisor);
    m_status.game_name = cmd.game_name;
    m_status.pending = PendingAction::StopEmulation;
    SetPhase(Phase::Preparing, now_ms);
    break;
  case CommandType::Join:
    if (m_status.phase != Phase::MatchFound || m_status.role != Role::Joiner)
      break;
    m_status.traversal = cmd.traversal;
    m_status.host_code = cmd.host_code;
    m_status.address = cmd.address;
    m_status.port = cmd.port;
    m_status.pending = PendingAction::StopEmulation;
    SetPhase(Phase::Preparing, now_ms);
    break;
  case CommandType::Start:
    if (m_status.phase != Phase::Lobby || m_status.role != Role::Host)
      break;
    if (cmd.buffer_frames)
    {
      m_status.buffer_frames = cmd.buffer_frames;
    }
    else
    {
      const int worst = m_status.peer_pings.empty() ?
                            0 :
                            *std::max_element(m_status.peer_pings.begin(), m_status.peer_pings.end());
      const int computed = (worst + m_status.buffer_divisor - 1) / m_status.buffer_divisor;
      m_status.buffer_frames = std::max(m_status.buffer_min, computed);
    }
    m_status.pending = PendingAction::StartGame;
    SetPhase(Phase::Starting, now_ms);
    break;
  case CommandType::SetBuffer:
    if (cmd.buffer_frames)
      m_status.buffer_frames = cmd.buffer_frames;
    break;
  case CommandType::Cancel:
    if (m_status.phase == Phase::Queued)
    {
      ClearMatch();
      SetPhase(Phase::Idle, now_ms);
    }
    else if (m_status.phase == Phase::MatchFound)
    {
      // Pre-session cancel (opponent declined, etc.): back to the queue.
      const std::string mode = m_status.mode;
      ClearMatch();
      m_status.mode = mode;
      SetPhase(Phase::Queued, now_ms);
    }
    else if (InSession() || m_status.phase == Phase::Preparing)
    {
      m_status.error_code = "E_CANCELLED";
      m_status.error_detail = cmd.reason;
      m_status.pending = PendingAction::LeaveSession;
      SetPhase(Phase::Failed, now_ms);
    }
    break;
  case CommandType::Notice:
    m_status.notice = cmd.text;
    break;
  case CommandType::Unknown:
    break;
  }
  return out;
}

std::vector<Event> MatchState::OnLocal(const LocalInput& in, u64 now_ms)
{
  std::vector<Event> out;
  using Kind = LocalInput::Kind;
  switch (in.kind)
  {
  case Kind::QueueJoin:
    if (m_status.phase == Phase::Idle || m_status.phase == Phase::Ended ||
        (m_status.phase == Phase::Failed && m_status.pending == PendingAction::None))
    {
      ClearMatch();
      m_status.mode = in.mode;
      m_status.searching = true;
      SetPhase(Phase::Queued, now_ms);
      Event e = MakeEvent(EventType::QueueJoin);
      e.payload["mode"] = picojson::value(in.mode);
      out.push_back(std::move(e));
    }
    break;
  case Kind::QueueLeave:
    if (m_status.phase == Phase::Queued)
    {
      out.push_back(MakeEvent(EventType::QueueLeave));
      ClearMatch();
      SetPhase(Phase::Idle, now_ms);
    }
    break;
  case Kind::CancelMatch:
    if (m_status.phase == Phase::MatchFound && !m_status.accepted)
    {
      out.push_back(MakeEvent(EventType::Decline));
      ClearMatch();
      SetPhase(Phase::Idle, now_ms);
    }
    else if (InSession())
    {
      out.push_back(MakeEvent(EventType::Left));
      m_status.pending = PendingAction::LeaveSession;
      SetPhase(Phase::Ended, now_ms);
    }
    break;
  case Kind::ActionDone:
    if (in.action != m_status.pending)
      break;
    m_status.pending = PendingAction::None;
    switch (in.action)
    {
    case PendingAction::StopEmulation:
      m_status.pending = m_status.role == Role::Host ? PendingAction::HostSession :
                                                       PendingAction::JoinSession;
      break;
    case PendingAction::HostSession:
      SetPhase(Phase::Hosting, now_ms);
      break;
    case PendingAction::JoinSession:
      out.push_back(MakeEvent(EventType::Joined));
      SetPhase(Phase::Lobby, now_ms);
      break;
    case PendingAction::StartGame:
      break;  // stay in Starting until emulation runs
    case PendingAction::LeaveSession:
      break;  // phase already Ended/Failed; Tick clears Failed
    case PendingAction::None:
      break;
    }
    break;
  case Kind::ActionFailed:
    if (in.action == m_status.pending)
      Fail(in.error_code, in.error_detail, now_ms, &out);
    break;
  case Kind::HostCode:
    if (m_status.phase == Phase::Hosting && m_status.host_code.empty() && !in.host_code.empty())
    {
      m_status.host_code = in.host_code;
      Event e = MakeEvent(EventType::Hosting);
      e.payload["host_code"] = picojson::value(in.host_code);
      out.push_back(std::move(e));
    }
    break;
  case Kind::PlayersUpdate:
    if (m_status.phase == Phase::Hosting || m_status.phase == Phase::Lobby)
    {
      if (in.player_count > m_status.present_players)
      {
        Event e = MakeEvent(EventType::PlayerJoined);
        e.payload["player_count"] = picojson::value(static_cast<double>(in.player_count));
        out.push_back(std::move(e));
      }
      m_status.present_players = in.player_count;
      m_status.peer_pings = in.peer_pings;
      if (m_status.phase == Phase::Hosting && in.player_count >= m_status.expected_players)
      {
        Event e = MakeEvent(EventType::LobbyReady);
        e.payload["player_count"] = picojson::value(static_cast<double>(in.player_count));
        out.push_back(std::move(e));
        SetPhase(Phase::Lobby, now_ms);
      }
    }
    break;
  case Kind::EmulationRunning:
    if (m_status.phase == Phase::Lobby || m_status.phase == Phase::Starting)
    {
      m_status.game_started_ms = now_ms;
      out.push_back(MakeEvent(EventType::GameStarted));
      SetPhase(Phase::InGame, now_ms);
    }
    break;
  case Kind::EmulationStopped:
    if (m_status.phase == Phase::InGame)
    {
      Event e = MakeEvent(EventType::GameEnded);
      e.payload["reason"] = picojson::value(std::string("stopped"));
      e.payload["duration_s"] =
          picojson::value(static_cast<double>((now_ms - m_status.game_started_ms) / 1000));
      out.push_back(std::move(e));
      m_status.pending = PendingAction::LeaveSession;
      SetPhase(Phase::Ended, now_ms);
    }
    break;
  case Kind::Disconnected:
    if (InSession())
      Fail("E_DISCONNECT", in.error_detail, now_ms, &out);
    break;
  case Kind::Tick:
    if (m_status.phase == Phase::MatchFound && !m_status.accepted)
    {
      const int elapsed_s = static_cast<int>((now_ms - m_countdown_started_ms) / 1000);
      const int remaining = m_countdown_total_s - elapsed_s;
      m_status.countdown_s = std::max(0, remaining);
      if (remaining <= 0)
      {
        m_status.accepted = true;
        out.push_back(MakeEvent(EventType::Accept));
      }
    }
    else if (m_status.phase == Phase::Preparing && now_ms - m_status.phase_since_ms > PREPARING_TIMEOUT_MS)
    {
      Fail("E_STOP_TIMEOUT", "emulation did not stop", now_ms, &out);
    }
    else if (m_status.phase == Phase::Hosting && m_status.host_code.empty() &&
             now_ms - m_status.phase_since_ms > HOSTING_TIMEOUT_MS)
    {
      Fail("E_HOST_TIMEOUT", "no traversal host code", now_ms, &out);
    }
    else if (m_status.phase == Phase::Joining && now_ms - m_status.phase_since_ms > JOINING_TIMEOUT_MS)
    {
      Fail("E_JOIN_TIMEOUT", "could not join host", now_ms, &out);
    }
    else if (m_status.phase == Phase::Starting && now_ms - m_status.phase_since_ms > STARTING_TIMEOUT_MS)
    {
      Fail("E_START_TIMEOUT", "game did not start", now_ms, &out);
    }
    else if (m_status.phase == Phase::Failed && m_status.pending == PendingAction::None &&
             now_ms - m_status.phase_since_ms > FAILED_BANNER_MS)
    {
      ClearMatch();
      SetPhase(Phase::Idle, now_ms);
    }
    break;
  }
  return out;
}
}  // namespace BrawlLive
```
Two details the code above relies on: the header's private members must include `int m_countdown_total_s = 0;` after `m_countdown_started_ms`, and the `MatchFound` case must set `m_countdown_total_s = cmd.countdown_s;` right after `m_countdown_started_ms = now_ms;`. The test `HostHappyPath` expects `countdown_s == 3` at 3000 ms and `Accept` at 6000 ms for a 5 s countdown started at 1000 ms.

Register in `DolphinLib.props` and `Source/Core/UICommon/CMakeLists.txt`.

- [ ] **Step 4: Run tests to verify they pass**

Run: `pwsh -File .\build.ps1 -Tests`
Expected: all five `MatchState.*` tests PASS.

- [ ] **Step 5: Commit**

```powershell
git add Source/Core/UICommon/BrawlLive/MatchState.* Source/Core/UICommon/CMakeLists.txt Source/Core/DolphinLib.props Source/UnitTests
git commit -m "feat(brawllive): client match state machine

Co-Authored-By: Claude Fable 5.1 <noreply@anthropic.com>"
```

### Task 6: Transport and agent thread

**Files:**
- Create: `Source/Core/UICommon/BrawlLive/Transport.h`, `Transport.cpp`, `Agent.h`, `Agent.cpp`
- Create: `Source/UnitTests/UICommon/BrawlLive/AgentTest.cpp`
- Modify: registration files as in Task 4

**Interfaces:**
- Consumes: Task 3 protocol functions.
- Produces (namespace `BrawlLive`):
  - `struct HttpResult { bool transport_ok = false; long status = 0; std::string body; };`
  - `using Headers = std::map<std::string, std::string>; using AbortCheck = std::function<bool()>;`
  - `class ITransport { virtual HttpResult Get(const std::string& url, const Headers&, u32 timeout_ms, AbortCheck abort) = 0; virtual HttpResult Post(const std::string& url, const std::string& body, const Headers&, u32 timeout_ms) = 0; };`
  - `class CurlTransport final : public ITransport` over `Common::HttpRequest`.
  - `struct AgentConfig { std::string server_url, device_token, nickname, build, identity; u32 poll_wait_s = 25; };`
  - `struct AgentCallbacks { std::function<void(const Command&)> on_command; std::function<void(const std::string& token)> on_registered; std::function<void(bool online)> on_connectivity; std::function<std::string()> now_iso; std::function<u32()> jitter_ms; };`
  - `class Agent { Agent(std::unique_ptr<ITransport>, AgentConfig, AgentCallbacks); void Start(); void Stop(); void PushEvent(Event); bool IsOnline() const; u32 RunOnce(); }` — `RunOnce` performs one iteration (register if needed, flush events, poll once) and returns the milliseconds to sleep before the next iteration (0 after success, exponential backoff 1 s doubling to a 30 s cap after failures).
  - Callbacks fire on the agent thread; the controller marshals to Qt.

- [ ] **Step 1: Write the failing tests**

`Source/UnitTests/UICommon/BrawlLive/AgentTest.cpp`:
```cpp
// Copyright 2026 BrawlLive Contributors
// SPDX-License-Identifier: GPL-2.0-or-later

#include <gtest/gtest.h>

#include <deque>
#include <memory>
#include <string>
#include <vector>

#include "UICommon/BrawlLive/Agent.h"
#include "UICommon/BrawlLive/Transport.h"

using namespace BrawlLive;

namespace
{
struct Recorded
{
  std::string method;
  std::string url;
  std::string body;
  Headers headers;
};

class FakeTransport final : public ITransport
{
public:
  std::deque<HttpResult> script;
  std::vector<Recorded> requests;
  HttpResult Next()
  {
    if (script.empty())
      return HttpResult{false, 0, ""};
    HttpResult r = script.front();
    script.pop_front();
    return r;
  }
  HttpResult Get(const std::string& url, const Headers& headers, u32, AbortCheck) override
  {
    requests.push_back({"GET", url, "", headers});
    return Next();
  }
  HttpResult Post(const std::string& url, const std::string& body, const Headers& headers,
                  u32) override
  {
    requests.push_back({"POST", url, body, headers});
    return Next();
  }
};

struct Fixture
{
  FakeTransport* transport = nullptr;
  std::vector<Command> commands;
  std::vector<std::string> tokens;
  std::vector<bool> connectivity;
  std::unique_ptr<Agent> agent;

  explicit Fixture(std::string token)
  {
    auto t = std::make_unique<FakeTransport>();
    transport = t.get();
    AgentConfig cfg;
    cfg.server_url = "http://stub";
    cfg.device_token = std::move(token);
    cfg.nickname = "Dave";
    cfg.build = "v3.2.0-brawllive.1";
    cfg.identity = "448d1dbe";
    AgentCallbacks cb;
    cb.on_command = [this](const Command& c) { commands.push_back(c); };
    cb.on_registered = [this](const std::string& tok) { tokens.push_back(tok); };
    cb.on_connectivity = [this](bool online) { connectivity.push_back(online); };
    cb.now_iso = [] { return std::string("2026-10-15T20:00:00Z"); };
    cb.jitter_ms = [] { return 0u; };
    agent = std::make_unique<Agent>(std::move(t), cfg, cb);
  }
};
}  // namespace

TEST(Agent, RegistersWhenNoToken)
{
  Fixture f("");
  f.transport->script.push_back({true, 200, R"({"device_id":"dev_1","device_token":"tok1"})"});
  f.transport->script.push_back({true, 204, ""});
  EXPECT_EQ(f.agent->RunOnce(), 0u);
  ASSERT_EQ(f.tokens.size(), 1u);
  EXPECT_EQ(f.tokens[0], "tok1");
  ASSERT_EQ(f.transport->requests.size(), 2u);
  EXPECT_EQ(f.transport->requests[0].method, "POST");
  EXPECT_EQ(f.transport->requests[0].url, "http://stub/v1/devices");
  EXPECT_EQ(f.transport->requests[1].method, "GET");
  EXPECT_EQ(f.transport->requests[1].headers.at("Authorization"), "Bearer tok1");
}

TEST(Agent, DeliversCommandsAndAdvancesSeq)
{
  Fixture f("tok");
  f.transport->script.push_back(
      {true, 200,
       R"({"commands":[{"seq":12,"id":"a","type":"queue_status","payload":{"position":3}},)"
       R"({"seq":13,"id":"b","type":"notice","payload":{"text":"hi"}}]})"});
  f.transport->script.push_back({true, 204, ""});
  EXPECT_EQ(f.agent->RunOnce(), 0u);
  ASSERT_EQ(f.commands.size(), 2u);
  EXPECT_EQ(f.commands[0].position, 3);
  EXPECT_EQ(f.commands[1].text, "hi");
  f.agent->RunOnce();
  EXPECT_EQ(f.transport->requests[0].url, "http://stub/v1/agent/commands?after=0&wait=25");
  EXPECT_EQ(f.transport->requests[1].url, "http://stub/v1/agent/commands?after=13&wait=25");
}

TEST(Agent, FlushesEventsBeforePollAndRetriesOnFailure)
{
  Fixture f("tok");
  Event e;
  e.type = EventType::QueueJoin;
  e.client_seq = 1;
  f.agent->PushEvent(e);
  f.transport->script.push_back({true, 500, "boom"});  // POST fails
  EXPECT_GT(f.agent->RunOnce(), 0u);
  ASSERT_EQ(f.transport->requests.size(), 1u);
  EXPECT_EQ(f.transport->requests[0].method, "POST");
  EXPECT_EQ(f.transport->requests[0].url, "http://stub/v1/agent/events");
  EXPECT_NE(f.transport->requests[0].body.find("\"at\":\"2026-10-15T20:00:00Z\""), std::string::npos);

  f.transport->script.push_back({true, 200, R"({"accepted_through":1})"});
  f.transport->script.push_back({true, 204, ""});
  EXPECT_EQ(f.agent->RunOnce(), 0u);
  ASSERT_EQ(f.transport->requests.size(), 3u);
  EXPECT_EQ(f.transport->requests[1].method, "POST");
  EXPECT_EQ(f.transport->requests[2].method, "GET");

  f.transport->script.push_back({true, 204, ""});
  f.agent->RunOnce();
  EXPECT_EQ(f.transport->requests.back().method, "GET");  // nothing left to flush
}

TEST(Agent, BackoffCapsAndRecovers)
{
  Fixture f("tok");
  std::vector<u32> sleeps;
  for (int i = 0; i < 7; ++i)
    sleeps.push_back(f.agent->RunOnce());  // transport_ok=false every time
  EXPECT_EQ(sleeps, (std::vector<u32>{1000, 2000, 4000, 8000, 16000, 30000, 30000}));
  ASSERT_FALSE(f.connectivity.empty());
  EXPECT_FALSE(f.connectivity.back());
  EXPECT_FALSE(f.agent->IsOnline());
  f.transport->script.push_back({true, 204, ""});
  EXPECT_EQ(f.agent->RunOnce(), 0u);
  EXPECT_TRUE(f.connectivity.back());
  EXPECT_TRUE(f.agent->IsOnline());
}

TEST(Agent, ReregistersOnUnauthorized)
{
  Fixture f("stale");
  f.transport->script.push_back({true, 401, ""});
  EXPECT_GT(f.agent->RunOnce(), 0u);
  f.transport->script.push_back({true, 200, R"({"device_id":"dev_2","device_token":"fresh"})"});
  f.transport->script.push_back({true, 204, ""});
  EXPECT_EQ(f.agent->RunOnce(), 0u);
  EXPECT_EQ(f.tokens.back(), "fresh");
}
```
Register in `UnitTests.vcxproj` and `UICommon/BrawlLive/CMakeLists.txt`.

- [ ] **Step 2: Run to verify failure**

Run: `pwsh -File .\build.ps1 -Tests`
Expected: compile error, `UICommon/BrawlLive/Agent.h` missing.

- [ ] **Step 3: Check the HttpRequest progress-callback semantics**

Read `Source/Core/Common/HttpRequest.cpp` around the function named `CurlProgressCallback` (grep for `CURLOPT_XFERINFOFUNCTION` or `CURLOPT_PROGRESSFUNCTION`). Record whether the Dolphin-level callback returning `false` aborts the transfer (curl aborts when the C callback returns non-zero). Write the finding as a comment at the top of `Transport.cpp`. The code below assumes `false` aborts; if the opposite holds, invert the two `return` statements in `MakeProgressCallback`.

- [ ] **Step 4: Implement transport and agent**

`Source/Core/UICommon/BrawlLive/Transport.h`:
```cpp
// Copyright 2026 BrawlLive Contributors
// SPDX-License-Identifier: GPL-2.0-or-later

#pragma once

#include <functional>
#include <map>
#include <string>

#include "Common/CommonTypes.h"

namespace BrawlLive
{
struct HttpResult
{
  bool transport_ok = false;  // false: no HTTP response at all (DNS, connect, abort)
  long status = 0;
  std::string body;
};

using Headers = std::map<std::string, std::string>;
using AbortCheck = std::function<bool()>;  // return true to abort an in-flight request

class ITransport
{
public:
  virtual ~ITransport() = default;
  virtual HttpResult Get(const std::string& url, const Headers& headers, u32 timeout_ms,
                         AbortCheck abort) = 0;
  virtual HttpResult Post(const std::string& url, const std::string& body, const Headers& headers,
                          u32 timeout_ms) = 0;
};

// libcurl via Common::HttpRequest. A total deadline is enforced through the progress callback
// because HttpRequest's timeout only covers the connect phase.
class CurlTransport final : public ITransport
{
public:
  HttpResult Get(const std::string& url, const Headers& headers, u32 timeout_ms,
                 AbortCheck abort) override;
  HttpResult Post(const std::string& url, const std::string& body, const Headers& headers,
                  u32 timeout_ms) override;
};
}  // namespace BrawlLive
```

`Source/Core/UICommon/BrawlLive/Transport.cpp`:
```cpp
// Copyright 2026 BrawlLive Contributors
// SPDX-License-Identifier: GPL-2.0-or-later

#include "UICommon/BrawlLive/Transport.h"

#include <chrono>

#include "Common/HttpRequest.h"

namespace BrawlLive
{
namespace
{
Common::HttpRequest::Headers ToDolphinHeaders(const Headers& headers)
{
  Common::HttpRequest::Headers out;
  for (const auto& [k, v] : headers)
    out[k] = v;
  return out;
}

Common::HttpRequest::ProgressCallback MakeProgressCallback(u32 timeout_ms, AbortCheck abort)
{
  const auto deadline = std::chrono::steady_clock::now() + std::chrono::milliseconds(timeout_ms);
  return [deadline, abort](s64, s64, s64, s64) {
    if (std::chrono::steady_clock::now() > deadline)
      return false;  // abort: deadline passed
    if (abort && abort())
      return false;  // abort: caller asked
    return true;     // continue
  };
}

HttpResult FromResponse(Common::HttpRequest& req, Common::HttpRequest::Response response)
{
  HttpResult r;
  r.status = req.GetLastResponseCode();
  r.transport_ok = r.status > 0;
  if (response)
    r.body.assign(response->begin(), response->end());
  return r;
}
}  // namespace

HttpResult CurlTransport::Get(const std::string& url, const Headers& headers, u32 timeout_ms,
                              AbortCheck abort)
{
  Common::HttpRequest req(std::chrono::milliseconds(5000), MakeProgressCallback(timeout_ms, abort));
  auto response = req.Get(url, ToDolphinHeaders(headers), Common::HttpRequest::AllowedReturnCodes::All);
  return FromResponse(req, std::move(response));
}

HttpResult CurlTransport::Post(const std::string& url, const std::string& body,
                               const Headers& headers, u32 timeout_ms)
{
  Common::HttpRequest req(std::chrono::milliseconds(5000), MakeProgressCallback(timeout_ms, nullptr));
  Headers h = headers;
  h["Content-Type"] = "application/json";
  auto response =
      req.Post(url, std::string_view(body), ToDolphinHeaders(h), Common::HttpRequest::AllowedReturnCodes::All);
  return FromResponse(req, std::move(response));
}
}  // namespace BrawlLive
```

`Source/Core/UICommon/BrawlLive/Agent.h`:
```cpp
// Copyright 2026 BrawlLive Contributors
// SPDX-License-Identifier: GPL-2.0-or-later

#pragma once

#include <atomic>
#include <condition_variable>
#include <functional>
#include <memory>
#include <mutex>
#include <string>
#include <thread>
#include <vector>

#include "Common/CommonTypes.h"
#include "UICommon/BrawlLive/Protocol.h"
#include "UICommon/BrawlLive/Transport.h"

namespace BrawlLive
{
struct AgentConfig
{
  std::string server_url;
  std::string device_token;
  std::string nickname;
  std::string build;
  std::string identity;
  u32 poll_wait_s = 25;
};

struct AgentCallbacks
{
  std::function<void(const Command&)> on_command;
  std::function<void(const std::string& token)> on_registered;
  std::function<void(bool online)> on_connectivity;
  std::function<std::string()> now_iso;
  std::function<u32()> jitter_ms;  // added to backoff; tests return 0
};

// Background client: registration, event flush, long-poll. Callbacks run on the agent thread.
class Agent
{
public:
  Agent(std::unique_ptr<ITransport> transport, AgentConfig config, AgentCallbacks callbacks);
  ~Agent();

  void Start();
  void Stop();
  void PushEvent(Event event);
  bool IsOnline() const { return m_online.load(); }
  // One iteration; returns ms to sleep before the next one. Public for tests.
  u32 RunOnce();

  static constexpr u32 BACKOFF_INITIAL_MS = 1000;
  static constexpr u32 BACKOFF_CAP_MS = 30000;
  static constexpr u32 EVENT_TIMEOUT_MS = 10000;
  static constexpr u32 REGISTER_TIMEOUT_MS = 10000;

private:
  bool EnsureRegistered();
  bool FlushEvents();
  bool Poll();
  u32 Failure();
  void Success();
  Headers AuthHeaders() const;
  void ThreadMain();

  std::unique_ptr<ITransport> m_transport;
  AgentConfig m_config;
  AgentCallbacks m_callbacks;

  std::mutex m_mutex;
  std::condition_variable m_wake;
  std::vector<Event> m_pending_events;
  bool m_stop_requested = false;
  std::thread m_thread;

  std::atomic<bool> m_online{false};
  std::atomic<bool> m_events_waiting{false};
  u64 m_last_seq = 0;
  u32 m_backoff_ms = 0;
  int m_consecutive_failures = 0;
};
}  // namespace BrawlLive
```

`Source/Core/UICommon/BrawlLive/Agent.cpp`:
```cpp
// Copyright 2026 BrawlLive Contributors
// SPDX-License-Identifier: GPL-2.0-or-later

#include "UICommon/BrawlLive/Agent.h"

#include <algorithm>
#include <chrono>

#include "Common/Logging/Log.h"
#include "Common/Thread.h"

namespace BrawlLive
{
Agent::Agent(std::unique_ptr<ITransport> transport, AgentConfig config, AgentCallbacks callbacks)
    : m_transport(std::move(transport)), m_config(std::move(config)), m_callbacks(std::move(callbacks))
{
}

Agent::~Agent()
{
  Stop();
}

void Agent::Start()
{
  if (m_thread.joinable())
    return;
  m_stop_requested = false;
  m_thread = std::thread(&Agent::ThreadMain, this);
}

void Agent::Stop()
{
  {
    std::lock_guard lk(m_mutex);
    m_stop_requested = true;
  }
  m_wake.notify_all();
  if (m_thread.joinable())
    m_thread.join();
}

void Agent::PushEvent(Event event)
{
  {
    std::lock_guard lk(m_mutex);
    if (event.at.empty() && m_callbacks.now_iso)
      event.at = m_callbacks.now_iso();
    m_pending_events.push_back(std::move(event));
  }
  m_events_waiting = true;  // aborts an in-flight long-poll so the event goes out promptly
  m_wake.notify_all();
}

void Agent::ThreadMain()
{
  Common::SetCurrentThreadName("BrawlLive Agent");
  while (true)
  {
    const u32 sleep_ms = RunOnce();
    std::unique_lock lk(m_mutex);
    if (m_stop_requested)
      return;
    if (sleep_ms > 0)
      m_wake.wait_for(lk, std::chrono::milliseconds(sleep_ms),
                      [this] { return m_stop_requested || !m_pending_events.empty(); });
    if (m_stop_requested)
      return;
  }
}

Headers Agent::AuthHeaders() const
{
  Headers h;
  h["Authorization"] = "Bearer " + m_config.device_token;
  h["X-BrawlLive-Build"] = m_config.build;
  h["X-BrawlLive-Identity"] = m_config.identity;
  h["User-Agent"] = "BrawlLive-Dolphin/" + m_config.build;
  return h;
}

u32 Agent::Failure()
{
  ++m_consecutive_failures;
  m_backoff_ms = m_backoff_ms == 0 ? BACKOFF_INITIAL_MS : std::min(m_backoff_ms * 2, BACKOFF_CAP_MS);
  if (m_consecutive_failures == 2)
  {
    // Report offline once, on the second consecutive failure; a single blip stays silent.
    m_online = false;
    if (m_callbacks.on_connectivity)
      m_callbacks.on_connectivity(false);
  }
  const u32 jitter = m_callbacks.jitter_ms ? m_callbacks.jitter_ms() : 0;
  return m_backoff_ms + jitter;
}

void Agent::Success()
{
  m_consecutive_failures = 0;
  m_backoff_ms = 0;
  if (!m_online.exchange(true) && m_callbacks.on_connectivity)
    m_callbacks.on_connectivity(true);
}

bool Agent::EnsureRegistered()
{
  if (!m_config.device_token.empty())
    return true;
  const std::string body = SerializeRegistration(m_config.nickname, m_config.build, m_config.identity);
  Headers h;
  h["Content-Type"] = "application/json";
  h["User-Agent"] = "BrawlLive-Dolphin/" + m_config.build;
  const HttpResult r = m_transport->Post(m_config.server_url + "/v1/devices", body, h, REGISTER_TIMEOUT_MS);
  if (!r.transport_ok || r.status != 200)
  {
    WARN_LOG_FMT(NETPLAY, "BrawlLive: registration failed (status {})", r.status);
    return false;
  }
  const auto reg = ParseRegistration(r.body);
  if (!reg)
    return false;
  m_config.device_token = reg->device_token;
  if (m_callbacks.on_registered)
    m_callbacks.on_registered(reg->device_token);
  return true;
}

bool Agent::FlushEvents()
{
  std::vector<Event> batch;
  {
    std::lock_guard lk(m_mutex);
    batch = m_pending_events;
  }
  m_events_waiting = false;
  if (batch.empty())
    return true;
  const HttpResult r =
      m_transport->Post(m_config.server_url + "/v1/agent/events", SerializeEvents(batch), AuthHeaders(), EVENT_TIMEOUT_MS);
  if (!r.transport_ok || r.status != 200)
  {
    if (r.status == 401)
      m_config.device_token.clear();
    return false;
  }
  const auto accepted = ParseAcceptedThrough(r.body);
  if (!accepted)
    return false;
  std::lock_guard lk(m_mutex);
  std::erase_if(m_pending_events, [&](const Event& e) { return e.client_seq <= *accepted; });
  return true;
}

bool Agent::Poll()
{
  const std::string url = m_config.server_url + "/v1/agent/commands?after=" + std::to_string(m_last_seq) +
                          "&wait=" + std::to_string(m_config.poll_wait_s);
  const HttpResult r = m_transport->Get(url, AuthHeaders(), (m_config.poll_wait_s + 10) * 1000,
                                        [this] { return m_events_waiting.load(); });
  if (!r.transport_ok)
    return m_events_waiting.load();  // an abort for pending events is not a failure
  if (r.status == 204)
    return true;
  if (r.status == 401)
  {
    m_config.device_token.clear();
    return false;
  }
  if (r.status != 200)
    return false;
  const auto batch = ParseCommands(r.body);
  if (!batch)
  {
    WARN_LOG_FMT(NETPLAY, "BrawlLive: dropped malformed command batch ({} bytes)", r.body.size());
    return false;
  }
  for (const Command& cmd : batch->commands)
  {
    if (cmd.seq <= m_last_seq)
      continue;
    m_last_seq = cmd.seq;
    if (m_callbacks.on_command)
      m_callbacks.on_command(cmd);
  }
  return true;
}

u32 Agent::RunOnce()
{
  if (!EnsureRegistered())
    return Failure();
  if (!FlushEvents())
    return Failure();
  if (!Poll())
    return Failure();
  Success();
  return 0;
}
}  // namespace BrawlLive
```
Register the four files in `DolphinLib.props` and `Source/Core/UICommon/CMakeLists.txt`.

- [ ] **Step 5: Run tests to verify they pass**

Run: `pwsh -File .\build.ps1 -Tests`
Expected: all five `Agent.*` tests PASS. Note `BackoffCapsAndRecovers` expects `1000, 2000, 4000, 8000, 16000, 30000, 30000`.

- [ ] **Step 6: Commit**

```powershell
git add Source/Core/UICommon/BrawlLive Source/Core/UICommon/CMakeLists.txt Source/Core/DolphinLib.props Source/UnitTests
git commit -m "feat(brawllive): libcurl transport and long-poll agent thread

Co-Authored-By: Claude Fable 5.1 <noreply@anthropic.com>"
```

---

### Task 7: Stub matchmaking service (Node, zero dependencies)

**Files (repo `brawllive`):**
- Create: `tools/stub-service/package.json`, `tools/stub-service/server.ts`, `tools/stub-service/main.ts`, `tools/stub-service/server.test.ts`

**Interfaces:**
- Produces: `createStubServer(options: { mode: 'direct' | 'traversal'; traversal?: { host: string; port: number; alt_port: number }; countdownS?: number; gameName?: string }) => { listen(port: number): Promise<number>; close(): Promise<void>; state(): StubState }` where `StubState = { devices: Record<string, DeviceState>; matches: MatchRecord[]; events: EventRecord[] }`.
- HTTP: `POST /v1/devices`, `GET /v1/agent/commands?after&wait`, `POST /v1/agent/events`, `GET /debug/state`, `POST /debug/reset`.
- Pairing: two devices with `queue_join` in the same mode form a match; first to queue hosts. Sequence per match: `match_found` to both → both `accept` → `host` to host → host `hosting` → `join` to joiner (direct: `address 127.0.0.1`, `port 2626`; traversal: `host_code`) → host `lobby_ready` → `start` to host. State per match: `matched | hosting | joining | lobby | starting | in_game | ended | failed`.

- [ ] **Step 1: Write the failing test**

`tools/stub-service/package.json`:
```json
{
  "name": "brawllive-stub-service",
  "private": true,
  "type": "module",
  "engines": { "node": ">=24.12" },
  "scripts": {
    "start": "node main.ts --port 8787 --mode direct",
    "test": "node --test server.test.ts"
  }
}
```

`tools/stub-service/server.test.ts`:
```ts
import { test } from 'node:test';
import assert from 'node:assert/strict';
import { createStubServer } from './server.ts';

type Json = Record<string, unknown>;

async function register(base: string, nickname: string): Promise<string> {
  const res = await fetch(`${base}/v1/devices`, {
    method: 'POST',
    headers: { 'content-type': 'application/json' },
    body: JSON.stringify({ nickname, build: 'v3.2.0-brawllive.1', identity: '448d1dbe', platform: 'windows' }),
  });
  assert.equal(res.status, 200);
  const body = (await res.json()) as { device_token: string };
  return body.device_token;
}

async function post(base: string, token: string, events: Json[]): Promise<number> {
  const res = await fetch(`${base}/v1/agent/events`, {
    method: 'POST',
    headers: { 'content-type': 'application/json', authorization: `Bearer ${token}` },
    body: JSON.stringify({ events: events.map((e, i) => ({ client_seq: i + 1, at: 'now', ...e })) }),
  });
  assert.equal(res.status, 200);
  return ((await res.json()) as { accepted_through: number }).accepted_through;
}

async function poll(base: string, token: string, after: number): Promise<Json[]> {
  const res = await fetch(`${base}/v1/agent/commands?after=${after}&wait=1`, {
    headers: { authorization: `Bearer ${token}` },
  });
  if (res.status === 204) return [];
  assert.equal(res.status, 200);
  return ((await res.json()) as { commands: Json[] }).commands;
}

test('pairs two devices and walks the direct-mode sequence', async () => {
  const server = createStubServer({ mode: 'direct', countdownS: 0 });
  const port = await server.listen(0);
  const base = `http://127.0.0.1:${port}`;
  try {
    const a = await register(base, 'A');
    const b = await register(base, 'B');
    assert.deepEqual(await poll(base, a, 0), []);

    await post(base, a, [{ type: 'queue_join', payload: { mode: 'casual_singles' } }]);
    await post(base, b, [{ type: 'queue_join', payload: { mode: 'casual_singles' } }]);

    const foundA = await poll(base, a, 0);
    const foundB = await poll(base, b, 0);
    assert.equal(foundA[0].type, 'match_found');
    assert.equal((foundA[0].payload as Json).role, 'host');
    assert.equal((foundB[0].payload as Json).role, 'join');
    const matchId = (foundA[0].payload as Json).match_id as string;

    await post(base, a, [{ type: 'accept', match_id: matchId, payload: {} }]);
    await post(base, b, [{ type: 'accept', match_id: matchId, payload: {} }]);
    const hostCmd = await poll(base, a, foundA[0].seq as number);
    assert.equal(hostCmd[0].type, 'host');
    assert.equal((hostCmd[0].payload as Json).traversal, undefined);
    assert.equal((hostCmd[0].payload as Json).game_name, 'Project+ Netplay Launcher');

    await post(base, a, [{ type: 'hosting', match_id: matchId, payload: { host_code: 'direct' } }]);
    const joinCmd = await poll(base, b, foundB[0].seq as number);
    assert.equal(joinCmd[0].type, 'join');
    assert.equal((joinCmd[0].payload as Json).address, '127.0.0.1');
    assert.equal((joinCmd[0].payload as Json).port, 2626);

    await post(base, b, [{ type: 'joined', match_id: matchId, payload: {} }]);
    await post(base, a, [{ type: 'lobby_ready', match_id: matchId, payload: { player_count: 2 } }]);
    const startCmd = await poll(base, a, hostCmd[0].seq as number);
    assert.equal(startCmd[0].type, 'start');

    await post(base, a, [{ type: 'game_started', match_id: matchId, payload: {} }]);
    await post(base, b, [{ type: 'game_started', match_id: matchId, payload: {} }]);
    const state = server.state();
    assert.equal(state.matches[0].state, 'in_game');
    assert.ok(state.events.some((e) => e.type === 'game_started'));
  } finally {
    await server.close();
  }
});

test('rejects bad auth and duplicate client_seq', async () => {
  const server = createStubServer({ mode: 'direct' });
  const port = await server.listen(0);
  const base = `http://127.0.0.1:${port}`;
  try {
    const res = await fetch(`${base}/v1/agent/commands?after=0&wait=0`, {
      headers: { authorization: 'Bearer nope' },
    });
    assert.equal(res.status, 401);
    const a = await register(base, 'A');
    assert.equal(await post(base, a, [{ type: 'queue_join', payload: { mode: 'x' } }]), 1);
    assert.equal(await post(base, a, [{ type: 'queue_join', payload: { mode: 'x' } }]), 1);
    assert.equal(server.state().events.length, 1);
  } finally {
    await server.close();
  }
});
```

- [ ] **Step 2: Run to verify failure**

Run (in `D:\Projects\Games\BRAWLLIVE\tools\stub-service`): `node --test server.test.ts`
Expected: fails with `Cannot find module './server.ts'`.

- [ ] **Step 3: Implement the stub**

`tools/stub-service/server.ts`:
```ts
import { createServer, type IncomingMessage, type Server, type ServerResponse } from 'node:http';
import { randomBytes } from 'node:crypto';

export type Mode = 'direct' | 'traversal';
export type Traversal = { host: string; port: number; alt_port: number };
export type StubOptions = {
  mode: Mode;
  traversal?: Traversal;
  countdownS?: number;
  gameName?: string;
};

type Json = Record<string, unknown>;
type Command = { seq: number; id: string; type: string; payload: Json };
type MatchStateName = 'matched' | 'hosting' | 'joining' | 'lobby' | 'starting' | 'in_game' | 'ended' | 'failed';

export type DeviceState = {
  id: string;
  token: string;
  nickname: string;
  queueMode: string | null;
  lastSeq: number;
  commands: Command[];
  nextSeq: number;
  acceptedThrough: number;
  waiters: Array<() => void>;
};
export type MatchRecord = {
  id: string;
  mode: string;
  host: string;
  joiner: string;
  state: MatchStateName;
  accepted: Set<string>;
  started: Set<string>;
  hostCode: string | null;
};
export type EventRecord = { device: string; client_seq: number; type: string; match_id?: string; payload: Json; at: string };
export type StubState = { devices: Record<string, DeviceState>; matches: MatchRecord[]; events: EventRecord[] };

export function createStubServer(options: StubOptions) {
  const countdownS = options.countdownS ?? 2;
  const gameName = options.gameName ?? 'Project+ Netplay Launcher';
  const devices = new Map<string, DeviceState>(); // by token
  const matches: MatchRecord[] = [];
  const events: EventRecord[] = [];
  let matchCounter = 0;

  const send = (device: DeviceState, type: string, payload: Json) => {
    const cmd: Command = { seq: device.nextSeq++, id: `cmd_${device.id}_${device.nextSeq}`, type, payload };
    device.commands.push(cmd);
    const waiters = device.waiters.splice(0);
    for (const wake of waiters) wake();
  };

  const findMatch = (device: DeviceState, matchId: string | undefined) =>
    matches.find((m) => m.id === matchId && (m.host === device.id || m.joiner === device.id));

  const tryPair = () => {
    const queued = [...devices.values()].filter((d) => d.queueMode !== null);
    const byMode = new Map<string, DeviceState[]>();
    for (const d of queued) byMode.set(d.queueMode as string, [...(byMode.get(d.queueMode as string) ?? []), d]);
    for (const [mode, list] of byMode) {
      while (list.length >= 2) {
        const host = list.shift() as DeviceState;
        const joiner = list.shift() as DeviceState;
        host.queueMode = null;
        joiner.queueMode = null;
        const id = `m${++matchCounter}`;
        matches.push({ id, mode, host: host.id, joiner: joiner.id, state: 'matched', accepted: new Set(), started: new Set(), hostCode: null });
        const players = [
          { slot: 0, nickname: host.nickname },
          { slot: 1, nickname: joiner.nickname },
        ];
        send(host, 'match_found', { match_id: id, mode, role: 'host', players, ping_estimate_ms: 1, countdown_s: countdownS });
        send(joiner, 'match_found', { match_id: id, mode, role: 'join', players, ping_estimate_ms: 1, countdown_s: countdownS });
      }
    }
  };

  const applyEvent = (device: DeviceState, ev: EventRecord) => {
    const match = findMatch(device, ev.match_id);
    switch (ev.type) {
      case 'queue_join':
        device.queueMode = String(ev.payload.mode ?? 'casual_singles');
        tryPair();
        break;
      case 'queue_leave':
        device.queueMode = null;
        break;
      case 'accept':
        if (!match) break;
        match.accepted.add(device.id);
        if (match.accepted.size === 2 && match.state === 'matched') {
          match.state = 'hosting';
          const host = [...devices.values()].find((d) => d.id === match.host) as DeviceState;
          const hostPayload: Json = {
            match_id: match.id,
            expected_players: 2,
            pad_map: [1, 2, 0, 0],
            buffer: { min: 3, divisor: 8 },
            game_name: gameName,
          };
          if (options.mode === 'traversal') hostPayload.traversal = options.traversal;
          send(host, 'host', hostPayload);
        }
        break;
      case 'decline':
        if (match) match.state = 'failed';
        break;
      case 'hosting':
        if (!match || match.host !== device.id) break;
        match.hostCode = String(ev.payload.host_code ?? '');
        match.state = 'joining';
        {
          const joiner = [...devices.values()].find((d) => d.id === match.joiner) as DeviceState;
          const joinPayload: Json = { match_id: match.id };
          if (options.mode === 'traversal') {
            joinPayload.host_code = match.hostCode;
            joinPayload.traversal = options.traversal;
          } else {
            joinPayload.address = '127.0.0.1';
            joinPayload.port = 2626;
          }
          send(joiner, 'join', joinPayload);
        }
        break;
      case 'lobby_ready':
        if (!match || match.host !== device.id) break;
        match.state = 'starting';
        send(device, 'start', { match_id: match.id });
        break;
      case 'game_started':
        if (!match) break;
        match.started.add(device.id);
        if (match.started.size === 2) match.state = 'in_game';
        break;
      case 'game_ended':
      case 'left':
        if (match) match.state = 'ended';
        break;
      case 'error':
        if (match) match.state = 'failed';
        break;
      default:
        break;
    }
  };

  const readBody = (req: IncomingMessage): Promise<string> =>
    new Promise((resolve, reject) => {
      let data = '';
      req.on('data', (chunk) => { data += chunk; if (data.length > 1_000_000) req.destroy(); });
      req.on('end', () => resolve(data));
      req.on('error', reject);
    });

  const json = (res: ServerResponse, status: number, body: unknown) => {
    const text = JSON.stringify(body);
    res.writeHead(status, { 'content-type': 'application/json', 'content-length': Buffer.byteLength(text) });
    res.end(text);
  };

  const auth = (req: IncomingMessage): DeviceState | null => {
    const header = req.headers.authorization ?? '';
    const token = header.startsWith('Bearer ') ? header.slice(7) : '';
    return devices.get(token) ?? null;
  };

  const handler = async (req: IncomingMessage, res: ServerResponse) => {
    const url = new URL(req.url ?? '/', 'http://localhost');
    try {
      if (req.method === 'POST' && url.pathname === '/v1/devices') {
        const body = JSON.parse(await readBody(req)) as Json;
        const token = randomBytes(16).toString('hex');
        const id = `dev_${devices.size + 1}`;
        devices.set(token, { id, token, nickname: String(body.nickname ?? id), queueMode: null, lastSeq: 0, commands: [], nextSeq: 1, acceptedThrough: 0, waiters: [] });
        return json(res, 200, { device_id: id, device_token: token });
      }
      if (url.pathname === '/debug/state' && req.method === 'GET') {
        const state: StubState = { devices: Object.fromEntries([...devices.values()].map((d) => [d.id, d])), matches, events };
        return json(res, 200, JSON.parse(JSON.stringify(state, (_k, v) => (v instanceof Set ? [...v] : v))));
      }
      if (url.pathname === '/debug/reset' && req.method === 'POST') {
        devices.clear(); matches.length = 0; events.length = 0; matchCounter = 0;
        return json(res, 200, { ok: true });
      }
      const device = auth(req);
      if (!device) return json(res, 401, { error: 'unauthorized' });

      if (req.method === 'GET' && url.pathname === '/v1/agent/commands') {
        const after = Number(url.searchParams.get('after') ?? '0');
        const wait = Math.min(25, Number(url.searchParams.get('wait') ?? '0'));
        const pending = () => device.commands.filter((c) => c.seq > after);
        if (pending().length === 0 && wait > 0) {
          await new Promise<void>((resolve) => {
            const timer = setTimeout(resolve, wait * 1000);
            device.waiters.push(() => { clearTimeout(timer); resolve(); });
          });
        }
        const commands = pending();
        if (commands.length === 0) { res.writeHead(204); return res.end(); }
        return json(res, 200, { commands, now: new Date().toISOString() });
      }
      if (req.method === 'POST' && url.pathname === '/v1/agent/events') {
        const body = JSON.parse(await readBody(req)) as { events?: Array<Json> };
        for (const raw of body.events ?? []) {
          const seq = Number(raw.client_seq);
          if (!Number.isFinite(seq) || seq <= device.acceptedThrough) continue;
          const ev: EventRecord = { device: device.id, client_seq: seq, type: String(raw.type), match_id: raw.match_id as string | undefined, payload: (raw.payload as Json) ?? {}, at: String(raw.at ?? '') };
          events.push(ev);
          device.acceptedThrough = seq;
          applyEvent(device, ev);
        }
        return json(res, 200, { accepted_through: device.acceptedThrough });
      }
      return json(res, 404, { error: 'not found' });
    } catch (err) {
      return json(res, 400, { error: String(err) });
    }
  };

  let server: Server | null = null;
  return {
    listen(port: number): Promise<number> {
      return new Promise((resolve) => {
        server = createServer((req, res) => { void handler(req, res); });
        server.listen(port, '127.0.0.1', () => {
          const address = server?.address();
          resolve(typeof address === 'object' && address ? address.port : port);
        });
      });
    },
    close(): Promise<void> {
      return new Promise((resolve) => {
        for (const d of devices.values()) for (const wake of d.waiters.splice(0)) wake();
        server ? server.close(() => resolve()) : resolve();
      });
    },
    state(): StubState {
      return { devices: Object.fromEntries([...devices.values()].map((d) => [d.id, d])), matches, events };
    },
  };
}
```

`tools/stub-service/main.ts`:
```ts
import { createStubServer, type Mode } from './server.ts';

const args = new Map<string, string>();
for (let i = 2; i < process.argv.length; i += 2) args.set(process.argv[i], process.argv[i + 1] ?? '');
const port = Number(args.get('--port') ?? '8787');
const mode = (args.get('--mode') ?? 'direct') as Mode;
const server = createStubServer({
  mode,
  traversal: { host: args.get('--traversal-host') ?? 'stun.dolphin-emu.org', port: 6262, alt_port: 6226 },
  countdownS: Number(args.get('--countdown') ?? '3'),
});
server.listen(port).then((p) => console.log(`BrawlLive stub listening on http://127.0.0.1:${p} (${mode})`));
process.on('SIGINT', () => { void server.close().then(() => process.exit(0)); });
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `node --test server.test.ts`
Expected: `# pass 2`, `# fail 0`.

- [ ] **Step 5: Commit (repo `brawllive`)**

```powershell
git add tools/stub-service
git commit -m "feat(tools): protocol v1 stub matchmaking service for local tests

Co-Authored-By: Claude Fable 5.1 <noreply@anthropic.com>"
```

### Task 8: In-game overlay

**Files:**
- Create: `Source/Core/VideoCommon/BrawlLiveUI.h`, `Source/Core/VideoCommon/BrawlLiveUI.cpp`
- Modify: `Source/Core/VideoCommon/OnScreenUI.cpp:323-324` (add two lines after the golf UI call), `Source/Core/VideoCommon/CMakeLists.txt` (after `NetPlayGolfUI.h`), `Source/Core/DolphinLib.props` (near lines 758 and 1433)
- Create: `Source/UnitTests/VideoCommon/BrawlLiveUITest.cpp`; modify `Source/UnitTests/UnitTests.vcxproj`, `Source/UnitTests/VideoCommon/CMakeLists.txt`

**Interfaces:**
- Produces (namespace `BrawlLive`):
  - `struct OverlaySnapshot { bool visible = false; bool offline = false; std::string headline; std::vector<std::string> lines; std::vector<std::string> menu_items; int selected = -1; std::string error; std::string hint; };`
  - `std::vector<std::string> FormatOverlayLines(const OverlaySnapshot&);` pure, tested.
  - `class BrawlLiveUI { void SetSnapshot(OverlaySnapshot); void Display(); }` and `extern std::unique_ptr<BrawlLiveUI> g_brawllive_ui;` (created once by the controller, never reset per session).

- [ ] **Step 1: Write the failing test**

`Source/UnitTests/VideoCommon/BrawlLiveUITest.cpp`:
```cpp
// Copyright 2026 BrawlLive Contributors
// SPDX-License-Identifier: GPL-2.0-or-later

#include <gtest/gtest.h>

#include <string>
#include <vector>

#include "VideoCommon/BrawlLiveUI.h"

using namespace BrawlLive;

TEST(BrawlLiveUI, HiddenSnapshotFormatsNothing)
{
  OverlaySnapshot s;
  EXPECT_TRUE(FormatOverlayLines(s).empty());
}

TEST(BrawlLiveUI, FormatsHeadlineLinesMenuErrorHint)
{
  OverlaySnapshot s;
  s.visible = true;
  s.headline = "BrawlLive: Searching 0:42";
  s.lines = {"Region EU-West", "Max ping 120 ms"};
  s.menu_items = {"Casual singles", "Ranked singles", "Casual doubles"};
  s.selected = 1;
  s.error = "E_JOIN_TIMEOUT: could not join host";
  s.hint = "A select  B back  L+R+Start close";
  EXPECT_EQ(FormatOverlayLines(s),
            (std::vector<std::string>{"BrawlLive: Searching 0:42", "Region EU-West", "Max ping 120 ms",
                                      "  Casual singles", "> Ranked singles", "  Casual doubles",
                                      "E_JOIN_TIMEOUT: could not join host",
                                      "A select  B back  L+R+Start close"}));
}

TEST(BrawlLiveUI, OfflinePrefixesHeadline)
{
  OverlaySnapshot s;
  s.visible = true;
  s.offline = true;
  s.headline = "BrawlLive: Idle";
  EXPECT_EQ(FormatOverlayLines(s)[0], "BrawlLive: offline, retrying");
}
```
Register: `UnitTests.vcxproj` (`<ClCompile Include="VideoCommon\BrawlLiveUITest.cpp" />`) and `Source/UnitTests/VideoCommon/CMakeLists.txt` (`add_dolphin_test(BrawlLiveUITest BrawlLiveUITest.cpp)`).

- [ ] **Step 2: Run to verify failure**

Run: `pwsh -File .\build.ps1 -Tests`
Expected: compile error, `VideoCommon/BrawlLiveUI.h` missing.

- [ ] **Step 3: Implement the overlay**

`Source/Core/VideoCommon/BrawlLiveUI.h`:
```cpp
// Copyright 2026 BrawlLive Contributors
// SPDX-License-Identifier: GPL-2.0-or-later

#pragma once

#include <memory>
#include <mutex>
#include <string>
#include <vector>

namespace BrawlLive
{
struct OverlaySnapshot
{
  bool visible = false;
  bool offline = false;
  std::string headline;
  std::vector<std::string> lines;
  std::vector<std::string> menu_items;
  int selected = -1;
  std::string error;
  std::string hint;
};

// Pure projection used by Display() and by tests.
std::vector<std::string> FormatOverlayLines(const OverlaySnapshot& snapshot);

// ImGui overlay drawn from OnScreenUI::DrawDebugText on the video thread.
class BrawlLiveUI
{
public:
  void SetSnapshot(OverlaySnapshot snapshot);
  void Display();

private:
  std::mutex m_mutex;
  OverlaySnapshot m_snapshot;
};

extern std::unique_ptr<BrawlLiveUI> g_brawllive_ui;
}  // namespace BrawlLive
```

`Source/Core/VideoCommon/BrawlLiveUI.cpp`:
```cpp
// Copyright 2026 BrawlLive Contributors
// SPDX-License-Identifier: GPL-2.0-or-later

#include "VideoCommon/BrawlLiveUI.h"

#include <imgui.h>

#include "Core/Config/BrawlLiveSettings.h"

namespace BrawlLive
{
std::unique_ptr<BrawlLiveUI> g_brawllive_ui;

std::vector<std::string> FormatOverlayLines(const OverlaySnapshot& s)
{
  std::vector<std::string> out;
  if (!s.visible)
    return out;
  out.push_back(s.offline ? "BrawlLive: offline, retrying" : s.headline);
  for (const auto& line : s.lines)
    out.push_back(line);
  for (size_t i = 0; i < s.menu_items.size(); ++i)
    out.push_back((static_cast<int>(i) == s.selected ? "> " : "  ") + s.menu_items[i]);
  if (!s.error.empty())
    out.push_back(s.error);
  if (!s.hint.empty())
    out.push_back(s.hint);
  return out;
}

void BrawlLiveUI::SetSnapshot(OverlaySnapshot snapshot)
{
  std::lock_guard lk(m_mutex);
  m_snapshot = std::move(snapshot);
}

void BrawlLiveUI::Display()
{
  OverlaySnapshot s;
  {
    std::lock_guard lk(m_mutex);
    s = m_snapshot;
  }
  if (!s.visible)
    return;

  const float scale = ImGui::GetIO().DisplayFramebufferScale.x * Config::Get(Config::BRAWLLIVE_OVERLAY_SCALE);
  const float margin = 10.0f * scale;
  ImGui::SetNextWindowPos(ImVec2(ImGui::GetIO().DisplaySize.x - margin, margin), ImGuiCond_Always,
                          ImVec2(1.0f, 0.0f));
  ImGui::SetNextWindowBgAlpha(0.75f);
  const ImGuiWindowFlags flags = ImGuiWindowFlags_NoResize | ImGuiWindowFlags_NoMove |
                                 ImGuiWindowFlags_NoCollapse | ImGuiWindowFlags_AlwaysAutoResize |
                                 ImGuiWindowFlags_NoFocusOnAppearing | ImGuiWindowFlags_NoNav;
  if (!ImGui::Begin("BrawlLive", nullptr, flags))
  {
    ImGui::End();
    return;
  }
  ImGui::SetWindowFontScale(scale);
  const ImVec4 cyan(0.3f, 0.9f, 1.0f, 1.0f);
  const ImVec4 yellow(1.0f, 0.9f, 0.3f, 1.0f);
  const ImVec4 red(1.0f, 0.4f, 0.4f, 1.0f);
  const ImVec4 grey(0.7f, 0.7f, 0.7f, 1.0f);

  ImGui::TextColored(s.offline ? red : cyan, "%s", s.offline ? "BrawlLive: offline, retrying" : s.headline.c_str());
  for (const auto& line : s.lines)
    ImGui::TextUnformatted(line.c_str());
  for (size_t i = 0; i < s.menu_items.size(); ++i)
  {
    const bool sel = static_cast<int>(i) == s.selected;
    ImGui::TextColored(sel ? yellow : grey, "%s%s", sel ? "> " : "  ", s.menu_items[i].c_str());
  }
  if (!s.error.empty())
    ImGui::TextColored(red, "%s", s.error.c_str());
  if (!s.hint.empty())
    ImGui::TextColored(grey, "%s", s.hint.c_str());
  ImGui::End();
}
}  // namespace BrawlLive
```

Hook in `Source/Core/VideoCommon/OnScreenUI.cpp` directly after the golf UI block (lines 323–324):
```cpp
  if (Config::Get(Config::BRAWLLIVE_ENABLED) && BrawlLive::g_brawllive_ui)
    BrawlLive::g_brawllive_ui->Display();
```
with `#include "Core/Config/BrawlLiveSettings.h"` and `#include "VideoCommon/BrawlLiveUI.h"` added to the include block. Register the two new files in `Source/Core/VideoCommon/CMakeLists.txt` and `DolphinLib.props`.

- [ ] **Step 4: Run tests to verify they pass**

Run: `pwsh -File .\build.ps1 -Tests`
Expected: three `BrawlLiveUI.*` tests PASS; full build succeeds (the hook compiles).

- [ ] **Step 5: Commit**

```powershell
git add Source/Core/VideoCommon/BrawlLiveUI.* Source/Core/VideoCommon/OnScreenUI.cpp Source/Core/VideoCommon/CMakeLists.txt Source/Core/DolphinLib.props Source/UnitTests
git commit -m "feat(brawllive): ImGui overlay drawn with the netplay overlays

Co-Authored-By: Claude Fable 5.1 <noreply@anthropic.com>"
```

---

### Task 9: Qt controller, menu and MainWindow wiring

**Files:**
- Create: `Source/Core/DolphinQt/BrawlLive/BrawlLiveController.h`, `Source/Core/DolphinQt/BrawlLive/BrawlLiveController.cpp`
- Modify: `Source/Core/DolphinQt/MainWindow.h` (add `friend class BrawlLiveController;` inside the class, a forward declaration `class BrawlLiveController;`, and member `BrawlLiveController* m_brawllive = nullptr;` next to `m_netplay_dialog`), `Source/Core/DolphinQt/MainWindow.cpp` (construct after `NetPlayInit();` at line 272; connect menu signals in `ConnectMenuBar`), `Source/Core/DolphinQt/MenuBar.h/.cpp` (add `AddBrawlLiveMenu()` called after `AddToolsMenu()` at line 92), `Source/Core/DolphinQt/DolphinQt.vcxproj` (ClCompile + QtMoc entries), `Source/Core/DolphinQt/CMakeLists.txt`

**Interfaces:**
- Consumes: `MatchState`, `ComboDetector`, `Agent`, `CurlTransport`, `OverlaySnapshot`, `g_brawllive_ui`, `MainWindow::ForceStop/NetPlayHost/NetPlayJoin/NetPlayQuit`, `Settings::EmulationStateChanged`, `NetPlayServer::SetPadMapping/AdjustMinimumPadBufferSize/RequestStartGame`, `NetPlayClient::GetPlayers/IsConnected/DoAllPlayersHaveGame/IsLocalPlayer`, `Common::g_TraversalClient`.
- Produces: `class BrawlLiveController : public QObject` with public slots `void SetEnabled(bool)`, `void EditNickname()`, `void OpenLogFolder()`, `void ShowAbout()`; MenuBar signals `BrawlLiveEnable(bool)`, `BrawlLiveNickname()`, `BrawlLiveOpenLog()`, `BrawlLiveAbout()`.
- Queue modes: menu items `Casual singles`, `Ranked singles`, `Casual doubles`, `Ranked doubles`, `Cancel / Leave` mapping to `casual_singles`, `ranked_singles`, `casual_doubles`, `ranked_doubles`.
- Pad map convention: `pad_map[port] = slot + 1` (0 = unmapped); slot order = players sorted by pid (host pid 1 = slot 0).

- [ ] **Step 1: Add the menu**

`MenuBar.h` signals section: add
```cpp
  void BrawlLiveEnable(bool enabled);
  void BrawlLiveNickname();
  void BrawlLiveOpenLog();
  void BrawlLiveAbout();
```
and private `void AddBrawlLiveMenu();` plus `QAction* m_brawllive_enable_action = nullptr;`. In `MenuBar.cpp` constructor call `AddBrawlLiveMenu();` after `AddToolsMenu();`, and implement:
```cpp
void MenuBar::AddBrawlLiveMenu()
{
  QMenu* menu = addMenu(tr("&BrawlLive"));
  m_brawllive_enable_action = menu->addAction(tr("&Enable BrawlLive"));
  m_brawllive_enable_action->setCheckable(true);
  m_brawllive_enable_action->setChecked(Config::Get(Config::BRAWLLIVE_ENABLED));
  connect(m_brawllive_enable_action, &QAction::toggled, this, &MenuBar::BrawlLiveEnable);
  menu->addAction(tr("Set &Nickname..."), this, &MenuBar::BrawlLiveNickname);
  menu->addAction(tr("Open &Log Folder"), this, &MenuBar::BrawlLiveOpenLog);
  menu->addSeparator();
  menu->addAction(tr("&About BrawlLive"), this, &MenuBar::BrawlLiveAbout);
}
```
with `#include "Core/Config/BrawlLiveSettings.h"`.

- [ ] **Step 2: Write the controller header**

`Source/Core/DolphinQt/BrawlLive/BrawlLiveController.h`:
```cpp
// Copyright 2026 BrawlLive Contributors
// SPDX-License-Identifier: GPL-2.0-or-later

#pragma once

#include <memory>
#include <string>
#include <vector>

#include <QObject>

#include "Common/CommonTypes.h"
#include "Core/Core.h"
#include "UICommon/BrawlLive/Agent.h"
#include "UICommon/BrawlLive/MatchState.h"
#include "UICommon/BrawlLive/PadCombo.h"

class GameList;
class MainWindow;
class QTimer;

// Qt-thread orchestrator: owns the agent, state machine and combo detector, drives
// MainWindow's netplay entry points, and publishes the overlay snapshot.
class BrawlLiveController final : public QObject
{
  Q_OBJECT
public:
  BrawlLiveController(MainWindow* main_window, GameList* game_list, QObject* parent);
  ~BrawlLiveController() override;

public slots:
  void SetEnabled(bool enabled);
  void EditNickname();
  void OpenLogFolder();
  void ShowAbout();

private:
  void StartAgent();
  void StopAgent();
  void OnCommand(const BrawlLive::Command& cmd);
  void OnConnectivity(bool online);
  void OnEmulationStateChanged(Core::State state);
  void OnTick();
  void OnPadPoll();
  void HandleCombo(BrawlLive::ComboAction action);
  void ApplyLocal(const BrawlLive::LocalInput& in);
  void Dispatch(const std::vector<BrawlLive::Event>& events);
  void RunPendingAction();
  void ActionDone(BrawlLive::PendingAction action);
  void ActionFailed(BrawlLive::PendingAction action, const std::string& code, const std::string& detail);
  void PollSession();
  void PublishSnapshot();
  void QueueJoin(const std::string& mode);
  void SaveNetplayConfig();
  void RestoreNetplayConfig();
  static u64 NowMs();

  MainWindow* m_main;
  GameList* m_game_list;
  BrawlLive::MatchState m_state;
  BrawlLive::ComboDetector m_combo;
  std::unique_ptr<BrawlLive::Agent> m_agent;
  QTimer* m_tick_timer;
  QTimer* m_pad_timer;

  bool m_menu_open = false;
  int m_menu_index = 0;
  bool m_action_in_flight = false;
  bool m_awaiting_stop = false;
  bool m_online = false;
  bool m_auto_queued = false;
  int m_last_player_count = 0;

  struct SavedNetplayConfig
  {
    std::string traversal_choice, traversal_server, host_code, address;
    u16 traversal_port = 0, traversal_port_alt = 0, host_port = 0, connect_port = 0;
    bool saved = false;
  } m_saved;
};
```

- [ ] **Step 3: Write the controller implementation**

`Source/Core/DolphinQt/BrawlLive/BrawlLiveController.cpp`:
```cpp
// Copyright 2026 BrawlLive Contributors
// SPDX-License-Identifier: GPL-2.0-or-later

#include "DolphinQt/BrawlLive/BrawlLiveController.h"

#include <algorithm>
#include <chrono>

#include <QDateTime>
#include <QDesktopServices>
#include <QInputDialog>
#include <QMessageBox>
#include <QRandomGenerator>
#include <QTimer>
#include <QUrl>

#include "Common/Config/Config.h"
#include "Common/FileUtil.h"
#include "Common/Logging/Log.h"
#include "Common/StringUtil.h"
#include "Common/TraversalClient.h"
#include "Common/Version.h"
#include "Core/Config/BrawlLiveSettings.h"
#include "Core/Config/MainSettings.h"
#include "Core/Config/NetplaySettings.h"
#include "Core/HW/GCPad.h"
#include "Core/HW/SI/SI_Device.h"
#include "Core/NetPlayClient.h"
#include "Core/NetPlayServer.h"
#include "DolphinQt/GameList/GameList.h"
#include "DolphinQt/GameList/GameListModel.h"
#include "DolphinQt/MainWindow.h"
#include "DolphinQt/NetPlay/NetPlayDialog.h"
#include "DolphinQt/Settings.h"
#include "InputCommon/GCAdapter.h"
#include "UICommon/BrawlLive/Transport.h"
#include "UICommon/GameFile.h"
#include "VideoCommon/BrawlLiveUI.h"

using namespace BrawlLive;

namespace
{
const std::vector<std::pair<std::string, std::string>> kMenu = {
    {"Casual singles", "casual_singles"},
    {"Ranked singles", "ranked_singles"},
    {"Casual doubles", "casual_doubles"},
    {"Ranked doubles", "ranked_doubles"},
    {"Cancel / Leave", ""},
};
constexpr int kCancelIndex = 4;

std::string PhaseHeadline(const Status& s, u64 now_ms)
{
  const auto secs = [&] { return (now_ms - s.phase_since_ms) / 1000; };
  switch (s.phase)
  {
  case Phase::Idle: return "BrawlLive: ready";
  case Phase::Queued: return fmt::format("BrawlLive: searching {}:{:02d}", secs() / 60, secs() % 60);
  case Phase::MatchFound: return fmt::format("BrawlLive: match found, starting in {}s", s.countdown_s);
  case Phase::Preparing: return "BrawlLive: preparing session";
  case Phase::Hosting: return "BrawlLive: hosting, waiting for opponent";
  case Phase::Joining: return "BrawlLive: joining";
  case Phase::Lobby: return "BrawlLive: lobby ready";
  case Phase::Starting: return "BrawlLive: starting game";
  case Phase::InGame: return "BrawlLive: in game";
  case Phase::Ended: return "BrawlLive: match over";
  case Phase::Failed: return "BrawlLive: failed";
  }
  return "BrawlLive";
}
}  // namespace

BrawlLiveController::BrawlLiveController(MainWindow* main_window, GameList* game_list, QObject* parent)
    : QObject(parent), m_main(main_window), m_game_list(game_list),
      m_combo(ComboConfig{PAD_TRIGGER_L | PAD_TRIGGER_R | PAD_BUTTON_START,
                          Config::Get(Config::BRAWLLIVE_OPEN_HOLD_MS), 3000})
{
  if (!g_brawllive_ui)
    g_brawllive_ui = std::make_unique<BrawlLiveUI>();

  m_tick_timer = new QTimer(this);
  connect(m_tick_timer, &QTimer::timeout, this, &BrawlLiveController::OnTick);
  m_tick_timer->start(250);
  m_pad_timer = new QTimer(this);
  connect(m_pad_timer, &QTimer::timeout, this, &BrawlLiveController::OnPadPoll);
  m_pad_timer->start(16);

  connect(&Settings::Instance(), &Settings::EmulationStateChanged, this,
          &BrawlLiveController::OnEmulationStateChanged);

  if (Config::Get(Config::BRAWLLIVE_ENABLED))
    StartAgent();
  PublishSnapshot();
}

BrawlLiveController::~BrawlLiveController()
{
  StopAgent();
}

u64 BrawlLiveController::NowMs()
{
  return static_cast<u64>(std::chrono::duration_cast<std::chrono::milliseconds>(
                              std::chrono::steady_clock::now().time_since_epoch())
                              .count());
}

void BrawlLiveController::StartAgent()
{
  if (m_agent)
    return;
  const std::string url = Config::Get(Config::BRAWLLIVE_SERVER_URL);
  if (url.empty())
  {
    m_main->statusBar()->showMessage(tr("BrawlLive: no server configured (set [BrawlLive] ServerURL)"));
    return;
  }
  AgentConfig cfg;
  cfg.server_url = url;
  cfg.device_token = Config::Get(Config::BRAWLLIVE_DEVICE_TOKEN);
  cfg.nickname = Config::Get(Config::NETPLAY_NICKNAME);
  cfg.build = Common::GetScmDescStr();
  cfg.identity = Common::GetScmRevGitStr();
  AgentCallbacks cb;
  cb.on_command = [this](const Command& cmd) {
    QMetaObject::invokeMethod(this, [this, cmd] { OnCommand(cmd); }, Qt::QueuedConnection);
  };
  cb.on_registered = [this](const std::string& token) {
    QMetaObject::invokeMethod(this, [token] {
      Config::SetBase(Config::BRAWLLIVE_DEVICE_TOKEN, token);
      Config::Save();
    }, Qt::QueuedConnection);
  };
  cb.on_connectivity = [this](bool online) {
    QMetaObject::invokeMethod(this, [this, online] { OnConnectivity(online); }, Qt::QueuedConnection);
  };
  cb.now_iso = [] { return QDateTime::currentDateTimeUtc().toString(Qt::ISODate).toStdString(); };
  cb.jitter_ms = [] { return static_cast<u32>(QRandomGenerator::global()->bounded(500)); };
  m_agent = std::make_unique<Agent>(std::make_unique<CurlTransport>(), cfg, cb);
  m_agent->Start();
  INFO_LOG_FMT(NETPLAY, "BrawlLive: agent started for {}", url);
}

void BrawlLiveController::StopAgent()
{
  if (!m_agent)
    return;
  m_agent->Stop();
  m_agent.reset();
}

void BrawlLiveController::SetEnabled(bool enabled)
{
  Config::SetBase(Config::BRAWLLIVE_ENABLED, enabled);
  Config::Save();
  if (enabled)
    StartAgent();
  else
    StopAgent();
  PublishSnapshot();
}

void BrawlLiveController::EditNickname()
{
  bool ok = false;
  const QString name = QInputDialog::getText(
      m_main, tr("BrawlLive nickname"), tr("Shown to opponents:"), QLineEdit::Normal,
      QString::fromStdString(Config::Get(Config::NETPLAY_NICKNAME)), &ok);
  if (ok && !name.trimmed().isEmpty())
  {
    Config::SetBase(Config::NETPLAY_NICKNAME, name.trimmed().toStdString());
    Config::Save();
  }
}

void BrawlLiveController::OpenLogFolder()
{
  QDesktopServices::openUrl(QUrl::fromLocalFile(QString::fromStdString(File::GetUserPath(D_LOGS_IDX))));
}

void BrawlLiveController::ShowAbout()
{
  QMessageBox::information(m_main, tr("About BrawlLive"),
                           tr("BrawlLive in-game matchmaking.\nBuild %1\nNetplay identity %2")
                               .arg(QString::fromStdString(Common::GetScmDescStr()))
                               .arg(QString::fromStdString(Common::GetScmRevGitStr()).left(8)));
}

void BrawlLiveController::OnConnectivity(bool online)
{
  m_online = online;
  const std::string auto_mode = Config::Get(Config::BRAWLLIVE_AUTO_QUEUE);
  if (online && !m_auto_queued && !auto_mode.empty())
  {
    m_auto_queued = true;
    QueueJoin(auto_mode);
  }
  PublishSnapshot();
}

void BrawlLiveController::OnCommand(const Command& cmd)
{
  Dispatch(m_state.OnCommand(cmd, NowMs()));
  RunPendingAction();
  PublishSnapshot();
}

void BrawlLiveController::ApplyLocal(const LocalInput& in)
{
  Dispatch(m_state.OnLocal(in, NowMs()));
  RunPendingAction();
  PublishSnapshot();
}

void BrawlLiveController::Dispatch(const std::vector<Event>& events)
{
  if (!m_agent)
    return;
  for (const Event& e : events)
    m_agent->PushEvent(e);
}

void BrawlLiveController::QueueJoin(const std::string& mode)
{
  LocalInput in;
  in.kind = LocalInput::Kind::QueueJoin;
  in.mode = mode;
  ApplyLocal(in);
}

void BrawlLiveController::OnTick()
{
  LocalInput tick;
  tick.kind = LocalInput::Kind::Tick;
  Dispatch(m_state.OnLocal(tick, NowMs()));
  PollSession();
  RunPendingAction();
  PublishSnapshot();
}

void BrawlLiveController::OnPadPoll()
{
  if (!Config::Get(Config::BRAWLLIVE_ENABLED) || !Pad::IsInitialized())
    return;
  const Status& st = m_state.GetStatus();
  m_combo.SetMenuOpen(m_menu_open);
  m_combo.SetInSession(st.phase == Phase::InGame || st.phase == Phase::Starting || st.phase == Phase::Lobby);
  for (int port = 0; port < 4; ++port)
  {
    const auto device = Config::Get(Config::GetInfoForSIDevice(port));
    if (device == SerialInterface::SIDEVICE_NONE)
      continue;
    GCPadStatus status;
    bool connected = false;
    if (device == SerialInterface::SIDEVICE_WIIU_ADAPTER)
    {
      connected = GCAdapter::DeviceConnected(port);
      status = connected ? GCAdapter::Input(port) : GCPadStatus{};
    }
    else
    {
      status = Pad::GetStatus(port);
      connected = status.isConnected;
    }
    const ComboAction action = m_combo.Feed(status.button, connected, NowMs());
    if (action != ComboAction::None)
    {
      HandleCombo(action);
      return;  // one action per poll
    }
    if (connected)
      return;  // only the first connected port drives the menu
  }
}

void BrawlLiveController::HandleCombo(ComboAction action)
{
  const Phase phase = m_state.GetStatus().phase;
  switch (action)
  {
  case ComboAction::OpenMenu:
    m_menu_open = !m_menu_open;
    break;
  case ComboAction::Leave:
  {
    LocalInput in;
    in.kind = LocalInput::Kind::CancelMatch;
    ApplyLocal(in);
    break;
  }
  case ComboAction::Up:
    m_menu_index = (m_menu_index + static_cast<int>(kMenu.size()) - 1) % static_cast<int>(kMenu.size());
    break;
  case ComboAction::Down:
    m_menu_index = (m_menu_index + 1) % static_cast<int>(kMenu.size());
    break;
  case ComboAction::Back:
    m_menu_open = false;
    break;
  case ComboAction::Confirm:
    if (m_menu_index == kCancelIndex)
    {
      LocalInput in;
      in.kind = phase == Phase::Queued ? LocalInput::Kind::QueueLeave : LocalInput::Kind::CancelMatch;
      ApplyLocal(in);
    }
    else if (phase == Phase::Idle || phase == Phase::Ended || phase == Phase::Failed)
    {
      QueueJoin(kMenu[m_menu_index].second);
      m_menu_open = false;
    }
    break;
  case ComboAction::None:
    break;
  }
  PublishSnapshot();
}

void BrawlLiveController::OnEmulationStateChanged(Core::State state)
{
  if (state == Core::State::Uninitialized)
  {
    if (m_awaiting_stop)
    {
      m_awaiting_stop = false;
      ActionDone(PendingAction::StopEmulation);
      return;
    }
    LocalInput in;
    in.kind = LocalInput::Kind::EmulationStopped;
    ApplyLocal(in);
  }
  else if (state == Core::State::Running && !m_awaiting_stop)
  {
    LocalInput in;
    in.kind = LocalInput::Kind::EmulationRunning;
    ApplyLocal(in);
  }
}

void BrawlLiveController::ActionDone(PendingAction action)
{
  m_action_in_flight = false;
  LocalInput in;
  in.kind = LocalInput::Kind::ActionDone;
  in.action = action;
  ApplyLocal(in);
}

void BrawlLiveController::ActionFailed(PendingAction action, const std::string& code, const std::string& detail)
{
  m_action_in_flight = false;
  ERROR_LOG_FMT(NETPLAY, "BrawlLive: {} failed: {} ({})", static_cast<int>(action), code, detail);
  LocalInput in;
  in.kind = LocalInput::Kind::ActionFailed;
  in.action = action;
  in.error_code = code;
  in.error_detail = detail;
  ApplyLocal(in);
}

void BrawlLiveController::SaveNetplayConfig()
{
  if (m_saved.saved)
    return;
  m_saved.traversal_choice = Config::Get(Config::NETPLAY_TRAVERSAL_CHOICE);
  m_saved.traversal_server = Config::Get(Config::NETPLAY_TRAVERSAL_SERVER);
  m_saved.traversal_port = Config::Get(Config::NETPLAY_TRAVERSAL_PORT);
  m_saved.traversal_port_alt = Config::Get(Config::NETPLAY_TRAVERSAL_PORT_ALT);
  m_saved.host_code = Config::Get(Config::NETPLAY_HOST_CODE);
  m_saved.address = Config::Get(Config::NETPLAY_ADDRESS);
  m_saved.host_port = Config::Get(Config::NETPLAY_HOST_PORT);
  m_saved.connect_port = Config::Get(Config::NETPLAY_CONNECT_PORT);
  m_saved.saved = true;
}

void BrawlLiveController::RestoreNetplayConfig()
{
  if (!m_saved.saved)
    return;
  Config::SetCurrent(Config::NETPLAY_TRAVERSAL_CHOICE, m_saved.traversal_choice);
  Config::SetCurrent(Config::NETPLAY_TRAVERSAL_SERVER, m_saved.traversal_server);
  Config::SetCurrent(Config::NETPLAY_TRAVERSAL_PORT, m_saved.traversal_port);
  Config::SetCurrent(Config::NETPLAY_TRAVERSAL_PORT_ALT, m_saved.traversal_port_alt);
  Config::SetCurrent(Config::NETPLAY_HOST_CODE, m_saved.host_code);
  Config::SetCurrent(Config::NETPLAY_ADDRESS, m_saved.address);
  Config::SetCurrent(Config::NETPLAY_HOST_PORT, m_saved.host_port);
  Config::SetCurrent(Config::NETPLAY_CONNECT_PORT, m_saved.connect_port);
  m_saved.saved = false;
}

void BrawlLiveController::RunPendingAction()
{
  if (m_action_in_flight)
    return;
  const Status& st = m_state.GetStatus();
  switch (st.pending)
  {
  case PendingAction::None:
    return;
  case PendingAction::StopEmulation:
    m_action_in_flight = true;
    if (Core::IsUninitialized(m_main->m_system))
    {
      ActionDone(PendingAction::StopEmulation);
      return;
    }
    m_awaiting_stop = true;
    m_main->ForceStop();
    return;
  case PendingAction::HostSession:
  {
    m_action_in_flight = true;
    std::shared_ptr<const UICommon::GameFile> game;
    const GameListModel& model = m_game_list->GetGameListModel();
    for (int i = 0; i < model.rowCount(QModelIndex()); ++i)
    {
      const auto file = model.GetGameFile(i);
      const std::string path = file->GetFilePath();
      const std::string wanted = st.game_name.empty() ? "Project+ Netplay Launcher" : st.game_name;
      if (path.find(wanted) != std::string::npos)
        game = file;
    }
    if (!game)
    {
      ActionFailed(PendingAction::HostSession, "E_GAME_NOT_FOUND", "launcher not in game list");
      return;
    }
    SaveNetplayConfig();
    if (st.traversal)
    {
      Config::SetCurrent(Config::NETPLAY_TRAVERSAL_CHOICE, std::string("traversal"));
      Config::SetCurrent(Config::NETPLAY_TRAVERSAL_SERVER, st.traversal->host);
      Config::SetCurrent(Config::NETPLAY_TRAVERSAL_PORT, st.traversal->port);
      Config::SetCurrent(Config::NETPLAY_TRAVERSAL_PORT_ALT, st.traversal->alt_port);
    }
    else
    {
      Config::SetCurrent(Config::NETPLAY_TRAVERSAL_CHOICE, std::string("direct"));
      Config::SetCurrent(Config::NETPLAY_HOST_PORT, static_cast<u16>(2626));
    }
    const bool direct = !st.traversal.has_value();
    if (!m_main->NetPlayHost(*game))
    {
      RestoreNetplayConfig();
      ActionFailed(PendingAction::HostSession, "E_HOST_FAILED", "NetPlayHost returned false");
      return;
    }
    m_main->m_netplay_dialog->hide();
    ActionDone(PendingAction::HostSession);
    if (direct)
    {
      LocalInput in;
      in.kind = LocalInput::Kind::HostCode;
      in.host_code = "direct";
      ApplyLocal(in);
    }
    return;
  }
  case PendingAction::JoinSession:
  {
    m_action_in_flight = true;
    SaveNetplayConfig();
    if (st.traversal)
    {
      Config::SetCurrent(Config::NETPLAY_TRAVERSAL_CHOICE, std::string("traversal"));
      Config::SetCurrent(Config::NETPLAY_TRAVERSAL_SERVER, st.traversal->host);
      Config::SetCurrent(Config::NETPLAY_TRAVERSAL_PORT, st.traversal->port);
      Config::SetCurrent(Config::NETPLAY_HOST_CODE, st.host_code);
    }
    else
    {
      Config::SetCurrent(Config::NETPLAY_TRAVERSAL_CHOICE, std::string("direct"));
      Config::SetCurrent(Config::NETPLAY_ADDRESS, st.address);
      Config::SetCurrent(Config::NETPLAY_CONNECT_PORT, st.port);
    }
    if (!m_main->NetPlayJoin())
    {
      RestoreNetplayConfig();
      ActionFailed(PendingAction::JoinSession, "E_JOIN_REJECTED", "NetPlayJoin returned false");
      return;
    }
    m_main->m_netplay_dialog->hide();
    ActionDone(PendingAction::JoinSession);
    return;
  }
  case PendingAction::StartGame:
  {
    m_action_in_flight = true;
    auto server = Settings::Instance().GetNetPlayServer();
    auto client = Settings::Instance().GetNetPlayClient();
    if (!server || !client)
    {
      ActionFailed(PendingAction::StartGame, "E_START_FAILED", "no netplay server");
      return;
    }
    if (!client->DoAllPlayersHaveGame())
    {
      ActionFailed(PendingAction::StartGame, "E_GAME_NOT_FOUND", "a player lacks the launcher");
      return;
    }
    std::vector<const NetPlay::Player*> players = client->GetPlayers();
    std::sort(players.begin(), players.end(),
              [](const NetPlay::Player* a, const NetPlay::Player* b) { return a->pid < b->pid; });
    NetPlay::PadMappingArray mapping{};
    for (size_t port = 0; port < mapping.size(); ++port)
    {
      const int slot = st.pad_map[port] - 1;
      mapping[port] = (slot >= 0 && slot < static_cast<int>(players.size())) ? players[slot]->pid : 0;
    }
    server->SetPadMapping(mapping);
    server->AdjustMinimumPadBufferSize(static_cast<unsigned int>(st.buffer_frames.value_or(st.buffer_min)));
    if (!server->RequestStartGame())
    {
      ActionFailed(PendingAction::StartGame, "E_START_FAILED", "RequestStartGame returned false");
      return;
    }
    ActionDone(PendingAction::StartGame);
    return;
  }
  case PendingAction::LeaveSession:
    m_action_in_flight = true;
    m_main->NetPlayQuit();
    RestoreNetplayConfig();
    m_last_player_count = 0;
    ActionDone(PendingAction::LeaveSession);
    return;
  }
}

void BrawlLiveController::PollSession()
{
  const Status& st = m_state.GetStatus();
  const bool in_session = st.phase == Phase::Hosting || st.phase == Phase::Joining ||
                          st.phase == Phase::Lobby || st.phase == Phase::Starting ||
                          st.phase == Phase::InGame;
  if (!in_session)
    return;
  auto client = Settings::Instance().GetNetPlayClient();
  if (!client || !client->IsConnected())
  {
    LocalInput in;
    in.kind = LocalInput::Kind::Disconnected;
    in.error_detail = "netplay connection lost";
    Dispatch(m_state.OnLocal(in, NowMs()));
    return;
  }
  if (st.phase == Phase::Hosting && st.host_code.empty() && st.traversal && Common::g_TraversalClient &&
      Common::g_TraversalClient->GetState() == Common::TraversalClient::State::Connected)
  {
    const auto id = Common::g_TraversalClient->GetHostID();
    const std::string code(id.begin(), id.end());
    if (!code.empty() && code[0] != '\0')
    {
      LocalInput in;
      in.kind = LocalInput::Kind::HostCode;
      in.host_code = code.c_str();
      Dispatch(m_state.OnLocal(in, NowMs()));
    }
  }
  if (st.phase == Phase::Hosting || st.phase == Phase::Lobby)
  {
    const auto players = client->GetPlayers();
    LocalInput in;
    in.kind = LocalInput::Kind::PlayersUpdate;
    in.player_count = static_cast<int>(players.size());
    for (const auto* p : players)
      if (!client->IsLocalPlayer(p->pid))
        in.peer_pings.push_back(static_cast<int>(p->ping));
    if (in.player_count != m_last_player_count || st.phase == Phase::Hosting)
    {
      m_last_player_count = in.player_count;
      Dispatch(m_state.OnLocal(in, NowMs()));
    }
  }
}

void BrawlLiveController::PublishSnapshot()
{
  const Status& st = m_state.GetStatus();
  const u64 now = NowMs();
  OverlaySnapshot s;
  s.offline = m_agent && !m_online;
  s.visible = m_menu_open || st.phase != Phase::Idle || s.offline;
  s.headline = PhaseHeadline(st, now);
  if (st.phase == Phase::Queued)
    s.lines.push_back(fmt::format("Region {}  max ping {} ms", Config::Get(Config::BRAWLLIVE_REGION),
                                  Config::Get(Config::BRAWLLIVE_MAX_PING_MS)));
  if (st.phase == Phase::MatchFound || st.phase == Phase::Lobby || st.phase == Phase::InGame)
    for (const auto& p : st.players)
      s.lines.push_back(fmt::format("P{} {}", p.slot + 1, p.nickname));
  if (!st.peer_pings.empty())
    s.lines.push_back(fmt::format("Ping {} ms  buffer {}", *std::max_element(st.peer_pings.begin(), st.peer_pings.end()),
                                  st.buffer_frames.value_or(st.buffer_min)));
  if (!st.notice.empty())
    s.lines.push_back(st.notice);
  if (m_menu_open)
  {
    for (const auto& item : kMenu)
      s.menu_items.push_back(item.first);
    s.selected = m_menu_index;
    s.hint = "A select  B back  L+R+Start close";
  }
  else if (st.phase == Phase::Idle)
  {
    s.hint = "Hold L+R+Start to find a match";
  }
  else if (st.phase == Phase::InGame)
  {
    s.hint = "Hold L+R+Start 3s to leave";
  }
  if (!st.error_code.empty())
    s.error = st.error_code + ": " + st.error_detail;
  g_brawllive_ui->SetSnapshot(s);
  const auto lines = FormatOverlayLines(s);
  m_main->statusBar()->showMessage(QString::fromStdString(lines.empty() ? "BrawlLive: ready" : lines[0]));
}
```
Add `#include <fmt/format.h>` with the Common includes. `Config::NETPLAY_TRAVERSAL_PORT_ALT`, `NETPLAY_HOST_PORT`, `NETPLAY_CONNECT_PORT` are `Info<u16>`; confirm exact names in `NetplaySettings.h` and adjust if the fork spells them differently.

- [ ] **Step 4: Wire MainWindow and register the files**

`MainWindow.h`: after `class NetPlaySetupDialog;` add `class BrawlLiveController;`; inside the class add `friend class BrawlLiveController;` after the `Q_OBJECT` macro, and `BrawlLiveController* m_brawllive = nullptr;` next to `m_netplay_dialog`. `MainWindow.cpp`: `#include "DolphinQt/BrawlLive/BrawlLiveController.h"`; after `NetPlayInit();` (line 272) add `m_brawllive = new BrawlLiveController(this, m_game_list, this);`. In `ConnectMenuBar` add:
```cpp
  connect(m_menu_bar, &MenuBar::BrawlLiveEnable, this, [this](bool on) { m_brawllive->SetEnabled(on); });
  connect(m_menu_bar, &MenuBar::BrawlLiveNickname, this, [this] { m_brawllive->EditNickname(); });
  connect(m_menu_bar, &MenuBar::BrawlLiveOpenLog, this, [this] { m_brawllive->OpenLogFolder(); });
  connect(m_menu_bar, &MenuBar::BrawlLiveAbout, this, [this] { m_brawllive->ShowAbout(); });
```
`DolphinQt.vcxproj`: `<ClCompile Include="BrawlLive\BrawlLiveController.cpp" />` next to line 196 and `<QtMoc Include="BrawlLive\BrawlLiveController.h" />` next to line 429. `DolphinQt/CMakeLists.txt`: add both files next to `NetPlay/NetPlayDialog.cpp`.

- [ ] **Step 5: Build and smoke-test with the stub**

Run: `pwsh -File .\build.ps1 -Tests`. Then in `D:\Projects\Games\BRAWLLIVE\tools\stub-service` run `node main.ts --port 8787 --mode direct --countdown 3`. Copy `Binary\x64\Dolphin.exe` over `D:\Projects\Games\BRAWLLIVE\work\pplus-baseline\Dolphin.exe`, add to that folder's `user\Config\Dolphin.ini`:
```ini
[BrawlLive]
ServerURL = http://127.0.0.1:8787
```
Launch Dolphin. Expected: the status bar reads `BrawlLive: ready`, `GET /debug/state` on the stub lists one device, and the Dolphin.ini gains a `DeviceToken`. Open the BrawlLive menu: Enable toggles, Set Nickname works. Start the Offline Launcher (it will stop at the ISO prompt if no ISO is configured; that is fine for this step), hold L+R+Start on a pad for one second: the overlay opens with the five menu items; D-pad moves the marker; A on "Casual singles" shows `BrawlLive: searching 0:0x` and the stub records a `queue_join` event. Select Cancel / Leave: `queue_leave` recorded, overlay returns to ready. Stop the stub: within 10 s the overlay shows `BrawlLive: offline, retrying`; restart it: the overlay recovers.

- [ ] **Step 6: Commit**

```powershell
git add Source/Core/DolphinQt
git commit -m "feat(brawllive): Qt controller, menu and MainWindow wiring

Co-Authored-By: Claude Fable 5.1 <noreply@anthropic.com>"
```

### Task 10: Netplay identity override

**Files:**
- Modify: `Source/Core/Common/make_scmrev.h.js` (where `%DOLPHIN_DISTRIBUTOR%` is read, lines ~103–123 emit the defines), `CMake/ScmRevGen.cmake`
- Modify: `BRAWLLIVE.md` (document the env var)

**Interfaces:**
- Produces: when the environment variable `BRAWLLIVE_NETPLAY_IDENTITY` holds a 40-hex string, the generated `scmrev.h` defines `SCM_REV_STR` to it; `SCM_DESC_STR` stays the git describe of our tag. `build.ps1` (Task 1) already sets the variable.

- [ ] **Step 1: Patch the Windows generator**

In `make_scmrev.h.js`, next to the line that reads `DOLPHIN_DISTRIBUTOR` from the environment (search for `DOLPHIN_DISTRIBUTOR`), read `BRAWLLIVE_NETPLAY_IDENTITY` the same way (the script uses a `WScript.Shell` and `ExpandEnvironmentStrings` or `Environment("Process")`; mirror whichever it uses). Then, before the defines are emitted:
```js
var identity = /* value read above */;
if (/^[0-9a-f]{40}$/.test(identity)) {
  // BrawlLive: frontend-only fork. Advertise the upstream P+ commit so netplay accepts us.
  revision = identity;
}
```
where `revision` is the variable the script writes into `SCM_REV_STR` (named `rev`, `revision` or similar; keep the script's name).

- [ ] **Step 2: Patch the CMake generator**

In `CMake/ScmRevGen.cmake`, after the git `rev-parse HEAD` result is stored (the variable that becomes `DOLPHIN_WC_REVISION`), add:
```cmake
if(DEFINED ENV{BRAWLLIVE_NETPLAY_IDENTITY} AND "$ENV{BRAWLLIVE_NETPLAY_IDENTITY}" MATCHES "^[0-9a-f]+$")
  set(DOLPHIN_WC_REVISION "$ENV{BRAWLLIVE_NETPLAY_IDENTITY}")
endif()
```

- [ ] **Step 3: Rebuild and verify the embedded strings**

Run: `pwsh -File .\build.ps1` then:
```powershell
$text = [Text.Encoding]::ASCII.GetString([IO.File]::ReadAllBytes('Binary\x64\Dolphin.exe'))
$text.Contains('448d1dbee83429f6a64f269a3511dc9a5fb11a98')
$text -match 'Project\+ Dolphin/v3\.2\.0[^\s"]*'
$Matches[0]
```
Expected: `True`, `True`, and a user-agent such as `Project+ Dolphin/v3.2.0-12-g…` (or `v3.2.0-brawllive.1` once tagged). Also run the build once with the variable cleared (`$env:BRAWLLIVE_NETPLAY_IDENTITY=''` in a temporary shell) and confirm the exe then embeds the fork's real HEAD hash instead, to prove the override is opt-in.

- [ ] **Step 4: Document and commit**

Add to `BRAWLLIVE.md` under Rules: "`BRAWLLIVE_NETPLAY_IDENTITY` must equal the upstream P+ tag commit the branch is based on; bump it whenever the fork rebases onto a new P+ release."
```powershell
git add Source/Core/Common/make_scmrev.h.js CMake/ScmRevGen.cmake BRAWLLIVE.md
git commit -m "build(brawllive): netplay identity override from BRAWLLIVE_NETPLAY_IDENTITY

Co-Authored-By: Claude Fable 5.1 <noreply@anthropic.com>"
```

---

### Task 11: Updater redirect and quiet failure

**Files:**
- Modify: `Source/Core/DolphinQt/MainWindow.cpp` (`ShowUpdateDialog` ~1431 and `CheckForUpdatesAuto` ~1467)

**Interfaces:**
- Produces: both update checks call `https://api.github.com/repos/NoobyGains/brawllive-dolphin/releases/latest`; the automatic check never shows a message box on failure; release assets must contain "Windows" in the name (existing filter in `UpdateDialog::accept`) and the zip must contain only `Dolphin.exe` (the fork's `update.bat` xcopies the zip contents over the install folder).

- [ ] **Step 1: Change the URL and silence the auto-check**

Introduce one constant at the top of `MainWindow.cpp` after the includes:
```cpp
namespace
{
constexpr const char* BRAWLLIVE_RELEASES_URL =
    "https://api.github.com/repos/NoobyGains/brawllive-dolphin/releases/latest";
}
```
Replace the literal P+ URL in both functions with `BRAWLLIVE_RELEASES_URL`. In `CheckForUpdatesAuto`, replace the `QMessageBox::critical(...)` on request failure with `WARN_LOG_FMT(NETPLAY, "BrawlLive: update check failed");` followed by `return;`. In `ShowUpdateDialog`, when the HTTP status is 404 show `QMessageBox::information(this, tr("Check for Updates"), tr("No BrawlLive release published yet."))` and return.

- [ ] **Step 2: Build and verify**

Run: `pwsh -File .\build.ps1`, launch the work copy offline (disable the network adapter or set `ServerURL` to an unreachable host): no message box appears at startup. Online: Help → Check for Updates shows "No BrawlLive release published yet." until Task 14 publishes one.

- [ ] **Step 3: Commit**

```powershell
git add Source/Core/DolphinQt/MainWindow.cpp
git commit -m "feat(brawllive): point updater at BrawlLive releases, quiet auto-check

Co-Authored-By: Claude Fable 5.1 <noreply@anthropic.com>"
```

---

### Task 12: Installer scripts (repo `brawllive`)

**Files:**
- Create: `tools/installer/Install-BrawlLive.ps1`, `tools/installer/Uninstall-BrawlLive.ps1`, `tools/installer/Install-BrawlLive.Tests.ps1`

**Interfaces:**
- `Install-BrawlLive.ps1 -PPlusDir <dir> -Exe <BrawlLive Dolphin.exe> [-Build <tag>] [-Force]`: validates the folder, backs up the stock exe to `Dolphin.exe.stock` on first install, copies the new exe, writes `brawllive-install.json`. Exit 0 on success, 1 on validation failure.
- `Uninstall-BrawlLive.ps1 -PPlusDir <dir>`: restores `Dolphin.exe.stock`, removes the marker.
- Known stock hash for P+ 3.2.0: `8206123EAB4962FC5CE995C82D47F74854AFEFD779AA4F65710296AE77E3A0BB`.

- [ ] **Step 1: Write the failing test script**

`tools/installer/Install-BrawlLive.Tests.ps1`:
```powershell
#Requires -Version 7
$ErrorActionPreference = 'Stop'
$here = $PSScriptRoot
$tmp = Join-Path ([IO.Path]::GetTempPath()) ("brawllive-installer-test-" + [guid]::NewGuid())
New-Item -ItemType Directory -Path "$tmp\pplus\user\Launcher" -Force | Out-Null
New-Item -ItemType File -Path "$tmp\pplus\portable.txt" | Out-Null
[IO.File]::WriteAllBytes("$tmp\pplus\Dolphin.exe", [byte[]](1..64))
[IO.File]::WriteAllBytes("$tmp\new.exe", [byte[]](65..128))
$stockHash = (Get-FileHash "$tmp\pplus\Dolphin.exe").Hash
$newHash = (Get-FileHash "$tmp\new.exe").Hash
$failures = 0
function Assert($cond, $msg) { if (-not $cond) { Write-Host "FAIL: $msg" -ForegroundColor Red; $script:failures++ } else { Write-Host "ok: $msg" } }

# Unknown stock hash without -Force must refuse.
& "$here\Install-BrawlLive.ps1" -PPlusDir "$tmp\pplus" -Exe "$tmp\new.exe" -Build 'v0-test' *> $null
Assert ($LASTEXITCODE -eq 1) 'refuses unknown stock exe without -Force'

& "$here\Install-BrawlLive.ps1" -PPlusDir "$tmp\pplus" -Exe "$tmp\new.exe" -Build 'v0-test' -Force *> $null
Assert ($LASTEXITCODE -eq 0) 'installs with -Force'
Assert ((Get-FileHash "$tmp\pplus\Dolphin.exe").Hash -eq $newHash) 'new exe in place'
Assert ((Get-FileHash "$tmp\pplus\Dolphin.exe.stock").Hash -eq $stockHash) 'stock backed up'
$marker = Get-Content "$tmp\pplus\brawllive-install.json" | ConvertFrom-Json
Assert ($marker.build -eq 'v0-test' -and $marker.stock_hash -eq $stockHash) 'marker written'

# Upgrade keeps the original backup.
[IO.File]::WriteAllBytes("$tmp\new2.exe", [byte[]](129..192))
& "$here\Install-BrawlLive.ps1" -PPlusDir "$tmp\pplus" -Exe "$tmp\new2.exe" -Build 'v0-test2' *> $null
Assert ($LASTEXITCODE -eq 0) 'upgrade succeeds without -Force'
Assert ((Get-FileHash "$tmp\pplus\Dolphin.exe.stock").Hash -eq $stockHash) 'backup untouched on upgrade'

& "$here\Uninstall-BrawlLive.ps1" -PPlusDir "$tmp\pplus" *> $null
Assert ($LASTEXITCODE -eq 0) 'uninstall succeeds'
Assert ((Get-FileHash "$tmp\pplus\Dolphin.exe").Hash -eq $stockHash) 'stock restored'
Assert (-not (Test-Path "$tmp\pplus\brawllive-install.json")) 'marker removed'

Remove-Item $tmp -Recurse -Force
if ($failures -gt 0) { exit 1 } else { Write-Host 'installer tests passed'; exit 0 }
```

- [ ] **Step 2: Run to verify failure**

Run: `pwsh -File tools\installer\Install-BrawlLive.Tests.ps1`
Expected: errors that `Install-BrawlLive.ps1` does not exist, exit code 1.

- [ ] **Step 3: Write the installer and uninstaller**

`tools/installer/Install-BrawlLive.ps1`:
```powershell
#Requires -Version 7
<#
.SYNOPSIS  Install the BrawlLive Dolphin.exe into an existing Project+ 3.2 netplay folder.
#>
param(
  [Parameter(Mandatory)] [string] $PPlusDir,
  [Parameter(Mandatory)] [string] $Exe,
  [string] $Build = 'unknown',
  [switch] $Force
)
$ErrorActionPreference = 'Stop'
$knownStockHashes = @(
  '8206123EAB4962FC5CE995C82D47F74854AFEFD779AA4F65710296AE77E3A0BB'  # Project+ v3.2.0 Netplay Windows
)
$target = Join-Path $PPlusDir 'Dolphin.exe'
$backup = Join-Path $PPlusDir 'Dolphin.exe.stock'
$marker = Join-Path $PPlusDir 'brawllive-install.json'

foreach ($required in @($target, (Join-Path $PPlusDir 'portable.txt'), (Join-Path $PPlusDir 'user\Launcher'))) {
  if (-not (Test-Path $required)) { Write-Host "Not a Project+ netplay folder: missing $required" -ForegroundColor Red; exit 1 }
}
if (-not (Test-Path $Exe)) { Write-Host "BrawlLive exe not found: $Exe" -ForegroundColor Red; exit 1 }

$currentHash = (Get-FileHash $target -Algorithm SHA256).Hash
$alreadyInstalled = Test-Path $marker
if (-not $alreadyInstalled) {
  if (($knownStockHashes -notcontains $currentHash) -and -not $Force) {
    Write-Host "Dolphin.exe hash $currentHash is not a known Project+ release. Re-run with -Force to install anyway." -ForegroundColor Red
    exit 1
  }
  if ((Test-Path $backup) -and -not $Force) {
    Write-Host "Backup $backup already exists. Re-run with -Force to overwrite it." -ForegroundColor Red
    exit 1
  }
  Copy-Item $target $backup -Force
  $stockHash = $currentHash
} else {
  $stockHash = (Get-Content $marker | ConvertFrom-Json).stock_hash
}

Copy-Item $Exe $target -Force
$installed = @{
  build        = $Build
  installed_at = (Get-Date).ToString('o')
  exe_hash     = (Get-FileHash $target -Algorithm SHA256).Hash
  stock_hash   = $stockHash
}
$installed | ConvertTo-Json | Set-Content $marker -Encoding utf8
Write-Host "BrawlLive $Build installed into $PPlusDir (stock exe kept as Dolphin.exe.stock)"
exit 0
```

`tools/installer/Uninstall-BrawlLive.ps1`:
```powershell
#Requires -Version 7
param([Parameter(Mandatory)] [string] $PPlusDir)
$ErrorActionPreference = 'Stop'
$target = Join-Path $PPlusDir 'Dolphin.exe'
$backup = Join-Path $PPlusDir 'Dolphin.exe.stock'
$marker = Join-Path $PPlusDir 'brawllive-install.json'
if (-not (Test-Path $backup)) { Write-Host "No backup at $backup; nothing to restore." -ForegroundColor Red; exit 1 }
Copy-Item $backup $target -Force
Remove-Item $backup -Force
if (Test-Path $marker) { Remove-Item $marker -Force }
Write-Host "Stock Dolphin.exe restored in $PPlusDir"
exit 0
```

- [ ] **Step 4: Run the tests**

Run: `pwsh -File tools\installer\Install-BrawlLive.Tests.ps1`
Expected: every line `ok:`, then `installer tests passed`.

- [ ] **Step 5: Commit (repo `brawllive`)**

```powershell
git add tools/installer
git commit -m "feat(tools): BrawlLive installer and uninstaller scripts with tests

Co-Authored-By: Claude Fable 5.1 <noreply@anthropic.com>"
```

---

### Task 13: Milestone harness: two instances auto-host, auto-join, auto-start

**Files (repo `brawllive`):**
- Create: `tools/milestone/Prepare-Assets.ps1`, `tools/milestone/Run-TwoInstance.ps1`
- Modify: `.gitignore` (add `work/`)

**Interfaces:**
- `Prepare-Assets.ps1 -SevenZipArchive <7z> -OutDir <dir>`: extracts the ISO once with 7-Zip (installs 7-Zip via winget when absent, after printing the command) and prints the ISO path.
- `Run-TwoInstance.ps1 -Exe <BrawlLive Dolphin.exe> -PPlusDir <stock folder> -IsoPath <iso> [-StubPort 8787] [-TimeoutSec 90] [-Scenario happy|swap]`: builds two instance user folders, starts the stub in direct mode, launches both Dolphins, waits for both `game_started` events, prints timings, kills everything, exits 0 or 1.
- Exit criterion of sub-project 1: `Run-TwoInstance.ps1` exits 0 for both scenarios on David's PC.

- [ ] **Step 1: Write `Prepare-Assets.ps1`**

```powershell
#Requires -Version 7
param(
  [Parameter(Mandatory)] [string] $SevenZipArchive,
  [Parameter(Mandatory)] [string] $OutDir
)
$ErrorActionPreference = 'Stop'
$sevenZip = 'C:\Program Files\7-Zip\7z.exe'
if (-not (Test-Path $sevenZip)) {
  Write-Host '7-Zip not found; installing with: winget install -e --id 7zip.7zip'
  winget install -e --id 7zip.7zip --accept-package-agreements --accept-source-agreements | Out-Null
  if (-not (Test-Path $sevenZip)) { throw '7-Zip install failed; install it manually and re-run' }
}
New-Item -ItemType Directory -Path $OutDir -Force | Out-Null
$existing = Get-ChildItem $OutDir -Filter *.iso -ErrorAction SilentlyContinue | Select-Object -First 1
if (-not $existing) {
  & $sevenZip x "$SevenZipArchive" "-o$OutDir" -y | Out-Null
  if ($LASTEXITCODE -ne 0) { throw "7-Zip extraction failed ($LASTEXITCODE)" }
  $existing = Get-ChildItem $OutDir -Filter *.iso | Select-Object -First 1
}
if (-not $existing) { throw 'No .iso found after extraction' }
Write-Host "ISO: $($existing.FullName) ($([math]::Round($existing.Length / 1GB, 2)) GB)"
$existing.FullName
```
Run: `pwsh -File tools\milestone\Prepare-Assets.ps1 -SevenZipArchive 'D:\Projects\Games\BRAWLLIVE\Super Smash Bros. Brawl (USA) (Rev 1).7z' -OutDir D:\Projects\Games\BRAWLLIVE\work\iso`
Expected: prints the ISO path; the ISO is about 7.9 GB for Brawl's dual-layer disc (any size over 4 GB is plausible; a much smaller file means a wrong dump).

- [ ] **Step 2: Write `Run-TwoInstance.ps1`**

```powershell
#Requires -Version 7
param(
  [Parameter(Mandatory)] [string] $Exe,
  [Parameter(Mandatory)] [string] $PPlusDir,
  [Parameter(Mandatory)] [string] $IsoPath,
  [int] $StubPort = 8787,
  [int] $TimeoutSec = 90,
  [ValidateSet('happy','swap')] [string] $Scenario = 'happy'
)
$ErrorActionPreference = 'Stop'
$root = Resolve-Path (Join-Path $PSScriptRoot '..\..')
$work = Join-Path $root 'work\milestone'
$app = Join-Path $work 'app'

function Set-IniValue([string] $Path, [string] $Section, [string] $Key, [string] $Value) {
  $lines = if (Test-Path $Path) { Get-Content $Path } else { @() }
  $out = New-Object System.Collections.Generic.List[string]
  $inSection = $false; $done = $false; $sectionSeen = $false
  foreach ($line in $lines) {
    if ($line -match '^\[(.+)\]$') {
      if ($inSection -and -not $done) { $out.Add("$Key = $Value"); $done = $true }
      $inSection = ($Matches[1] -eq $Section); if ($inSection) { $sectionSeen = $true }
      $out.Add($line); continue
    }
    if ($inSection -and $line -match "^\s*$([regex]::Escape($Key))\s*=") { if (-not $done) { $out.Add("$Key = $Value"); $done = $true }; continue }
    $out.Add($line)
  }
  if (-not $sectionSeen) { $out.Add("[$Section]") }
  if (-not $done) { $out.Add("$Key = $Value") }
  Set-Content $Path $out -Encoding utf8
}

# 1. App folder: full copy of the stock P+ folder with the BrawlLive exe (Sys is read from here).
if (-not (Test-Path $app)) {
  robocopy $PPlusDir $app /E /NFL /NDL /NJH /NJS /XD "user" | Out-Null
}
Copy-Item $Exe (Join-Path $app 'Dolphin.exe') -Force

# 2. Per-instance user folders: copies of the stock user folder (sd.raw, Launcher, NAND).
$instances = @('A', 'B')
foreach ($name in $instances) {
  $user = Join-Path $work "user-$name"
  if (-not (Test-Path $user)) {
    robocopy (Join-Path $PPlusDir 'user') $user /E /NFL /NDL /NJH /NJS | Out-Null
    if (Test-Path "$user\Cache (rename folder to just Cache to use)") { Rename-Item "$user\Cache (rename folder to just Cache to use)" "$user\Cache" }
  }
  $ini = Join-Path $user 'Config\Dolphin.ini'
  New-Item -ItemType Directory -Path (Split-Path $ini) -Force | Out-Null
  Set-IniValue $ini 'BrawlLive' 'ServerURL' "http://127.0.0.1:$StubPort"
  Set-IniValue $ini 'BrawlLive' 'AutoQueue' 'casual_singles'
  Set-IniValue $ini 'BrawlLive' 'DeviceToken' ''
  Set-IniValue $ini 'NetPlay' 'Nickname' "Instance$name"
  Set-IniValue $ini 'Core' 'DefaultISO' $IsoPath
  Set-IniValue $ini 'General' 'ISOPath0' (Split-Path $IsoPath)
  Set-IniValue $ini 'General' 'ISOPaths' '1'
  Set-IniValue $ini 'Interface' 'ConfirmStop' 'False'
}

# 3. Stub in direct mode.
$stub = Start-Process -FilePath 'node' -ArgumentList @("$root\tools\stub-service\main.ts", '--port', $StubPort, '--mode', 'direct', '--countdown', '2') -PassThru -WindowStyle Hidden
Start-Sleep -Seconds 1
try {
  Invoke-RestMethod "http://127.0.0.1:$StubPort/debug/reset" -Method Post | Out-Null

  # 4. Launch instances. 'swap' starts B first so B becomes the host.
  $order = if ($Scenario -eq 'swap') { @('B', 'A') } else { @('A', 'B') }
  $procs = @()
  foreach ($name in $order) {
    $procs += Start-Process -FilePath (Join-Path $app 'Dolphin.exe') -ArgumentList @('--user', (Join-Path $work "user-$name")) -PassThru
    Start-Sleep -Seconds 4   # let the first instance register before the second queues
  }

  # 5. Wait for both game_started events.
  $deadline = (Get-Date).AddSeconds($TimeoutSec)
  $started = @()
  $state = $null
  while ((Get-Date) -lt $deadline) {
    Start-Sleep -Milliseconds 500
    $state = Invoke-RestMethod "http://127.0.0.1:$StubPort/debug/state"
    $started = @($state.events | Where-Object { $_.type -eq 'game_started' })
    $errors = @($state.events | Where-Object { $_.type -eq 'error' })
    if ($errors.Count -gt 0) { Write-Host "Agent error: $($errors[0].payload | ConvertTo-Json -Compress)" -ForegroundColor Red; break }
    if ($started.Count -ge 2) { break }
  }

  Write-Host "Match states: $(@($state.matches | ForEach-Object { $_.state }) -join ', ')"
  Write-Host "Events: $(@($state.events | ForEach-Object { "$($_.device):$($_.type)" }) -join ' ')"
  if ($started.Count -ge 2 -and $state.matches[0].state -eq 'in_game') {
    Write-Host "MILESTONE PASSED ($Scenario): both instances reported game_started" -ForegroundColor Green
    $exit = 0
  } else {
    Write-Host "MILESTONE FAILED ($Scenario)" -ForegroundColor Red
    $exit = 1
  }
  Start-Sleep -Seconds 5  # keep the game up briefly for a visual check
  foreach ($p in $procs) { if (-not $p.HasExited) { Stop-Process -Id $p.Id -Force } }
  exit $exit
}
finally {
  if ($stub -and -not $stub.HasExited) { Stop-Process -Id $stub.Id -Force }
}
```
Add `work/` to `D:\Projects\Games\BRAWLLIVE\.gitignore` if Task 1 did not.

- [ ] **Step 3: First run, fix what breaks, then pass both scenarios**

Run:
```powershell
pwsh -File tools\milestone\Run-TwoInstance.ps1 -Exe D:\Projects\Games\brawllive-dolphin\Binary\x64\Dolphin.exe -PPlusDir 'D:\Projects\Games\BRAWLLIVE\Project+.v3.2.Netplay.Windows' -IsoPath <iso path from Step 1>
pwsh -File tools\milestone\Run-TwoInstance.ps1 ... -Scenario swap
```
Expected: `MILESTONE PASSED (happy)` and `MILESTONE PASSED (swap)`, each within 90 s, with two Dolphin windows showing the P+ launcher booting in netplay. Likely first-run issues and their fixes, in order of probability:
  1. The joiner's `NetPlayJoin` runs before the host's server listens: add a 1 s delay before `JoinSession` in `RunPendingAction` or let the stub delay the `join` command by 1 s after `hosting`.
  2. `DoAllPlayersHaveGame()` false because the game list has not finished scanning in a fresh user folder: wait for `GameList` to report a non-empty model before `HostSession` (poll `rowCount` on the tick and keep `pending` until found, up to the 15 s Preparing timeout).
  3. Both instances fight over input devices: set `[Core] SIDevice0 = 6` (standard controller) in instance B's ini so the adapter is not opened twice.
  Record each fix as its own commit in the fork.

- [ ] **Step 4: Stock compatibility check (manual)**

Create `work\pplus-stock` as a copy of the stock P+ folder with the original exe. Start it, Tools → Start NetPlay → Host, direct connection, port 2626, game "Project+ Netplay Launcher". Start the BrawlLive app copy with `--user work\milestone\user-A`, open Tools → Start NetPlay → Connect to `127.0.0.1:2626`. Expected: the BrawlLive build appears in the stock host's player list with revision `v3.2.0-…  Win` and no "version mismatch" error; starting the game boots both. Record the result in `BRAWLLIVE.md`.

- [ ] **Step 5: Commit (repo `brawllive`)**

```powershell
git add tools/milestone .gitignore
git commit -m "feat(tools): two-instance milestone harness and asset preparation

Co-Authored-By: Claude Fable 5.1 <noreply@anthropic.com>"
```

---

### Task 14: CI with core-path guard, build, tests, artifact and release

**Files (fork):**
- Create: `.github/workflows/brawllive.yml`
- Modify: `BRAWLLIVE.md` (release procedure)

**Interfaces:**
- Produces: on every push to `brawllive` and on tags `v*-brawllive.*`: guard job, Windows build job with unit tests and artifact `BrawlLive-Dolphin-Windows`; on tags, a GitHub Release with `BrawlLive-Dolphin-Windows.zip` containing only `Dolphin.exe`.

- [ ] **Step 1: Verify action versions**

Check the current major versions of `actions/checkout`, `actions/upload-artifact`, `actions/download-artifact`, `microsoft/setup-msbuild` and `softprops/action-gh-release` on the GitHub Marketplace and use those tags in the workflow below (the plan assumes `v4`, `v4`, `v4`, `v2`, `v2`).

- [ ] **Step 2: Write the workflow**

`.github/workflows/brawllive.yml`:
```yaml
name: BrawlLive Dolphin
on:
  push:
    branches: [brawllive]
    tags: ['v*-brawllive.*']
  pull_request:
    branches: [brawllive]

env:
  BRAWLLIVE_NETPLAY_IDENTITY: 448d1dbee83429f6a64f269a3511dc9a5fb11a98
  DOLPHIN_DISTRIBUTOR: BrawlLive

jobs:
  guard:
    runs-on: ubuntu-24.04
    steps:
      - uses: actions/checkout@v4
        with: { fetch-depth: 0 }
      - name: Fetch upstream tag
        run: git fetch https://github.com/Project-Plus-Development-Team/Project-Plus-Dolphin.git refs/tags/v3.2.0:refs/tags/v3.2.0
      - name: Core path guard
        shell: pwsh
        run: ./tools/core-path-guard.ps1 -Base v3.2.0 -Head HEAD

  build-windows:
    needs: guard
    runs-on: windows-2022
    steps:
      - uses: actions/checkout@v4
        with: { submodules: recursive, fetch-depth: 0 }
      - uses: microsoft/setup-msbuild@v2
      - name: Build Dolphin and unit tests
        shell: pwsh
        run: |
          msbuild Source\dolphin-emu.sln /m /verbosity:minimal /t:DolphinQt;UnitTests /p:Configuration=Release /p:Platform=x64
          if ($LASTEXITCODE -ne 0) { exit 1 }
      - name: Run unit tests
        shell: pwsh
        run: |
          $t = Get-ChildItem -Recurse Build\x64\Release -Filter 'UnitTests*.exe' | Select-Object -First 1
          & $t.FullName --gtest_brief=1 --gtest_filter='BrawlLive*:Protocol*:PadCombo*:MatchState*:Agent*'
          if ($LASTEXITCODE -ne 0) { exit 1 }
      - name: Verify netplay identity
        shell: pwsh
        run: |
          $text = [Text.Encoding]::ASCII.GetString([IO.File]::ReadAllBytes('Binary\x64\Dolphin.exe'))
          if (-not $text.Contains($env:BRAWLLIVE_NETPLAY_IDENTITY)) { Write-Host 'identity missing'; exit 1 }
      - name: Package
        shell: pwsh
        run: |
          New-Item -ItemType Directory dist | Out-Null
          Compress-Archive -Path Binary\x64\Dolphin.exe -DestinationPath dist\BrawlLive-Dolphin-Windows.zip
          Get-FileHash dist\BrawlLive-Dolphin-Windows.zip | Out-File dist\SHA256.txt
      - uses: actions/upload-artifact@v4
        with:
          name: BrawlLive-Dolphin-Windows
          path: dist/

  release:
    if: startsWith(github.ref, 'refs/tags/')
    needs: build-windows
    runs-on: ubuntu-24.04
    permissions: { contents: write }
    steps:
      - uses: actions/download-artifact@v4
        with: { name: BrawlLive-Dolphin-Windows, path: dist }
      - uses: softprops/action-gh-release@v2
        with:
          files: |
            dist/BrawlLive-Dolphin-Windows.zip
            dist/SHA256.txt
          body: "BrawlLive Dolphin for Project+ 3.2.0. Drop Dolphin.exe into your P+ folder with tools/installer/Install-BrawlLive.ps1. Netplay identity 448d1dbe (stock-compatible)."
```

- [ ] **Step 3: Push the branch and watch the run**

```powershell
git add .github/workflows/brawllive.yml BRAWLLIVE.md
git commit -m "ci(brawllive): guard, Windows build, tests, artifact, release

Co-Authored-By: Claude Fable 5.1 <noreply@anthropic.com>"
git push -u origin brawllive
gh run watch
```
Expected: `guard` and `build-windows` green; artifact downloadable. Note: the fork's existing `ci.yml` also triggers on push; disable it on our branch by adding `if: github.repository == 'Project-Plus-Development-Team/Project-Plus-Dolphin'` to each of its jobs in a separate commit, so only our workflow runs.

- [ ] **Step 4: Tag the first pre-release**

```powershell
git tag -a v3.2.0-brawllive.1 -m "BrawlLive Dolphin pre-release 1"
git push origin v3.2.0-brawllive.1
gh run watch
gh release view v3.2.0-brawllive.1
```
Expected: a GitHub Release with the zip and hash file. Then Task 11's Check for Updates in the work copy reports the release. Record the release procedure in `BRAWLLIVE.md`: bump tag, push, verify release, run `Run-TwoInstance.ps1` against the downloaded exe.

---

## Spec coverage

| Spec section | Task |
|---|---|
| 6.1 agent, overlay, controller, config, identity, updater | 2, 3, 5, 6, 8, 9, 10, 11 |
| 6.5 installer | 12 |
| 7 protocol v1 (client side) | 3, 6 |
| 8 match lifecycle (client view, local timeouts) | 5, 9 |
| 9 in-game UI and input | 4, 8, 9 |
| 10 error handling (codes, backoff, offline) | 5, 6, 9 |
| 12 testing (gtest, simulated service, milestone, compat) | every task; 7, 13 |
| 13 CI path guard, artifact, release | 2, 14 |
| 14.1 exit criterion | 13 |

Deferred to later sub-projects by the spec: the real service (SP2), website (SP3), ranked result claims (SP4), doubles pad maps beyond the convention (SP5), domain and VPS (SP6). The agent and state machine already carry `ranked_*` and `*_doubles` modes and four-slot pad maps so no protocol change is needed later.

## Execution notes

- Model use per David's rule: this plan is executed by Opus or Sonnet workers per task with the orchestrator verifying each task's tests and commits; mechanical registration edits (vcxproj, CMake) can go to Haiku. No per-task Claude reviewers; one Codex review pass after Task 14, then targeted re-tests.
- Build time is the bottleneck. Run `build.ps1 -Tests` once per task, not per step, unless a step says otherwise. Keep the baseline build directory so incremental builds take minutes.
- Every fork task runs `tools/core-path-guard.ps1` before committing.

