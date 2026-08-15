# Configuration guide

## How configuration is stored

The integration reads its device configuration from Home Assistant storage
(`.storage/myhome_config`). A legacy YAML file (e.g. `myhome.yml` /
`myhome.yaml`) is imported **once** into storage; after that, the YAML file is
no longer read at boot.

Practical consequence: after the first import, changes to devices must be made
from the **web panel** (or by clearing the storage entry to force a re-import).
Editing the YAML file alone has no effect. The panel manages light, switch
(since 1.1.2), cover, climate and sensor devices.

## YAML schema (legacy import)

```yaml
server1:
  mac: "00:03:50:xx:xx:xx"
  light:
    garage:
      where: "16"
      name: "Luce Garage"
  switch:
    irrigation:
      where: "31"
      name: "Irrigazione"
      class: "switch"
  cover:
    living_shutter:
      where: "21"
      name: "Tapparella Soggiorno"
  climate:
    living:
      zone: "1"
      name: "Termostato Soggiorno"
```

- `light` / `switch` / `cover` use `where` (SCS address).
- `climate` uses `zone`.
- Default WHO values: light `1`, cover `2`, climate `4`.

### Climate zones

Each zone accepts `heat` (default `true`), `cool` (default `false`),
`standalone` and `fan` (default `false`):

```yaml
  climate:
    salotto:
      zone: "22"
      name: "Termostato Salotto"
      heat: true
      cool: true
      standalone: true
      fan: true
```

Set `fan: true` only on **fan coil** zones. It adds the Home Assistant fan
control (`auto`, `low`, `medium`, `high`), which reads the speed the zone
reports and writes it back with `*#4*<zone>*#11*<speed>##` (0 auto, 1 low,
2 medium, 3 high). On radiator or underfloor zones there is no fan to drive,
so leave it off and no control is shown.

There is no "off" fan mode on purpose: turning the zone off is what the HVAC
`off` mode does, so an off fan speed would only duplicate it.

The same flags are available from the web panel, both in the manual add form
and on the discovery candidates, all defaulting to off except `heat`.

Zones that are **already configured** can be edited in place: in the panel's
device list each climate row shows the four flags as checkboxes with a **Save**
button. Saving reloads the integration, so enabling `fan` makes the fan control
appear without restarting Home Assistant. The `unique_id` is derived from the
zone (`{mac}-4-{zone}`), so the entity keeps its entity_id, name and history.

### Binary sensors

`binary_sensor` covers dry contacts, auxiliary channels and motion sensors.
It takes a `where`, a device `class` and an explicit `who`:

```yaml
  binary_sensor:
    presenza_corridoio:
      who: "1"
      where: "1015"
      name: "Presenza Corridoio"
      class: motion
      timeout: 75
```

- `who: "25"` (default) — dry contact
- `who: "9"` — auxiliary channel
- `who: "1"` — motion sensor; only the `motion` class is supported, since it
  is the only WHO 1 combination that produces an entity
- `timeout` (optional, 5-3600 seconds) — how long a motion entity stays `on`
  after the last detection

### Motion timeout

A motion entity is switched back to `off` by a timer, not by a bus message.
The integration asks the device for its own timeout and, when the device
answers, uses that value plus 15s. A sensor driving a **dedicated (unused)
address** never answers that query — nothing lives at that address — so the
entity would fall back to the built-in default (300s) no matter what timer is
configured on the sensor itself.

Set `timeout` explicitly in that case: a configured value always wins over
the one reported by the bus. For a sensor set to 1 minute, `timeout: 75`
(60s plus a small margin) mirrors the device behaviour.

A MyHome PIR/presence sensor configured in "local" mode toward an unused
address becomes a usable motion entity this way: it broadcasts WHAT `34`
(motion detected) on that address, which is decoded as motion. Note that the
same address must **not** also be configured as a `light`: WHAT `34` is not a
switch-on, so a light entity on it would never turn on.

Since **1.1.4** all of this is configurable from the web panel (platform
`binary_sensor`), with WHO and device-class selectors.

## Unique IDs and entity IDs

Entities are registered with `unique_id = {gateway_mac}-{who}-{where}`
(climate: `{gateway_mac}-4-{zone}`). The integration never forces entity_ids
or names: Home Assistant derives them once and preserves them in the entity
registry.

**Warning:** for `sensor`/`binary_sensor`, the configured `class` is part of
the unique_id. Changing the device class of an existing sensor in the
configuration will orphan the old entity and create a new one (losing history
and the `_2`-free entity_id). Avoid changing it on entities you rely on.

## Options

From `Settings -> Devices & Services -> bticino MyHome -> Configure`:

- **Name** — config entry title.
- **Address** — gateway IP.
- **Password** — gateway OPEN password. Leave empty to keep the current one.
- **Worker count** — parallel command sessions (1-10, default 1).
- **Generate events** — republish raw OWN messages on the HA event bus as
  `myhome_message_event` (useful for automations on CEN/CEN+ or diagnostics).
  Note: events include the raw message and the gateway host; any recorder or
  event listener can read them.
