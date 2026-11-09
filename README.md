# Aegis Defender: reviewed build and Windows installer

This kit installs a reviewed, hardened build of Aegis Defender, the
cross-platform anti-ransomware agent.

## Install (Windows 10/11)

1. Extract the whole ZIP.
2. Double-click **Install_Aegis_Defender.bat** and approve the administrator
   prompt.

The installer then does everything:

- finds Python 3.10+ with Tk, or installs Python 3.12 for all users (winget,
  or the python.org installer after checking its digital signature);
- downloads the original Aegis release, checks it against a pinned hash, and
  applies the reviewed fixes;
- installs Aegis into `C:\Program Files\Aegis Defender` with its own Python
  environment and all dependencies, then checks the installed copy;
- creates an **Aegis Defender** icon on the desktop and in the Start menu,
  both set to run as administrator;
- adds Aegis Defender to Settings > Apps so it can be removed there.

It needs an internet connection: Python, the dependencies and the Aegis
release are downloaded. If GitHub is blocked, put the original
`Aegis_Defender.zip` next to the installer; it gets the same hash check.

Run the installer again to update. Settings, incidents and file snapshots are
kept in `%LOCALAPPDATA%\Aegis`. To remove Aegis, use Settings > Apps or
**Uninstall_Aegis_Defender.bat**; it asks before deleting that data.

**First run:** open Aegis Defender, add your folders or click Full Drive
Protection, then start protection. To only detect and log at first (nothing
is stopped), open `%LOCALAPPDATA%\Aegis\config.yaml` and set `dry_run: true`
under `response:`, then restart Aegis.

## What the reviewed build changes

- **Hybrid full-drive mode.** Important folders are snapshotted and can be
  rolled back. Your home folder and every local drive are watched and an
  attacker is stopped there too, but those files are not snapshotted: recover
  them from shadow copies or backups.
- **Windows full-drive and auto mode start again** (they used to fail on
  drive roots such as `C:\`).
- **Fewer false alarms:** the Linux kernel sensor scores real writes instead
  of file opens, and noisy folders (AppData, `.git`, `node_modules`, ...) are
  ignored in the watch-only tier.
- **Stop protection really stops** every background monitor.
- Smaller fixes: `~` paths, local times in the dashboard, a diagnostics tool
  (`Run_Diagnostics.cmd` in the install folder).

The installed copy's README lists the details and known limits.

## Kit contents

| File | Purpose |
| --- | --- |
| `Install_Aegis_Defender.bat` | Install or update (asks for administrator rights) |
| `Uninstall_Aegis_Defender.bat` | Remove Aegis Defender |
| `windows\Install-Aegis.ps1` | The installer the two `.bat` files run |
| `windows\aegis.ico` | Desktop icon |
| `upgrade_aegis.py` | Builds the reviewed `Aegis_Defender_Improved.zip` |
| `tests\`, `Run_Kit_Tests.bat` | Self-tests for the builder |

Linux and macOS: `python3 upgrade_aegis.py`, extract
`Aegis_Defender_Improved.zip`, then run `./install.sh` inside it.

This is a hardening pass, not a security certification. Containment on
Windows and macOS still needs testing on real machines.
