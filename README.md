# paoloantinori/zmk

A [ZMK Firmware](https://zmk.dev/) fork carrying Corne-ish Zen display patches and a layer state relay feature, rebased on top of upstream ZMK main (Zephyr 4.1).

> **Default branch:** `zen-v1+v2-rebased` — this is the branch used by [zmk-config-zen-2](https://github.com/paoloantinori/zmk-config-zen-2) via `west.yml`.

## Relationship to upstream

This repo tracks [`zmkfirmware/zmk`](https://github.com/zmkfirmware/zmk) `main`. The `zen-v1+v2-rebased` branch sits on top of upstream with two additional commits:

```
upstream/main  26246da1  chore(deps): bump actions/stale ...
       │
       ├── fe58a8ba  Zen display patches (squashed)
       │
       └── abb8daf5  Layer state relay (central → peripheral)
```

The branch is regularly rebased onto the latest upstream to pick up new fixes and features.

## Patches

### 1. Corne-ish Zen Display Patches

Carries forward non-merged patches from [caksoylar's Zen branch](https://github.com/caksoylar/zmk/tree/zen-v1+v2), adapted for HWMv2 board paths (`arm/` → `lowprokb/`) and LVGL/Zephyr 4.1 API updates.

| Feature | Config option | Description |
|---------|--------------|-------------|
| IL0323 display invert | `CONFIG_IL0323_INVERT` | Invert e-ink display (white-on-black) |
| IL0323 alt partial refresh | `CONFIG_IL0323_ALTERNATIVE_REFRESH` | Alternative partial refresh mode for display driver |
| Display full refresh timer | `CONFIG_ZMK_DISPLAY_FULL_REFRESH_PERIOD` | Periodic full refresh to clear e-ink ghosting |
| Momentary layer tracking | `CONFIG_ZMK_TRACK_MOMENTARY_LAYERS` | Track layers activated via `&mo`/`&lt` (upstream only tracks `&tog`) |
| Hide momentary layers | `CONFIG_ZMK_DISPLAY_HIDE_MOMENTARY_LAYERS` | Hide momentary layers in Zen layer widget to reduce flicker |
| Battery widget tweaks | *(always active)* | Skip updates when unchanged; adjusted thresholds (87/62/37/12/5) |
| Custom status screen | `CONFIG_CUSTOM_WIDGET_LOGO_IMAGE_*` | Rearranged widget layout; selectable logos (ZEN/LPKB/ZMK/MIRYOKU) |
| Conditional layer fixup | *(always active)* | Propagate momentary state through conditional layer chains |

**Files touched:** `app/boards/lowprokb/corneish_zen/`, `app/module/drivers/display/il0323.c`, `app/src/keymap.c`, `app/src/display/`, `app/src/behaviors/`, `app/src/conditional_layer.c`

### 2. Layer State Relay (central → peripheral)

In a ZMK split keyboard, the **left half (central)** holds the keymap and tracks which layers are active. The **right half (peripheral)** only knows about keypresses — it has no idea what layer is active. This means the right half's e-ink display can only show static content, because it cannot react to layer changes happening on the left.

This patch fixes that by opening a **one-way data channel** from central to peripheral over the existing BLE connection:

```
Left half (central)                         Right half (peripheral)
┌─────────────────────┐                     ┌─────────────────────┐
│  User taps layer key │                     │                     │
│         ↓            │                     │                     │
│  zmk_layer_state_    │                     │                     │
│  changed event fires │                     │                     │
│         ↓            │                     │                     │
│  Listener computes   │                     │                     │
│  highest active      │   BLE GATT write   │  GATT write handler │
│  layer = 3 (FUNC)  ──┼────────────────────►│  receives "3"       │
│         ↓            │   (1 byte payload)  │         ↓           │
│  Change detection:   │                     │  Raises event:      │
│  only sends if       │                     │  split_peripheral_  │
│  layer actually      │                     │  layer_changed      │
│  changed              │                     │         ↓           │
└─────────────────────┘                     │  E-ink display      │
                                            │  updates to show    │
                                            │  "FUNC"             │
                                            └─────────────────────┘
```

**The practical result:** When you activate a layer on the left half (e.g. hold a NAV key, toggle MOUSE mode, or activate NUM via a combo), the right half's e-ink display updates to show the layer name in real time. Previously it could only show battery level and BT status — now it reflects what's actually happening on the keyboard.

This also establishes a reusable pattern for future cross-half communication. Any state on the central that the peripheral needs to know about can follow the same relay architecture.

#### Technical details

Enabled by `CONFIG_ZMK_SPLIT_PERIPHERAL_LAYER_STATE=y`. Follows ZMK's existing [HID indicators relay](https://zmk.dev/docs/features/split-keyboards) pattern (same code structure, new UUID/data type).

**How it works, step by step:**

1. **Central** subscribes to `zmk_layer_state_changed` events (fires on any layer activate/deactivate)
2. On each event, computes `zmk_keymap_highest_layer_active()` — the single highest layer index
3. Compares against last-sent value; skips if unchanged (avoids redundant BLE writes on multi-layer transitions)
4. Sends `SET_LAYER_STATE` command through the transport layer to all connected peripherals
5. **Central BLE** (`central.c`) writes 1 byte to the peripheral's GATT characteristic via `bt_gatt_write_without_response()`
6. **Peripheral BLE** (`service.c`) receives the write, raises `zmk_split_peripheral_layer_changed` event via deferred work queue
7. Peripheral-side widgets (e.g. the Zen `layer_status` widget) subscribe to this event and update the display

**Optimizations:**
- Change detection on both sides — central only sends when highest layer actually changes, peripheral only raises event when received value differs
- Direct byte assignment for the single-byte payload

**Known limitations:**
- No initial sync on reconnect — peripheral defaults to layer 0 (BASE) until the next layer change. Since users are on BASE 99% of the time, this is acceptable.
- Layer names on the peripheral are hardcoded (the peripheral has no access to devicetree keymap labels). If layer names change in the keymap, the peripheral widget must be updated to match.

**Files touched:** `app/src/split/bluetooth/service.c`, `app/src/split/bluetooth/central.c`, `app/src/split/central.c`, `app/include/zmk/split/bluetooth/uuid.h`, `app/include/zmk/split/transport/types.h`, `app/include/zmk/split/central.h`, `app/include/zmk/events/split_peripheral_layer_changed.h`, `app/src/events/split_peripheral_layer_changed.c`, `app/src/split/Kconfig`, `app/CMakeLists.txt`

## Updating the branch

```bash
git remote add upstream https://github.com/zmkfirmware/zmk.git  # once
git fetch upstream
git rebase upstream/main zen-v1+v2-rebased
git push --force-with-lease
```

## Credits

- [ZMK Contributors](https://zmk.dev/) — the upstream firmware
- [caksoylar](https://github.com/caksoylar) — original Corne-ish Zen display patches
- [paoloantinori](https://github.com/paoloantinori) — rebased patches, layer state relay feature
