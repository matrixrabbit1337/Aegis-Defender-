# Aegis — cross-platform behavioral anti-ransomware agent

**Phase 1: user-space MVP.** Aegis watches the directories you choose and kills
ransomware-like behavior before it finishes: mass file encryption, extension
renaming, ransom notes, and canary tripwires. When it fires, it kills the
offending process, quarantines the binary, writes an incident report, and can
roll your files back to their pre-attack state.

Fully local. No telemetry, no cloud, no account. MIT licensed.

> **Honest scope:** Phase 1 runs entirely in user space (watchdog + psutil).
> It cannot see what kernel-level ransomware does below the OS file APIs, and
> process attribution is best-effort (see *Limitations*). Kernel sensors
> (eBPF / EndpointSecurity / minifilter) and the exfiltration guard arrive in
> later phases. This tool does not claim to beat commercial EDR — it is an
> open, local, cross-platform layer that did not exist before.

## Author

Developed by **[@matrixrabbit1337](https://instagram.com/matrixrabbit1337)**.

Built as an open, local-first contribution to the cybersecurity community.

## Install

One installer per operating system (`install.sh` just picks the right one):

| OS | Run | File |
|---|---|---|
| Linux | `./install.sh` | `install_linux.sh` |
| macOS | `./install.sh` | `install_macos.sh` |
| Windows | `powershell -ExecutionPolicy Bypass -File .\install_windows.ps1` | `install_windows.ps1` |

Each installer checks for Python 3.10+ and, if it is missing, offers to fetch
it via the native package manager (apt / dnf / pacman, Homebrew, winget) —
with your confirmation first. It then creates `.venv/`, installs the pinned
dependencies and the `aegis` CLI, and drops a launcher that runs
`aegis launch` (first-run setup, then protection; the window stays open if
something goes wrong):

- **Linux:** `~/.local/share/applications/aegis.desktop` (+ `~/Desktop` copy).
- **macOS:** `~/Desktop/Aegis.command` — double-click.
- **Windows:** `Aegis.lnk` on the Desktop.

If Python cannot be installed automatically the installer stops and prints the
exact manual command for your OS, then re-run it.

## Quick start

```bash
# 1. Pick folders, write the config, plant canaries (interactive)
.venv/bin/aegis setup            # Windows: .venv\Scripts\aegis.exe setup

# 2. Start the daemon (Ctrl+C to stop) — or use the desktop icon
.venv/bin/aegis watch

# 3. Check status any time
.venv/bin/aegis status

# 4. After an incident: preview, then roll back
.venv/bin/aegis rollback --since inc-20261008-120101-a1b2c3 --dry-run
.venv/bin/aegis rollback --since inc-20261008-120101-a1b2c3
```

Config and state live in a per-OS default location: `~/.aegis` (Linux),
`~/Library/Application Support/Aegis` (macOS), `%LOCALAPPDATA%\Aegis`
(Windows). Override with `--config` / `--state-dir`.

Example `config.yaml`:

```yaml
protected_dirs:
  - ~/Documents
  - ~/Pictures
detection:
  trigger_score: 50
response:
  kill_process: true
  quarantine: true
```

**The agent only ever acts inside `protected_dirs`.** Nothing else on the
system is watched, snapshotted, or touched. It refuses your whole home
folder, its parents, drive roots and system directories (override with
`allow_dangerous_dirs: true` — not recommended).

## How detection works

Every filesystem event in a protected dir is scored per process over a
sliding 120-second window. When a process's score crosses the threshold
(default 50), incident response fires.

| Signal | Weight | What it catches |
|---|---|---|
| Canary tripwire touched | instant trigger | any access to a decoy file — near-zero false positives |
| Ransom-note filename (`HOW_TO_RECOVER.txt`, …) | 25 | notes dropped by the encryptor |
| High-entropy write (≥ 5.8 bits/byte) | 6 | ciphertext: raw AES ~8.0, base64/Fernet ~6.0, plaintext ~4–5 |
| Rename with extension change (`doc.txt` → `doc.txt.locked`) | 5 | mass renaming, the classic ransomware tell |

**Canaries** (`aegis seed-canaries`) are believable decoys — a tax return PDF,
a password-vault export, financial docs — seeded into each protected dir.
Legitimate software never touches them, so any modification, deletion, or
rename is treated as an attack immediately. Seeding never overwrites a real
file with the same name.

**Attribution (honest version):** watchdog reports *what* changed, not *which
process* changed it. Aegis attributes best-effort: first it looks for a live
process holding the touched file open, then falls back to the process with
the largest recent write volume. It never targets itself, PID 1, OS-critical processes (per-OS list),
or the test harness. **A canary trip never triggers a guess:** if no process
can be positively attributed, Aegis alerts and records the incident but kills
nothing. The write-volume guess is used only for behavioural triggers and can
be disabled with `attribution.top_writer_fallback: false`. True file→process telemetry needs the Phase-2 kernel sensors.

## Journal & rollback ("time machine")

At daemon start, Aegis snapshots every protected file (under the size cap)
into a journal. On incident, `aegis rollback --since <incident-id>` first
shows a plan (`--dry-run` stops there), asks for confirmation (`--yes` skips
it), then:

1. restores every baseline file byte-for-byte — each backup is hash-verified
   before it is used and the result is verified after, written atomically;
2. deletes only files that did **not** exist at snapshot time (ransom notes,
   renamed ciphertext, the attacker's key file);
3. re-arms any canary the attack touched.

Safety rules: files the snapshot could not back up (over `journal.max_file_mb`,
symlinks/junctions, locked or unreadable files) are listed as *not backed up*
and are **never deleted or overwritten** by rollback — but they also cannot be
restored if the attack hit them, so raise the cap if you have big files to
protect. Baselines needed by an unresolved incident, and the newest baseline,
are never rotated away.

Caps: `journal.max_file_mb` (default 100), `journal.max_total_mb` (default
2048). Rollback restores to **agent-start state** — legitimate work done after
the agent started is lost, so preview first. True copy-on-first-write needs
the Phase-2 minifilter/eBPF sensor.

## Incident response

On trigger, Aegis: kills the offending process tree (SIGTERM, then SIGKILL),
copies the binary (or, for interpreters like python/PowerShell/node, the
script it ran) to `quarantine/`, writes `incidents/<id>/report.json` plus a
human-readable `INCIDENT.log` (PID, reasons, score, kill outcome), prints a
console alert, and tries a desktop notification. If no process can be
attributed, it still records the incident and alerts. Triggers within 5
minutes of an open incident are merged into it (one attack = one incident).

## Testing the detector (simulator)

`simulator.py` is **fake ransomware for testing only**: it Fernet-encrypts
files, appends `.locked`, and drops ransom notes — enough to exercise
detection, kill, and rollback.

**Safety is hardcoded, not documented-wished:**

- It **refuses** (exit 2) unless `--target` resolves inside a sandbox literally
  named `<system temp dir>/aegis-sim-*`. `/tmp` itself, `/`, your home dir — all refused.
- Symlinks escaping the sandbox are resolved and refused.
- It never touches anything outside the sandbox. There is no override flag.

```bash
mkdir -p /tmp/aegis-sim-test1
# put throwaway files in it, then:
.venv/bin/python simulator.py --target /tmp/aegis-sim-test1
```

The test suite (`tests/`) covers entropy scoring, rename/note/canary signals,
cooldowns, journal round-trips, all simulator guard refusals, scope
confinement (agent ignores unprotected dirs; rollback never touches outside
dirs), journal caps, and a full end-to-end run: agent vs. simulator, asserting
detection, kill, and byte-for-byte rollback.

```bash
.venv/bin/python -m pytest tests/ -q
```

## Prevention without certificates (all 3 OSes)

No Windows driver certificate or Apple entitlement is needed. Every OS gets:

- **Real-time monitoring**: inotify (Linux), ReadDirectoryChangesW (Windows),
  FSEvents (macOS), all user-space via `watchdog`.
- **Suspend-first containment** (`aegis/containment.py`): the moment an attack
  is detected the attacker's whole process tree is *frozen*, so it stops
  touching files immediately, then it is killed. Freezing is reversible if it
  was a false positive.
- **Pre-encryption trips**: canaries, the recovery-inhibition guard
  (shadow-copy deletion, recovery-partition wipe, log clearing, ESXi VM kill,
  vulnerable-driver loading) and the exfiltration guard.
- **Rollback** of anything touched before the freeze.

On Linux running as root, the fanotify sensor adds true pre-write blocking.
On Windows and macOS the user-space path means one or two files can be hit in
the instant before the freeze; rollback restores them.

## Phases implemented (from Full_Plan.html)

| Phase | What | Status |
|---|---|---|
| A — event contract | Frozen cross-platform event schema + evidence log with reason codes | Done (`events.py`, `evidence.py`) |
| B — canary + behaviour | Decoys, entropy/velocity/rename/note signals, kill, dry-run audit | Done (`canary.py`, `heuristics.py`) |
| C — protected journal + restore | Baseline + pre-modification copy, hash-verified rollback, store tripwire | Done (`journal.py`, `integrity.py`) |
| D — native enforcement | **Linux: real fanotify pre-write BLOCK (root), tested.** Windows/macOS: user-space path + per-OS adapters; signed minifilter / EndpointSecurity extension need a certificate / Apple entitlement (not buildable here) | Linux done; Win/macOS scaffolded |
| E — egress + identity | Exfiltration guard (staging + tool + novel dest + volume). Identity half (credential/OAuth exposure) needs external integrations | Egress done; identity out of scope here |
| F — hardening | Dry-run/audit mode, degraded-mode status, evidence trail, fail-open kernel path | Core done |

Research behind the detectors is in [RESEARCH.md](RESEARCH.md).

## Limitations (read before trusting it)

- **User-space only.** Kernel- or driver-level ransomware, and anything that
  operates below the file APIs, can evade this phase. Phase 2 adds eBPF
  (Linux), EndpointSecurity (macOS), and minifilter (Windows) sensors.
- **Attribution is heuristic.** A slow, careful encryptor that keeps files
  open briefly may not be attributed on the first trigger; the canary still
  fires the alert, and continued activity re-triggers with more evidence.
- **Rollback = agent-start state**, not one-second-before-attack. Not a backup
  replacement — keep real backups.
- **No exfiltration guard yet.** This phase stops encryption; watching for
  data theft before encryption is Phase 3.
- **Per-OS behaviour** (each OS has its own adapter in `aegis/platforms/`):

  | | Linux | macOS | Windows |
  |---|---|---|---|
  | File watching | inotify | FSEvents | ReadDirectoryChangesW |
  | Process attribution | open files + write volume | open files only | open files (best effort) + write volume |
  | Auto-kill when unattributed | behavioural triggers only | **never** (alerts + rollback) | behavioural triggers only |
  | Notifications | `notify-send` | `osascript` | message box |
  | Paths | case-sensitive | case-insensitive | case-insensitive, junctions skipped |

  macOS has no per-process I/O counters in psutil, and protecting
  `~/Documents` etc. needs Files-and-Folders / Full Disk Access granted to your
  terminal. Windows paths over 260 characters may be skipped by the snapshot
  (reported as not backed up). Development and live testing so far were done
  on Linux; the macOS and Windows code paths are covered by unit tests and a
  3-OS CI matrix (`.github/workflows/ci.yml`) but have had far less real-world
  use. Driver-level integration per OS is Phase 2.

## Project layout

```
aegis/            the agent package
  agent.py        wiring: config → canaries → journal → engine → response
  watchdog → FileEvent translation
  heuristics.py   scoring engine + best-effort process attribution
  canary.py       decoy seeding + integrity tripwires
  journal.py      baseline snapshots, incidents, rollback
  response.py     kill tree, quarantine, reports, alerts
  cli.py          `aegis setup|launch|watch|status|rollback|seed-canaries`
  events.py       Phase-A cross-platform event contract (frozen schema)
  sensors/        file sensors: watchdog_sensor.py (userspace), fanotify_sensor.py (Linux kernel pre-write BLOCK)
  intel.py        research-derived indicators: known ransomware extensions, ransom-note names, T1490 recovery-inhibition commands, BYOVD drivers
  recovery_guard.py  watches for shadow-copy deletion / recovery-partition wipe / ESXi VM kill (pre-encryption)
  egress.py       Phase-E exfiltration guard (archive staging + transfer tool + novel destination + volume)
  integrity.py    protected-store + persistence tripwires
  evidence.py     append-only evidence log (reason code per action)
  platforms/      one adapter per OS: linux.py, macos.py, windows.py (+ base.py)
  config.py       YAML config with defaults
simulator.py      FAKE ransomware for testing (hard sandbox guard)
tests/            pytest suite (unit + safety + end-to-end)
install.sh        picks the installer for your OS
install_linux.sh  Linux installer (python bootstrap, venv, .desktop launcher)
install_macos.sh  macOS installer (python bootstrap, venv, Aegis.command)
install_windows.ps1  Windows installer (python bootstrap, venv, Aegis.lnk)
install.ps1       shim → install_windows.ps1
requirements.txt  pinned dependencies
```

## License

MIT — see [LICENSE](LICENSE).
