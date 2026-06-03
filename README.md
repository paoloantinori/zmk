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

Relays the highest active layer index from the central half to the peripheral half via a new BLE GATT characteristic. This allows the peripheral's e-ink display to show the active layer name without any direct access to the keymap state.

Follows ZMK's existing [HID indicators relay](https://zmk.dev/docs/features/split-keyboards) pattern exactly.

| Component | Change |
|-----------|--------|
| **UUID** | `ZMK_SPLIT_BT_UPDATE_LAYER_STATE_UUID` (0x00000007) |
| **Transport** | `SET_LAYER_STATE` command type, `uint8_t layer` payload |
| **Event** | `zmk_split_peripheral_layer_changed` (raised on peripheral) |
| **Kconfig** | `CONFIG_ZMK_SPLIT_PERIPHERAL_LAYER_STATE` |
| **Peripheral** | GATT write handler in `service.c`, raises event via deferred work |
| **Central (BLE)** | GATT discovery + `bt_gatt_write_without_response` dispatch in `central.c` |
| **Central (transport)** | `ZMK_LISTENER` on `zmk_layer_state_changed` in `split/central.c` |

**Optimizations:**
- Change detection on both sides — central only sends when highest layer actually changes, peripheral only raises event when received value differs
- Direct byte assignment (no memcpy) for the single-byte payload

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
