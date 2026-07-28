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
```

- `who: "25"` (default) — dry contact
- `who: "9"` — auxiliary channel
- `who: "1"` — motion sensor; only the `motion` class is supported, since it
  is the only WHO 1 combination that produces an entity

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
