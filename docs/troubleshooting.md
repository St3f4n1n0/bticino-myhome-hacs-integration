# Troubleshooting

## Connection behavior

The integration keeps one persistent **event session** (push updates) and one
or more **command sessions** to the gateway.

On connection loss it reconnects automatically with exponential backoff
(2s, 4s, 8s, ... capped at 60s), closing the previous socket first. MyHome
gateways allow only a few concurrent sessions: if the gateway refuses
connections, the integration keeps retrying with backoff and Home Assistant
shows the config entry as "Retrying setup" instead of failing hard.

TCP keepalive is enabled on every session (since 1.1.3). A gateway or an
intervening router/NAT can drop an idle connection without sending a
FIN/RST; without keepalive the blocking read on the event session would
hang forever and pushed state updates would silently stop until a reload.
The kernel now probes an idle connection (first probe after 60s, then every
10s, dropped after 3 unanswered probes), so a dead peer is detected in ~90s
and reconnection follows automatically. If you still see states freeze,
capture the log around the freeze time and check whether reconnection
messages appear afterwards.

Repeated `Connection refused` from a previously working gateway usually means
the gateway ran out of sessions (other clients connected, or a firmware
freeze): power-cycle the gateway.

## Useful log filters

Enable debug logging in `configuration.yaml`:

```yaml
logger:
  logs:
    custom_components.myhome: debug
```

Common messages:

- `Successfully connected to gateway.` — event session up.
- `Connection failed, retrying in N seconds` — backoff in progress; check
  gateway reachability if it persists.
- `Connection lost during message processing` — gateway dropped the session;
  reconnection follows with backoff.
- `Error processing message ..., skipping it.` — one malformed/unexpected bus
  message was skipped; the session stays up. Please report these with the
  logged message content.
- `Dropping message ... after 3 failed attempts.` — a command could not be
  delivered; check gateway health.
- `Unhandled heating command for zone N (dimension D) ...` — a WHO 4
  dimension-writing frame the integration does not act on (e.g. dimension
  17, a write-echo broadcast by a thermostat/central unit). It is traced,
  not an error: the zone's real state (temperature, setpoint, mode, valve)
  arrives through other frames that are handled. Before 1.1.3 these showed
  up as `Unsupported message type` warnings.

## Common issues

- **"Configured devices: 0" after migrating (all entities unavailable)** —
  affected versions up to 1.1.0: the one-time YAML import did not recognize
  the legacy layout (named section such as `server1:` with an inner `mac:`
  field) and persisted an empty configuration. Fixed in **1.1.1**. Recovery:
  update the integration, then with HA Core stopped (`ha core stop` from the
  SSH add-on or the host shell via `login`) delete
  `/config/.storage/myhome_config` (from the host shell:
  `/mnt/data/supervisor/homeassistant/.storage/myhome_config`), then
  `ha core start`. The YAML is re-imported at boot.
- **Discovery proposes your switches as "new lights"** — switches share the
  WHO 1 address space with lights. Fixed in **1.1.2**: WHO 1 endpoints
  configured as switch are treated as known. Do not import them as lights:
  they would clash with the switch unique_ids. After updating, use
  "Clear list" and hard-refresh the panel page (Ctrl+F5) to drop the cached
  panel script.

- **Entities stop updating (and only a reload fixes it)** — typically a
  silently dropped event session. Addressed in **1.1.3** with TCP keepalive,
  which detects a dead connection in ~90s and reconnects automatically. If it
  persists, check the log for reconnection loops and verify the gateway
  session limit.
- **Wrong password** — the config entry prompts for re-authentication. The
  OPEN password can be numeric (OPEN algorithm) or alphanumeric (HMAC).
- **Switch on a bus interface not refreshing** — fixed in this fork (status
  requests now include the `#4#<interface>` suffix).
- **Negative outdoor temperatures shown as positive** — fixed in this fork
  (sign digit is now parsed).
- **Climate set-temperature does nothing** — by design the integration does
  not send set-point commands while the zone is OFF.

## Reporting issues

Open an issue at
https://github.com/St3f4n1n0/bticino-myhome-hacs-integration/issues with:
gateway model (e.g. MH201/MH202/F454), integration version, relevant debug
log excerpt.
