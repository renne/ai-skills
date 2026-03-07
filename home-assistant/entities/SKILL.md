---
name: entities
description: Home Assistant entity registry and device registry covering entity IDs, naming, entity_category, disabling, hidden entities, device management, renaming, and the entity state model. Use when managing entity or device names, disabling or hiding entities, understanding entity attributes, querying entity state, diagnosing why sensors don't appear on the dashboard, or reorganizing the device registry.
---
# Home Assistant Entities & Device Registry

Source: https://www.home-assistant.io/docs/configuration/

## Overview

Home Assistant has two registries:

| Registry | What it tracks |
|----------|----------------|
| **Entity Registry** | Every entity: ID, name, icon, area, platform, disabled/hidden state |
| **Device Registry** | Physical or virtual devices: model, manufacturer, firmware, linked entities |

Every sensor, switch, light, and helper is an **entity**. Entities are created by integrations and stored in the entity registry. Multiple entities can belong to one device (e.g., a Zigbee plug may expose a `switch`, a `sensor` for power, and another `sensor` for energy).

---

## Entity IDs

Every entity has a unique ID in the form `<domain>.<object_id>`:

| Domain | Examples |
|--------|---------|
| `light` | `light.living_room`, `light.bedroom_lamp` |
| `switch` | `switch.coffee_maker`, `switch.garden_pump` |
| `sensor` | `sensor.outdoor_temperature`, `sensor.door_battery` |
| `binary_sensor` | `binary_sensor.front_door`, `binary_sensor.motion_hallway` |
| `climate` | `climate.downstairs` |
| `media_player` | `media_player.living_room_speaker` |
| `person` | `person.alice` |
| `input_boolean` | `input_boolean.guest_mode` |

The object ID portion is derived from the integration's slug and the entity's friendly name. It can be changed in the UI or via `ha_rename_entity`.

---

## Entity State Model

Every entity has:

- **state** — the primary value (e.g., `"on"`, `"23.5"`, `"home"`)
- **attributes** — a dict of additional values (e.g., `brightness`, `unit_of_measurement`, `friendly_name`)
- **last_changed** — when state last changed
- **last_updated** — when state was last written (even without change)

### Reading state in templates

```yaml
# State value
"{{ states('sensor.outdoor_temperature') }}"

# State as float
"{{ states('sensor.outdoor_temperature') | float }}"

# Attribute value
"{{ state_attr('light.living_room', 'brightness') }}"

# Check state
"{{ is_state('binary_sensor.front_door', 'on') }}"

# Check attribute
"{{ is_state_attr('light.living_room', 'color_mode', 'color_temp') }}"
```

### Getting state via ha-mcp

Use `ha_get_state(entity_id)` to retrieve the full state object including all attributes.

---

## Renaming Entities

Entities can be renamed without changing the underlying integration:

- **UI:** Settings → Devices → click entity → pencil icon → change Name.
- **Tool:** `ha_rename_entity(entity_id, new_name)` — changes the friendly name stored in the entity registry.

> Renaming via the UI or API changes the **friendly name** only. The entity ID (`light.old_name`) is **not** automatically updated. To change the entity ID, use **Settings → Devices → entity → pencil icon → Entity ID field**.

---

## Entity Category

The `entity_category` attribute classifies entities by their role. It affects visibility in the auto-generated Overview dashboard and entity pickers.

| Value | Meaning | Dashboard visible? |
|-------|---------|-------------------|
| `null` (none) | Primary entity — main control or measurement | ✅ Yes |
| `config` | Configuration parameter (e.g., a setting slider) | ❌ No |
| `diagnostic` | Technical/informational data (e.g., signal strength, firmware version, electrical measurements) | ❌ No |

### Impact on the auto-generated Overview dashboard

The default Overview dashboard uses the `original-states` strategy, which **excludes any entity where `entity_category` is non-null**. This means:

- Switches, lights, and primary controls (`entity_category: null`) appear as tiles.
- Sensors classified as `diagnostic` by their integration are hidden — even if they have useful data like power consumption, voltage, or current.

**Common gotcha — Matter electrical measurement sensors:**  
The Matter integration assigns `entity_category = diagnostic` to all electrical measurement sensors (power, voltage, current, energy). These sensors will NOT appear on the auto-generated dashboard even though they have live data and are fully enabled. The device page in Settings → Devices shows them, but the Overview dashboard does not.

### Fixing missing sensors on the auto-generated dashboard

**Option 1 — Clear the entity category (persistent fix):**

1. Go to **Settings → Devices & Services → [Integration] → [Device]**
2. Click the sensor entity → pencil icon
3. Set "Entity category" to *(none)* → Save

The entity will now appear on the auto-generated dashboard.

