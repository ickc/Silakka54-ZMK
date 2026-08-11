# Hardware notes

Board-specific findings for this particular Silakka54 build. Firmware-visible behaviour here does **not** always match the nice!nano documentation, because the controllers are not nice!nanos.

## Controller: nRF52840 SuperMini, not a nice!nano

The keyboard was bought pre-built and the listing advertises "nice!nano". The actual modules are **nRF52840 SuperMini** boards (also sold as "nRF52840 ProMicro"), a nice!nano clone with a *mix of v1 and v2 design features*.

The most useful reference is [sasodoma/nrf52840-promicro](https://github.com/sasodoma/nrf52840-promicro), which reverse-engineers the board.

This matters because the clone is close enough that everything builds and mostly works, so failures show up as wrong *values* rather than as errors.

## Build target

```yaml
board: nice_nano@2//zmk
```

Zephyr hardware-model-v2 syntax, `board[@revision]/soc[/variant]` — revision 2.0.0, empty SoC field (the board has only one), ZMK variant. See `AGENTS.md` in the parent repo for the full history of why this value has changed three times upstream.

**`@2` is not optional in spirit.** Revision 1.0.0 and 2.0.0 differ in `EXT_POWER` on P0.13 — `GPIO_ACTIVE_LOW` on v1, `GPIO_ACTIVE_HIGH` on v2 — and that pin gates the external VCC rail that powers the nice!view. Building as v1 leaves the display unpowered while the matrix, USB and BLE all work normally, because none of those sit on that rail. Symptom: **types and pairs fine, screen completely blank.**

To confirm a build got the revision you asked for, check the resolved config: 2.0.0 yields `CONFIG_ZMK_BATTERY_NRF_VDDH=y`; 1.0.0 yields the voltage-divider driver instead.

## Battery sensing is unreliable on this hardware

`CONFIG_ZMK_BATTERY_NRF_VDDH=y` (inherited from the nice!nano v2 revision) is the **correct and only workable choice** here.

The SuperMini has a documented design flaw: its schematic shows a battery voltage divider on **P0.04 (AIN2)**, but the hardware actually routes it to **P0.24**, which is unsuitable for the measurement. So the v1-style `zmk,battery-voltage-divider` approach cannot work — pointing the driver at AIN2 would read a pin that carries nothing. The teardown's own recommendation is to measure VDDH, which is what this config does.

Two consequences to expect, neither of them fixable in firmware:

1. **Reads ~100% whenever USB is connected.** VDDH is the *system* rail, and with USB present that rail is fed from USB rather than the cell. The teardown states it plainly: "the voltage will rise when USB is plugged in." Removing components NPQ2/NBD1/NPR7 and bridging pins 3–2 on NPQ2 is the only remedy offered, i.e. a hardware mod.
2. **Reads low the rest of the time.** See below.

### Suspected systematic under-read (unconfirmed)

Probing the top two pins of each controller gives, with the power switch **on**, roughly 3.6 V (left) and 3.3 V (right), and ~0.2 V with the switch **off**.

That last part is the important clue: **these are not the raw battery pads.** A cell wired to B+/B- still reads its own voltage with the switch off. Collapsing to 0.2 V means the probe point is downstream of the switch, so there is circuitry — the switch, and possibly a series diode — between the cell and both the probe point *and* VDDH.

If there is a forward drop in that path, ZMK under-reports by that drop, permanently. A full 4.2 V cell behind a ~0.6 V silicon diode presents as ~3.6 V, which ZMK maps to ~19%. That would explain a battery that reads near-empty no matter how long it charges. The teardown does note a series diode (W5) whose part varies by batch — regular silicon in some, BAT60B Schottky in others, with sleep current varying 4–60 µA as a result. Whether W5 sits in the sense path specifically is **not confirmed**.

**Test without disassembly:** run on battery from a "10%" indication and see how long it lasts. A 100 mAh cell genuinely at 10% dies quickly; if it runs for days, the reading is offset and the cells are fine.

### Reading the two halves separately

Both `CONFIG_ZMK_SPLIT_BLE_CENTRAL_BATTERY_LEVEL_FETCHING` and `..._PROXY` are enabled, which is what ZMK documents for reporting both sides. The host still shows one number, and that is expected — from ZMK's docs:

> Host support for multiple battery levels is undefined. It appears that in most of the cases only the main battery is being reported. In order to correctly display all the battery values, you probably need a special application or script.

The number macOS shows is the **central (left)** half. To read the peripheral: look at the right half's own nice!view, which shows its own level locally, or enumerate the GATT Battery Services with a BLE explorer.

These two settings change **nothing** on either display. nice_view builds `status.c` on the central and `peripheral_status.c` on the peripheral, and each subscribes only to its own `zmk_battery_state_changed`.

### Percentage ↔ voltage

ZMK's curve (`lithium_ion_mv_to_pct`) is linear between 3450 and 4200 mV:

```c
if (bat_mv >= 4200) return 100;
if (bat_mv <= 3450) return 0;
return bat_mv * 2 / 15 - 459;
```

Inverted: **mV = (pct + 459) × 7.5**. So 19% ⇒ ~3.59 V, 10% ⇒ ~3.52 V. Useful for sanity-checking a reading against a multimeter — and note the reported 19% matched the measured 3.6 V, so the ADC path itself is accurate; the question is only what voltage reaches it.

**Voltage cannot be displayed.** `zmk_battery_state_changed` carries only `state_of_charge`, with a literal `// TODO: Other battery channels` in the struct, so no widget can show volts without custom code.

## Charging

- Each half has its **own battery and own USB port**. Charging one does nothing for the other — both must be charged individually.
- Cells in this build are **100 mAh**, which is small.
- SuperMini charges at **~100 mA** by default, or **~300 mA** if the BOOST pad is bridged. The vendor states BOOST should only be connected for cells above 500 mAh. At 100 mAh, 300 mA would be 3C — verify that pad is not bridged.
- Charge termination is done in hardware by the charge IC. **Firmware has no involvement in charging** — ZMK only reads a voltage, so a wrong percentage is not evidence of a charging fault and cannot cause overcharge.
- If charging appears not to work, try with the power switch **on**: the switch sits in the battery path (see above), so it may gate the charge path too.

## Numeric battery percentage on the nice!view

Not available with the stock nice!view screen. `CONFIG_ZMK_WIDGET_BATTERY_STATUS_SHOW_PERCENTAGE` — already present in `lily58.conf` — is **inert**, because it belongs to ZMK's *built-in* status screen widget while `nice_view/Kconfig.defconfig` defaults the screen to `ZMK_DISPLAY_STATUS_SCREEN_CUSTOM`.

To get numbers, override the choice to `CONFIG_ZMK_DISPLAY_STATUS_SCREEN_BUILT_IN=y` and drop the redundant `..._CUSTOM=y` line. The cost is the nice!view's custom artwork and layout. This only became viable recently: upstream #3243 fixed built-in widgets failing to render on nice!view from insufficient LVGL memory.

## Display

nice!view driven through the `nice_view_adapter` shield:

```yaml
shield: lily58_left nice_view_adapter nice_view
```

The adapter supplies the `nice_view_spi` node. If a ZMK upgrade renames the board target without `build.yaml` following, the adapter's board overlay stops matching and the build fails with `undefined node label 'nice_view_spi'` — that is a board-name problem, not a display problem.
