# Changelog

## 1.1.6 (2026-08-15)

Follow-up to 1.1.5: the fan control could set a speed but never showed the
current one.

### Climate

- **the current fan speed is now reported.** Dimension-11 events
  (`*#4*<zone>*11*<speed>##`) were parsed but carried no message type, and the
  climate entity dispatches on exactly that, so they were dropped: the speed
  was only ever picked up indirectly from the valve status (dimension 19),
  which not every zone sends. They now carry a dedicated `fan_speed` type and
  are handled on their own.
- the new branch updates only the fan attributes. Reusing the existing
  "action" handling would have been shorter but would have read the speed as a
  valve state and reported a heating/cooling action that is not happening.

## 1.1.5 (2026-08-15)

### Climate

- **fan coil speed can now be set.** Zones with fan support expose the
  standard Home Assistant fan control (`auto`, `low`, `medium`, `high`),
  sending `*#4*<zone>*#11*<speed>##`. Until now the fan speed was only parsed
  from the bus, never surfaced (`ClimateEntityFeature.FAN_MODE` was never set)
  and never settable — `async_set_fan_mode` was commented out and OWNd had no
  command for it.
- the write scale mirrors the one the zone already reports in its
  dimension-11 events: 0 auto, 1 low, 2 medium, 3 high.
- there is deliberately no "off" fan mode: switching the zone off is what
  `HVACMode.OFF` is for, and an off fan mode would duplicate it. A fan the bus
  reports as stopped leaves the last known speed shown rather than an invalid
  mode.
- unchanged for zones without fan coils: fan support is opt-in per device
  (`fan: true`, default `false`), so nothing new appears on radiator or
  underfloor zones.

### Web panel

- the climate `fan` flag now defaults to **off** in the manual add form, in
  the discovery candidates and in the API, matching the YAML schema. It used
  to default to on, which was harmless while the flag did nothing but would
  now add a fan control to every zone added from the panel, fan coil or not.
- **climate flags are now editable in place.** Configured zones show
  `heat`/`cool`/`fan`/`standalone` as checkboxes with a Save button, so
  enabling the fan control on an existing zone no longer means deleting and
  recreating it. Saving reloads the config entry, so the new capabilities
  appear without restarting Home Assistant.

## 1.1.4.1 (2026-07-28)

Follow-up to 1.1.4, fixing two problems found while configuring a motion
sensor from the panel.

### Web panel

- text fields no longer lose focus while typing. The panel re-rendered its
  whole DOM on every `hass` assignment — that is, on every state change of
  every entity — so on a busy system the input being typed into was destroyed
  and recreated a few characters at a time. The panel does not display entity
  states, so it now renders once and then only on explicit user actions.
- a motion sensor configured as a WHO 1 `binary_sensor` is no longer proposed
  by the discovery as a "new light". Motion sensors share the WHO 1 address
  space with lights, and a PIR driving a dedicated address announces itself
  with WHAT 34: importing it as a light would create an entity that can never
  turn on. Same treatment already applied to switches in 1.1.2.

### Motion sensors

- motion binary sensors accept an optional `timeout` (5-3600 seconds),
  configurable from the panel, controlling how long the entity stays `on`
  after the last detection. A configured value always wins over the one
  reported by the bus: a sensor driving a dedicated (unused) address has no
  device answering the motion-timeout query, so previously the entity was
  stuck on the built-in default regardless of the timer set on the sensor.
- the built-in default is now **300s** (was 315s) and is still used when no
  timeout is configured and the device does not report one.

## 1.1.4 (2026-07-28)

### Web panel

- the `binary_sensor` platform can now be configured from the web panel:
  it is selectable in the manual add form, with a WHO selector (25 dry
  contact, 1 motion sensor, 9 auxiliary), the full list of supported device
  classes and an `inverted` flag. Configured binary sensors are listed and
  removable like any other device. Previously the platform existed in the
  integration but was only reachable through the one-time YAML import, so on
  installs already migrated to storage it could not be configured at all.
- WHO 1 endpoints are accepted only with the `motion` device class, which is
  the only combination `binary_sensor.py` instantiates; any other class is
  rejected with an explicit message instead of being silently persisted
  without producing an entity.

- the panel script URL now carries the integration version as a query
  parameter, so browsers stop serving a cached copy of the panel after an
  update (previously a manual hard-refresh was needed to see new features).

### Naming

- the sidebar entry and the panel header now read simply **bticino MyHome**,
  dropping the "Unofficial Integration" suffix. After updating, hard-refresh
  the panel page (Ctrl+F5) to drop the cached panel script.

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
