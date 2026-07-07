# Troubleshooting

## Connection behavior

The integration keeps one persistent **event session** (push updates) and one
or more **command sessions** to the gateway.

On connection loss it reconnects automatically with exponential backoff
(2s, 4s, 8s, ... capped at 60s), closing the previous socket first. MyHome
gateways allow only a few concurrent sessions: if the gateway refuses
connections, the integration keeps retrying with backoff and Home Assistant
shows the config entry as "Retrying setup" instead of failing hard.

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

## Common issues

- **Entities stop updating** — check the log for reconnection loops; verify
  the gateway session limit; restart Home Assistant if needed.
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
