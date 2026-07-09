<p align="left">
  <img src="custom_components/myhome/frontend/bticino-logo.svg" alt="bticino logo" width="180" />
</p>

# bticino MyHome — Home Assistant Integration

A modern, actively maintained Home Assistant integration for BTicino / Legrand
**MyHome** systems. It talks to your gateway over the **OpenWebNet** (SCS)
protocol on the local network — no cloud, no polling — and receives live push
updates directly from the bus.

Control and monitor lights, shutters/blinds, thermostats and heating zones,
energy metering, switches and dry contacts, all from Home Assistant.

## A modernized successor to `anotherjulien/MyHOME`

The main focus of this project is to be the **drop-in, modernized replacement**
for the popular but now-unmaintained
[`anotherjulien/MyHOME`](https://github.com/anotherjulien/MyHOME) integration.

If you already run MyHOME you can migrate **in place, without losing anything**:
the `myhome` domain and the `{mac}-{who}-{where}` `unique_id` scheme are
unchanged, so Home Assistant reuses your existing entities. Your entity_ids,
friendly names, dashboards, automations and history are preserved — no `_2`
suffixes, no rebuilding.

→ Full steps in the [migration guide](docs/migration-from-anotherjulien.md).

## Strong points

- **Seamless migration** — same domain and unique_id scheme as
  `anotherjulien/MyHOME`; entities, dashboards and automations keep working
  untouched.
- **Rock-solid connection** — a persistent event session for push updates plus
  command sessions, with single-point reconnection, exponential backoff, no
  busy-loops and no socket/session leaks. TCP keepalive detects silently
  dropped connections and reconnects on its own, so states don't freeze.
- **Web UI configuration** — discover, import, add and remove devices from a
  built-in panel; no need to hand-write and maintain YAML.
- **YAML optional, not mandatory** — a legacy `myhome.yaml` is imported **once**
  into integration storage and then managed from the panel (the original
  re-read the YAML at every boot).
- **Broad device support** — `light`, `cover`, `climate`, `sensor`, `switch`
  and `binary_sensor`, including power/energy endpoints (`WHO 18`) and
  thermoregulation zones.
- **Active and passive discovery** — devices are found both on demand and
  passively, by observing activity on the bus.
- **Hardened OWNd** — the vendored OWNd library ships with additional
  resilience fixes (reconnection, timeouts, keepalive, safer message parsing).
- **Fully local push** — `iot_class: local_push`; everything runs on your LAN.

## What it does (overview)

The integration connects Home Assistant to a MyHome/OpenWebNet gateway (for
example F454, F455, MH201/MH202, MyHOMEServer1 and similar) and exposes your
bus devices as native Home Assistant entities. State changes are pushed from
the gateway in real time, and commands are sent back over dedicated command
sessions. Setup is done entirely through the Home Assistant Config Flow and
the integration's web panel.

Supported platforms: `light`, `cover`, `climate`, `sensor`, `switch`,
`binary_sensor`. Custom MyHome services are available too (`sync_time`,
`send_message`, and discovery services).

## Documentation

- [Configuration guide](docs/configuration.md) — YAML schema, web panel, config storage
- [Migrating from anotherjulien/MyHOME](docs/migration-from-anotherjulien.md) — safe in-place upgrade, entity_ids preserved
- [Troubleshooting](docs/troubleshooting.md) — connection behavior, logs, common errors

## Requirements

- Home Assistant (Core or Container)
- IP connectivity from Home Assistant to your MyHome gateway
- Your gateway's OPEN password (numeric OPEN algorithm or alphanumeric HMAC)

## Installation

### HACS (recommended)

Prerequisite: HACS must already be installed.

1. Open Home Assistant and go to `HACS`.
2. Open the top-right menu (`⋮`) and choose `Custom repositories`.
3. Add:
   - Repository: `https://github.com/St3f4n1n0/bticino-myhome-hacs-integration`
   - Category: `Integration`
4. Click `Add`.
5. Search for `bticino MyHome` in HACS and open the integration page.
6. Click `Download` and complete installation.
7. Restart Home Assistant.
8. Go to `Settings` -> `Devices & Services` -> `Add Integration`.
9. Search for `bticino MyHome` and complete the config flow.

Optional direct link:

`https://my.home-assistant.io/redirect/hacs_repository/?owner=St3f4n1n0&repository=bticino-myhome-hacs-integration&category=integration`

### Manual

1. Copy `custom_components/myhome` to `config/custom_components/myhome`.
2. Restart Home Assistant.
3. Add and configure `bticino MyHome` from `Settings` -> `Devices & Services`.

## Device configuration

Use the integration web panel to manage devices:

- automatic discovery by activation (passive collection)
- import discovered devices into configuration
- manual device creation and deletion

See the [configuration guide](docs/configuration.md) for details.

## Migrating from `anotherjulien/MyHOME`

The upgrade is designed to preserve all your entity_ids, names, dashboards and
automations. Back up first, swap the `custom_components/myhome` folder, restart,
and verify no `_2` suffixes appear. Use version **1.1.1 or later**. Full steps
and caveats are in the [migration guide](docs/migration-from-anotherjulien.md).

## Credits

This project builds on the work of the original integration and its OWNd
library:

- https://github.com/anotherjulien/MyHOME

Thanks to the original maintainers and contributors.

## License

See `LICENSE`.
