# NTS-TimeTools
# nts-sync User Guide

## Overview

nts-sync is a Windows service that synchronizes your system clock using NTS (Network Time Security, RFC 8915). It replaces the built-in Windows Time Service (W32Time) with cryptographically authenticated time synchronization.

## Requirements

- Windows 7 or later (x64); Windows 8+ recommended for precise clock slewing
- Administrator privileges (for installation and clock adjustment)
- Network access to NTS servers on TCP 4460 and UDP 123

## Installation

### Using the Installer

1. Run `nts-sync-<version>-setup.exe` as Administrator.
2. Follow the prompts. The installer will:
   - Copy `nts-sync-service.exe` and `nts-sync-ctl.exe` to `%ProgramFiles%\nts-sync\`
   - Generate a default config at `%ProgramData%\nts-sync\config.toml`
   - Register the Windows service
   - Optionally start the service
3. The service is configured for automatic startup on boot.

### Manual Installation

1. Copy the binaries:
   ```
   copy nts-sync-service.exe "C:\Program Files\nts-sync\"
   copy nts-sync-ctl.exe "C:\Program Files\nts-sync\"
   copy nts-sync-tray.exe "C:\Program Files\nts-sync\"
   ```

3. Generate a default config:
   ```
   nts-sync-ctl config generate > "%ProgramData%\nts-sync\config.toml"
   ```

4. Edit the config as needed, then install the service (as Administrator):
   ```
   nts-sync-ctl install --config config.toml
   ```
   This will:
   - Register the service with the Windows SCM
   - Assign `SeSystemtimePrivilege` to the service account
   - Copy the config to `%ProgramData%\nts-sync\config.toml`
   - Create the state directory
   - Set service description ("Provides system time synchronization based on NTS")

   The config path can be relative or absolute — it is automatically resolved.

5. Start the service:
   ```
   nts-sync-ctl start
   ```

## System Tray Monitor

nts-sync includes a lightweight system tray monitor (`nts-sync-tray.exe`) that displays real-time synchronization status.

### What It Does

The tray icon sits in the Windows notification area and polls the nts-sync service for status updates. It shows at-a-glance sync health without requiring an admin command prompt.

### Running

Launch manually:
```
nts-sync-tray.exe
```

Or, if installed via the installer, enable the "Start tray monitor on login" option. This creates a scheduled task that launches the tray monitor at user logon.

Only one instance can run at a time - launching a second instance will silently exit.

### Icon States

| Icon Color | Meaning |
|------------|---------|
| Green | Synchronized - clock is in sync |
| Yellow | Transitional - sync in progress (initial key exchange or convergence) |
| Red | Error - sync failures or service errors |
| Grey | Offline - shows categorized reason: "Service not running", "Access denied", "Service busy", or "Connection timed out" |

### Tooltip and Context Menu

- **Tooltip**: Hover over the icon to see current status, offset, and server name.
- **Context menu** (right-click):
  - Status summary with offset (read-only)
  - Sources submenu - per-source offsets, selected source marked with checkmark
  - Tracking - frequency, discipline mode, leap indicator (read-only)
  - Reload Config - sends reload command to service (requires elevation)
  - Open logs folder - opens `%ProgramData%\nts-sync\logs\` in Explorer
  - Exit

### Behavior

- Polls the service every **5 seconds** when connected, **15 seconds** when offline
- Uses named pipe IPC (`\\.\pipe\nts-sync`) - requires no additional network access
- Connects once per poll cycle via atomic `Snapshot` request (returns status, sources, and tracking in one response)
- Automatically reconnects when the service restarts

---

## Configuration

The configuration file is TOML format, located at `%ProgramData%\nts-sync\config.toml` by default.

### Minimal Configuration

```toml
[[server]]
host = "time.cloudflare.com"
```

### Recommended Configuration

```toml
[general]
log_level = "info"
poll_interval_s = 64

[[server]]
host = "time.cloudflare.com"

[[server]]
host = "nts1.ntp.hr"

