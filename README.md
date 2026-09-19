# Windows 11 Update Loop Fix

![Platform](https://img.shields.io/badge/platform-Windows%2010%20%7C%2011-blue) ![License](https://img.shields.io/badge/license-MIT-green)

A Windows batch script that fixes Windows 11 getting permanently stuck on "Checking for updates", usually right after a fresh install. Unlike most reset scripts you find online, it renames the update caches instead of deleting them and leaves service permissions alone.

## What it does

- **Diagnoses first**: reports Windows build, free disk space, system time and any active Windows Update group policies before touching anything.
- **Flushes stuck transfers**: resets the BITS queue and clears the BITS database and Delivery Optimization cache.
- **Renames, never deletes**: `SoftwareDistribution` and `catroot2` become `.bak` folders, so nothing is lost.
- **Restores real service settings**: saves each service's start type from the registry and puts it back exactly, instead of forcing `auto`.
- **Fixes the clock**: syncs time with `w32tm`, because a wrong clock makes the TLS handshake to Microsoft's servers fail silently.
- **Repairs the component store**: `DISM /RestoreHealth`, then `sfc /scannow`, then `StartComponentCleanup` — with the update services running again.

## Quick start

[Download the latest release](https://github.com/exnermax/Windows-11-Update-Loop-Fix/releases/latest), unzip it and double-click `Win11UpdateFix.cmd`. It self-elevates through UAC. You need Windows 11 (it runs on Windows 10 and warns you), administrator rights, an internet connection and at least 20 GB free disk space for the feature updates that follow.

```bat
REM interactive: asks before starting and before rebooting
Win11UpdateFix.cmd

REM unattended: no prompts, no reboot
Win11UpdateFix.cmd /y
```

DISM and SFC take a while on a damaged component store. When it finishes it triggers an update scan, prints a summary and offers a reboot. Console output is in German.

## Log and rollback

The full run is logged to `C:\ProgramData\Win11UpdateFix\Win11UpdateFix.log`.

Nothing is deleted. The old caches are left at `C:\Windows\SoftwareDistribution.bak` and `C:\Windows\System32\catroot2.bak`, and Windows recreates both on the next start. Once updates work again you can delete the `.bak` folders — they can be several GB.

> [!NOTE]
> If the script is aborted mid-run, the update services stay disabled. It records the original start types in `services.state` and rolls them back automatically on the next launch.

## What it deliberately does not do

Most update-reset scripts reset service permissions with `sc.exe sdset`. Those security descriptors date from Windows Vista and 7. The Windows 10 and 11 defaults are different — they include NT SERVICE accounts and AppContainer entries that did not exist back then — so applying the old ones overwrites working permissions and can break the interaction between `wuauserv` and the Update Orchestrator (`usosvc`), causing the exact problem they claim to fix. Microsoft has removed this step from its current troubleshooting docs.

It also does not mass re-register DLLs with `regsvr32`; most of those libraries no longer exist on Windows 11.

## If it does not help

- **Service will not stop, or `SoftwareDistribution` is locked**: run the script again in Safe Mode.
- **WSUS policy warning on a home PC**: set the Windows Update policy back to *Not Configured* in `gpedit.msc`.
- **DISM error 0x800f081f**: no repair source available — mount a Windows ISO and use it as the source. Full logs are in `C:\Windows\Logs\DISM\dism.log` and `C:\Windows\Logs\CBS\CBS.log`.

## Changes in 2.0

- Removed the `sc sdset` calls, and restores the real start type per service instead of hardcoding `auto` (the Windows 11 default for `wuauserv` is `demand`).
- Repairs after the services are running again, runs `RestoreHealth` rather than only `StartComponentCleanup`, and handles `usosvc` and `dosvc` — the services Windows 11 actually uses to orchestrate updates.
- Adds diagnostics, logging, BITS queue flush, clock check, unattended mode and auto-rollback.

## License

MIT — Max Exner