**Option 2 — Add a manual card to the dashboard:**

Edit the Overview dashboard → Add Card → Entities or Tile card → manually add the sensor entity IDs.

**Option 3 — Use the Energy dashboard:**

For power/energy sensors with `device_class: power` or `device_class: energy`, use the dedicated Energy dashboard (Settings → Energy), which shows them regardless of `entity_category`.

### Checking entity_category via ha-mcp

```python
# Get entity registry entry to check category
entity = ha_get_entity(entity_id)
print(entity["entity_category"])  # null, "config", or "diagnostic"
```

---

## Disabling and Hiding Entities

### Disabled entities

A disabled entity does not appear in the UI and does not update its state. The integration still tracks it, but no state is published.

- **When to disable:** Entities from an integration you never use (e.g., redundant diagnostic sensors).
- **UI:** Settings → Devices → entity → Enable/Disable toggle.

### Hidden entities

A hidden entity updates normally but does not appear in the default entity picker. It is still usable in automations and templates.

- **When to hide:** Low-level or helper entities you want available but out of the way.
- **UI:** Settings → Devices → entity → visibility setting.

---

## Device Registry

Devices group related entities from the same physical or logical unit.

### Viewing devices

**UI:** Settings → Devices & Services → Devices

Each device shows:
- Manufacturer, model, firmware version
- Integration that created it
- All associated entities
- Area assignment

### Managing devices via ha-mcp

| Tool | Purpose |
|------|---------|
| `ha_get_device(device_id)` | Get full device details |
| `ha_update_device(device_id, ...)` | Rename device, assign area, add labels |
| `ha_remove_device(device_id)` | Remove a device (and optionally its entities) |
| `ha_rename_entity(entity_id, ...)` | Rename an entity's friendly name |

### Searching for entities

| Tool | Purpose |
|------|---------|
| `ha_search_entities(query)` | Fuzzy name search across all entities |
| `ha_get_state(entity_id)` | Get state + attributes for a specific entity |
| `ha_get_overview()` | High-level summary of all domains and entity counts |

---

## Entity Domains and Services

Each domain exposes domain-specific services. Use `ha_list_services(domain)` to discover available services.

### Common service patterns

```yaml
# Generic on/off (works for light, switch, fan, etc.)
- action: homeassistant.turn_on
  target:
    entity_id: light.living_room

- action: homeassistant.turn_off
  target:
    entity_id: switch.garden_pump

- action: homeassistant.toggle
  target:
    entity_id: fan.bedroom
```

### Targeting multiple entities

```yaml
# List of entity IDs
target:
  entity_id:
    - light.living_room
    - light.hallway

# By area (all entities in the area)
target:
  area_id: kitchen

# By device
target:
  device_id: abc123def456

# By label
target:
  label_id: vacation_mode
```

---

## Entity Naming Best Practices

- Use the **room or location** as a prefix where it helps avoid ambiguity: `Kitchen Lights`, not just `Lights`.
- Keep names short enough to be natural in voice commands: `Front Door Lock`, not `Z-Wave Node 5 Lock Deadbolt`.
- Assign every device to an **area** so voice control and automations can use room-level targeting.
- Avoid encoding the entity domain in the name — HA already shows the domain icon.

---

## Diagnosing Entity Issues

### Entity is unavailable

An `unavailable` state means the integration cannot reach the device:

1. Check the device is powered and on the network.
2. Check the integration's logs: **Settings → System → Logs**.
3. Reload the integration: **Settings → Devices & Services → integration → Reload**.

### Entity shows unexpected state

1. Open **Developer Tools → States** and search for the entity.
2. Check **last_changed** and **last_updated** timestamps.
3. Use `ha_get_history(entity_id)` to review historical states.
4. Check **Developer Tools → Logbook** for recent state change events.

### Finding what integration created an entity

Use `ha_get_entity_integration_source(entity_id)` to identify the source integration and its configuration entry.

---

## Tips

- Use **Developer Tools → States** to inspect any entity's live state and attributes.
- Use `ha_deep_search(query)` to search across entities, automations, and scripts simultaneously.
- Assign devices to areas as soon as they are added — it powers voice control and area-based automation targets.
- Disable diagnostic entities you never need to reduce clutter and improve performance.

## References

- [Entity Registry](https://www.home-assistant.io/docs/configuration/customizing-devices/)
- [Device Registry](https://developers.home-assistant.io/docs/device_registry_index/)
- [Entity States](https://www.home-assistant.io/docs/configuration/state_object/)
- [Targeting Entities](https://www.home-assistant.io/docs/scripts/service-calls/#targeting-areas-and-devices)
- [Developer Tools](https://www.home-assistant.io/docs/tools/dev-tools/)