[[server]]
host = "ptbtime1.ptb.de"
```

See [Config_Reference.md](Config_Reference.md) for all options.

### Generating Default Config

The `generate` command prints a default configuration to stdout. Redirect it to a file:

```
nts-sync-ctl config generate > "%ProgramData%\nts-sync\config.toml"
```

> **Note:** The `%ProgramData%\nts-sync\` directory must exist. Create it first if installing manually: `mkdir "%ProgramData%\nts-sync"`.

### Viewing Current Config

```
nts-sync-ctl config show
```

Displays the parsed configuration from the default location (`%ProgramData%\nts-sync\config.toml`).

### Validating Config

```
nts-sync-ctl config validate --config "%ProgramData%\nts-sync\config.toml"
```

## Service Management

All management commands require an elevated (Administrator) command prompt.

### Install

```
nts-sync-ctl install --config "%ProgramData%\nts-sync\config.toml"
```

Options:
- `--disable-w32time` — Stop and disable the Windows Time Service (W32Time). Recommended to avoid clock conflicts.
- `--system-account` — Run as LOCAL SYSTEM instead of the default virtual service account (`NT SERVICE\nts-sync`).

### Start / Stop / Restart

```
nts-sync-ctl start
nts-sync-ctl stop
nts-sync-ctl restart
```

### Reload Configuration

```
nts-sync-ctl reload
```

Reloads the configuration file without restarting the service. Server additions, removals, and parameter changes take effect immediately. Log level changes are applied in real-time.

**Note:** Reload requires an elevated (admin) command prompt. Non-elevated callers receive "Access denied: reload requires an elevated (admin) process".

### Remove

```
nts-sync-ctl stop
nts-sync-ctl remove
```

### Console Mode

For testing or debugging, run the service interactively:

```
nts-sync-service console --config config.toml
```

The `--config` flag defaults to `config.toml` in the current directory if not specified.

Press Ctrl+C to stop. The clock tick rate is reset to nominal on exit.

## Monitoring

**Note:** Status, sources, tracking, and snapshot commands work from both elevated and non-elevated command prompts. Only `reload` requires elevation.

### Status

```
nts-sync-ctl status
```

Example output:
```
State:       Synchronized
Server:      nts1.ntp.hr
Offset:      0.342 ms
Delay:       12.500 ms
Stratum:     3
Last sync:   2026-03-22T10:15:30Z
Poll:        64s
Cookies:     6
KE count:    1
NTP count:   47
NTP errors:  0
Steps:       1
Slews:       46
Reachable:   11111111
Frequency:   0.123456 ppm
```

### Source Information

```
nts-sync-ctl sources
```

Shows per-source reachability, offset, delay, jitter, and leap indicator. The selected source is marked with `*`.

Example output:
```
* time.cloudflare.com [Synchronized] reach=11111111 offset=0.342 ms delay=12.500 ms jitter=0.150 ms leap=NoWarning
  nts1.npt.hr [Polling] reach=11111100 offset=0.510 ms delay=25.300 ms jitter=0.280 ms leap=NoWarning
  ptbtime1.ptb.de [Polling] reach=11110000 offset=0.180 ms delay=18.900 ms jitter=0.200 ms leap=NoWarning
```

### Tracking

```
nts-sync-ctl tracking
```

Shows clock discipline parameters: frequency correction (ppm), phase offset, discipline mode, and leap indicator.

Example output:
```
Frequency:   0.123456 ppm
Offset:      0.342000 ms
Mode:        PLL
Sources:     1/3
Leap:        NoWarning
```

### Snapshot

```
nts-sync-ctl snapshot
```

Returns status, sources, and tracking from a single atomic engine state read — guarantees all three sections reflect the same moment. Example output:

```
=== Status ===
State:       Synchronized
Server:      ntp1.nts.hr
Offset:      -0.220 ms
...

=== Sources ===
* ntppool1.time.nl [syncing] reach=00000011 offset=-0.220 ms ...
  time.cloudflare.com [syncing] reach=00000111 offset=3.853 ms ...

