# nts-sync Configuration Reference

The configuration file uses TOML format. Default location: `%ProgramData%\nts-sync\config.toml`.

Generate a default config with all options (prints to stdout):
```
nts-sync-ctl config generate > "%ProgramData%\nts-sync\config.toml"
```

View the current active config:
```
nts-sync-ctl config show
```

## `[general]`

General timing and threshold settings. All fields are optional with sensible defaults.

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `log_level` | string | `"info"` | Tracing log level filter. Values: `trace`, `debug`, `info`, `warn`, `error`, `off`. Case-insensitive. |
| `step_threshold_ms` | float | `128.0` | Clock offsets above this value (in milliseconds) are corrected by stepping the clock immediately. Must be > 0. |
| `panic_threshold_s` | float | `1000.0` | Clock offsets above this value (in seconds) cause the service to abort. Protects against catastrophic clock changes. Must be > `step_threshold_ms / 1000`. |
| `initial_step` | bool | `true` | Allow an immediate clock step on the first synchronization. When `false`, the first correction is always a gradual slew. |
| `poll_interval_s` | integer | `64` | Base poll interval in seconds. With multi-server, each source adapts its own interval (16s–1024s) around this base. Range: 16–86400. |
| `timeout_ms` | integer | `5000` | Network timeout in milliseconds for NTS-KE and NTP operations. Must be >= 100. |
| `persist_interval_secs` | integer | `900` | Interval in seconds for persisting NTS session state (cookies, keys) to disk. Range: 0–86400. Set to `0` to disable periodic persistence (state is still saved after each KE handshake). |

### Example

```toml
[general]
log_level = "info"
step_threshold_ms = 128.0
panic_threshold_s = 1000.0
initial_step = true
poll_interval_s = 64
timeout_ms = 5000
persist_interval_secs = 900
```

## `[[server]]`

Server entries use TOML's array-of-tables syntax (`[[server]]`). At least one server is required. Multiple servers enable source selection and failover.

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `host` | string | *(required)* | NTS-KE server hostname. Must not be empty. |
| `ke_port` | integer | `4460` | TCP port for NTS Key Exchange. Must be > 0. |
| `ntp_port` | integer | `0` | UDP port for NTP queries. `0` means use the port negotiated during NTS-KE (typically 123). Set explicitly only if the server uses a non-standard NTP port. |
| `native_certs` | bool | `false` | Use the Windows native certificate store for TLS verification instead of the bundled webpki roots. Enable if the server uses certificates from a private CA or if you encounter certificate validation errors. |
| `prefer` | bool | `false` | Mark this server as preferred. In multi-server mode, the preferred server is selected as long as it passes reachability and delay filters, even if another source has a lower root distance. |

### Example

```toml
[[server]]
host = "time.cloudflare.com"
ke_port = 4460
native_certs = false
prefer = true

[[server]]
host = "ntppool1.time.nl"

[[server]]
host = "ptbtime1.ptb.de"
```

### Known Public NTS Servers

| Server | Stratum | Location | Notes |
|--------|---------|----------|-------|
| `time.cloudflare.com` | 3 | Anycast | Widely used, good global coverage |
| `ntppool1.time.nl` | 1 | Netherlands | |
| `ptbtime1.ptb.de` | 1 | Germany | Strict anti-amplification |
| `ptbtime2.ptb.de` | 1 | Germany | Strict anti-amplification |
| `nts.netnod.se` | 1 | Sweden | Redirects NTP to 194.58.203.196:4123 |
| `ntp3.fau.de` | 1 | Germany | Strict anti-amplification |

## `[logging]`

Optional file logging configuration. When enabled, logs are written to daily rotating files in `%ProgramData%\nts-sync\logs\` (filename pattern: `nts-sync.YYYY-MM-DD.log`) in addition to the Event Log. Logging configuration changes require a service restart to take effect.

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `file_enabled` | bool | `false` | Enable file logging. |
| `file_max_size_mb` | integer | `10` | Reserved for future use. Rotation is currently daily (time-based, not size-based). |
| `file_max_files` | integer | `5` | Number of rotated log files to retain. Must be 1–365. |

### Example

```toml
[logging]
file_enabled = true
file_max_files = 7
```

## Full Example

```toml
[general]
log_level = "info"
step_threshold_ms = 128.0
panic_threshold_s = 1000.0
initial_step = true
poll_interval_s = 64
timeout_ms = 5000
persist_interval_secs = 900

[[server]]
host = "time.cloudflare.com"
prefer = true

[[server]]
host = "ntppool1.time.nl"

[[server]]
host = "ptbtime1.ptb.de"

[logging]
file_enabled = false
```

## Validation Rules

The configuration is validated at load time. The following rules are enforced:

- At least one `[[server]]` entry is required
- `host` must not be empty
- `ke_port` must be > 0
- `step_threshold_ms` must be > 0
- `panic_threshold_s` must be > 0
- `panic_threshold_s` must be > `step_threshold_ms / 1000` (panic threshold must be strictly larger than step threshold)
- `poll_interval_s` must be >= 16 and <= 86400
- `timeout_ms` must be >= 100 and <= 60000
- `persist_interval_secs` must be <= 86400
- `file_max_files` must be between 1 and 365
- `file_max_size_mb` must be between 1 and 1024

Validate a config file without starting the service:
```
nts-sync-ctl config validate --config config.toml
```
