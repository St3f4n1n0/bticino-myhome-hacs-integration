# Changelog

## 1.1.3 (2026-07-09)

Connection resilience and cleaner heating-message handling.

### Connection stability

- TCP keepalive is now enabled on both the event and command sessions.
  MyHome gateways (and any intervening router/NAT) can silently drop an idle
  connection without sending a FIN/RST; the blocking read on the event
  session would then hang indefinitely, freezing all pushed state updates
  until a manual reload. The kernel now probes idle connections (first probe
  after 60s, then every 10s, dropped after 3 unanswered probes), so a dead
  peer is detected in ~90s and the listening loop reconnects on its own with
  the existing backoff. The same protection reduces the churn on the command
  session that surfaced as repeated "Reconnecting and retrying" warnings.

### Heating

- WHO 4 dimension-writing frames the integration does not act on (such as
  dimension 17, a write-echo periodically broadcast by a thermostat or
  central unit) are no longer reported as "Unsupported message type"
  warnings. They are now traced as an "Unhandled heating command" (INFO on
  first sight, then DEBUG, rate-limited). The affected zone's actual state
  keeps arriving through the frames that are already handled, so no
  functionality changes.

## 1.1.2 (2026-07-07)

- web panel: switch platform is now fully supported (listing, manual
  add/remove). Configured switches are counted and, being WHO 1 devices,
  are no longer proposed by the discovery as "new lights".

## 1.1.1 (2026-07-07)

Critical fix for the first-boot migration.

- the one-time YAML import failed for the standard legacy layout
  (named gateway section such as `server1:` with an inner `mac:` field)
  and silently persisted an empty configuration, leaving all entities
  unavailable. The gateway section is now matched by inner MAC as well,
  and an empty fallback is no longer persisted, so the import retries
  at the next boot.
- recovery for affected installs: with Home Assistant stopped, delete
  `.storage/myhome_config`, then start again.

## 1.1.0 (2026-07-07)

Stability and correctness release, addressing the findings of a full code
audit. No breaking changes; domain, unique_ids and entity_ids are unchanged.

### Connection stability

- `connect()` now raises on exhausted retries instead of silently returning:
  a refusing/unreachable gateway no longer causes a CPU busy-loop with the
  integration stuck in a fake "connected" state (C1)
- previous socket is closed before every reconnection: no more file
  descriptor leaks and stale sessions accumulating on the gateway (A3)
- single reconnection point with exponential backoff; the event reader no
  longer reconnects on its own (A4)
- backoff is applied also when the connection drops after a successful
  connect (A5)
- refused connections during setup produce a clean "retrying" state instead
  of an unhandled TypeError (A6)
- connection errors are no longer masked as "no data" (M12)

### Parsing and runtime resilience

- temperature values now honor the OWN sign digit: sub-zero readings are no
  longer reported as positive (M1)
- a malformed or unexpected bus message no longer tears down the event
  session; it is logged and skipped (M4, M5)
- failed outgoing commands are retried at most 3 times instead of being
  re-queued forever (M6)
- unload/reload now waits for worker termination (M7)
- log filters are cleaned up on failed setups (no leak) (M8)

### Entity fixes

- switch: status refresh now uses the full WHERE including the bus
  interface, like light already did (M3)
- climate: no crash when no target temperature is known; no unexpected
  zone activation when setting temperature while OFF (M9)
- sensor: legacy POWER unique_id migration now targets the mac-prefixed
  format instead of orphaning the entity (A1)

### Housekeeping

- manifest: `iot_class` corrected to `local_push`, repository references
  aligned (M10)
- dependencies: `python-dateutil` pinned, `pytz` replaced with stdlib
  `zoneinfo` (M11)
- password field in the UI is now masked and never prefilled (B1)
- removed latent debug `print()` of password material (B2)
- web panel no longer echoes internal validation errors to clients (B4)
- added `strings.json` (B7), updated GitHub Actions (B8), standard
  `async_unload_platforms` (B9)
- normalized line endings via `.gitattributes`
- new documentation: configuration, migration from anotherjulien/MyHOME,
  troubleshooting

## 1.0.0

Initial release of this fork (based on xmavgithub's work at the last
`myhome`-domain commit, with OWNd 0.7.49 vendored and llellouc's
resilience fixes applied).