=== Tracking ===
Frequency:   -15.651000 ppm
Offset:      -0.220000 ms
Mode:        slew
Sources:     1/3
Leap:        0
```

## How It Works

### Startup Sequence

1. Reset the clock tick rate to nominal (crash recovery safety)
2. Check if W32Time is running (logs a warning if so)
3. Perform NTS Key Exchange (NTS-KE) over TLS 1.3 with each configured server
4. Begin polling NTP with authenticated requests

### Clock Adjustment

- **Step**: If the offset exceeds the step threshold (default 128ms), the clock is set immediately. This happens on first sync and whenever the offset is large.
- **Slew**: For smaller offsets, the clock tick rate is adjusted to gradually converge. The PLL/FLL hybrid discipline blends proportional (PLL) and frequency-based (FLL) corrections.
- **Panic**: If the offset exceeds the panic threshold (default 1000s), the service aborts to avoid catastrophic clock changes.

### Key Exchange and Cookies

- NTS-KE establishes cryptographic keys and provides cookies for authenticated NTP queries
- Cookies are consumed with each NTP query; the server returns fresh cookies in responses
- If all cookies are exhausted, the service automatically re-keys
- After 8 consecutive NTP failures, a re-key is triggered
- Keys and cookies are persisted with DPAPI encryption for fast restart recovery

### State Persistence

- Lifetime counters (KE exchanges, NTP queries, errors, step/slew adjustments) are persisted to `state/sync.json` and survive service restarts
- NTS-KE keys and cookies are persisted with DPAPI encryption for fast restart recovery
- Frequency correction is saved to `state/frequency.dat` every ~15 minutes for fast reconvergence after restart

### Source Selection (Multi-Server)

With multiple servers configured, the service independently polls each source and selects the best one:

1. Sources with no recent responses (reachability = 0) are excluded
2. Sources with excessive delay (> 1.5s) are excluded
3. The source with the lowest root distance is selected
4. If a source is marked `prefer = true`, it is used if it passes the filters

## Event Log

nts-sync writes to the Windows Application Event Log under source "nts-sync":

| Event ID | Description |
|----------|-------------|
| 1000 | Service started |
| 1001 | Service stopped |
| 1002 | Initial time synchronization |
| 1003 | Clock stepped |
| 1004 | Server unreachable |
| 1005 | Key exchange failed |
| 1006 | Key exchange successful |
| 1007 | Configuration reloaded |
| 1008 | Panic threshold exceeded |

> **Note:** If the service panics, the panic message is logged as Event ID 1001 (Service stopped) with a "Service panicked:" prefix before the process terminates.

## Troubleshooting

### Service won't start

- **"OpenSCManager failed"** — Run as Administrator.
- **"Config validation failed"** — Check config syntax: `nts-sync-ctl config validate --config <path>`
- **"Failed to create sync engine"** — Check that the config file path in the service registration is correct.

### Clock not synchronizing

1. Check status: `nts-sync-ctl status`
2. If State is "KeyExchange" or "Error":
   - Verify network connectivity to the NTS server on TCP 4460
   - Try console mode with debug logging: `nts-sync-service console --config config.toml` (set `log_level = "debug"` in config)
3. Check Event Log for error details.

### W32Time conflict

If both nts-sync and W32Time are running, they will fight over clock adjustments. Either:
- Disable W32Time: `sc config w32time start= disabled && sc stop w32time`
- Or use the `--disable-w32time` flag during installation

### Large initial offset

If the clock is far off, the first sync will step it (if `initial_step = true`). If the offset exceeds `panic_threshold_s`, the service will abort. Increase the panic threshold or manually set the clock closer before starting.

### Common error codes

- **"Failed to open service 'nts-sync' (error 1060)"** — Service not installed. Run `nts-sync-ctl install --config <path>` first.
- **"Failed to start service (error 1056)"** — Service is already running.
- **"Failed to start service (error 1053)"** — Service did not respond in time. Check Event Log for details.
- **"error 5"** on any command — Access denied. Run as Administrator.

### "Failed to start service dispatcher" when running nts-sync-service

The `run` subcommand is for the Windows SCM only. For foreground testing, use:
```
nts-sync-service console --config config.toml
```

### Certificate errors

NTS-KE uses TLS 1.3. If you get certificate errors:
- Ensure the server hostname is correct
- Try `native_certs = true` in the server config to use the Windows certificate store
- Check that your system's CA certificates are up to date

## Uninstallation

### Using the Installer

Run the uninstaller from Add/Remove Programs, or re-run the installer and choose Uninstall.

### Manual Uninstallation

```
nts-sync-ctl stop
nts-sync-ctl remove
```

Optionally remove data:
```
rmdir /s /q "%ProgramData%\nts-sync"
rmdir /s /q "%ProgramFiles%\nts-sync"
```

Re-enable W32Time if needed:
```
sc config w32time start= auto
sc start w32time
```

## File Locations

| Path | Contents |
|------|----------|
| `%ProgramFiles%\nts-sync\` | Binaries (`nts-sync-service.exe`, `nts-sync-ctl.exe`, `nts-sync-tray.exe`) |
| `%ProgramData%\nts-sync\config.toml` | Configuration file |
| `%ProgramData%\nts-sync\state\` | Persisted state (DPAPI-encrypted cookies, frequency drift, sync counters) |
| `%ProgramData%\nts-sync\logs\` | Log files (daily rotation, when file logging is enabled) |
