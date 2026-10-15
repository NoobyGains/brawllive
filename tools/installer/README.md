# Installer (sub-project 1, Task 12)

PowerShell 7 scripts that drop the BrawlLive `Dolphin.exe` into an existing Project+ 3.2 netplay folder, keep the stock exe as `Dolphin.exe.stock`, write `brawllive-install.json`, and restore on uninstall.

Files to be created by Task 12 of the plan: `Install-BrawlLive.ps1`, `Uninstall-BrawlLive.ps1`, `Install-BrawlLive.Tests.ps1`.

```powershell
pwsh -File Install-BrawlLive.ps1 -PPlusDir "D:\Games\Project+" -Exe .\Dolphin.exe -Build v3.2.0-brawllive.1
pwsh -File Uninstall-BrawlLive.ps1 -PPlusDir "D:\Games\Project+"
pwsh -File Install-BrawlLive.Tests.ps1
```

Known stock hash for P+ v3.2.0 Netplay Windows `Dolphin.exe`: `8206123EAB4962FC5CE995C82D47F74854AFEFD779AA4F65710296AE77E3A0BB`.
