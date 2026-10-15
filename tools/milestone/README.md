# Milestone harness (sub-project 1, Task 13)

Exit criterion of sub-project 1: two Dolphin instances on one PC auto-host, auto-join and auto-start a P+ match against the local stub, in both host orders.

Files to be created by Task 13 of the plan: `Prepare-Assets.ps1` (extracts the player's own Brawl ISO once with 7-Zip) and `Run-TwoInstance.ps1` (builds two `--user` folders, starts the stub in direct mode, launches both Dolphins, waits for both `game_started` events).

```powershell
pwsh -File Prepare-Assets.ps1 -SevenZipArchive <your Brawl dump .7z> -OutDir ..\..\work\iso
pwsh -File Run-TwoInstance.ps1 -Exe <BrawlLive Dolphin.exe> -PPlusDir <stock P+ folder> -IsoPath <iso>
pwsh -File Run-TwoInstance.ps1 ... -Scenario swap
```

Everything the harness creates lives under `work/`, which is git-ignored. Game files never leave the player's machine.
