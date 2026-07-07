# Migrating from anotherjulien/MyHOME

This integration is designed as a **drop-in, in-place upgrade** for
installations using the unmaintained
[anotherjulien/MyHOME](https://github.com/anotherjulien/MyHOME) integration.

## Why it is safe

An integration swap preserves your entities if and only if:

1. the **domain** is the same (`myhome`), and
2. the **unique_id scheme** is the same (`{mac}-{who}-{where}`), and
3. entity_ids/names are **not forced** by the code.

This fork satisfies all three: Home Assistant matches the existing entity
registry entries and reuses them. Your entity_ids, friendly names, dashboards,
automations and history are preserved. No YAML changes are needed: the same
`myhome.yaml` schema is supported and imported.

## Upgrade procedure

1. **Backup** Home Assistant, including `.storage/core.entity_registry`,
   `.storage/core.device_registry` and your `myhome.yaml`.
2. Remove the old `custom_components/myhome` (or uninstall via HACS).
3. Install this integration (HACS custom repository or manual copy) — the
   folder is again `custom_components/myhome`.
4. Restart Home Assistant.
5. Verify: entities must keep their entity_id, with **no `_2` suffixes**
   appearing. If a `_2` suffix appears, the unique_id did not match — restore
   the backup and open an issue.

Testing on a separate instance with a copy of your `.storage` before touching
production is strongly recommended.

## Differences to be aware of

- Configuration is imported from YAML **once** into `.storage/myhome_config`,
  then managed from the web panel (the original re-read the YAML at each
  boot). See the [configuration guide](configuration.md).
- The vendored OWNd library includes additional resilience fixes
  (reconnection with exponential backoff, no socket leaks, timeouts).
