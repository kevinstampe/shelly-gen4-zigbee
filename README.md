# Shelly Gen4 Zigbee

ESPHome firmware that runs Shelly Gen4 relays and meters as **Zigbee routers**, for
Zigbee2MQTT or ZHA. No Wi-Fi, no cloud.

The pin maps, meter calibration and stock flash layout come from
[automatous-io/shelly-gen4-esphome](https://github.com/automatous-io/shelly-gen4-esphome),
which runs the same hardware on ESPHome over Wi-Fi. This repository swaps the network side for
ESPHome's [Zigbee component](https://esphome.io/components/zigbee/).

> **⚠️ Disclaimer.** Third-party firmware voids your Shelly warranty and can brick the device.
> Never connect a UART adapter while the Shelly is on mains. Back up the stock firmware over
> UART before flashing; the web UI install cannot make a backup. You flash at your own risk.

## Supported devices

| Model | Config | Exposed over Zigbee | Status |
|---|---|---|---|
| Shelly 1 Gen4 | [`shelly-1-gen4-zigbee.yaml`](shelly-1-gen4-zigbee.yaml) | relay, switch input, temperature (chip), fault | builds, untested |
| Shelly 1 Mini Gen4 | [`shelly-1-mini-gen4-zigbee.yaml`](shelly-1-mini-gen4-zigbee.yaml) | relay, switch input, temperature, fault | builds, untested |
| Shelly 1PM Gen4 | [`shelly-1pm-gen4-zigbee.yaml`](shelly-1pm-gen4-zigbee.yaml) | relay, switch input, voltage, current, power, energy, temperature, fault | tested (UART) |
| Shelly 1PM Mini Gen4 | [`shelly-1pm-mini-gen4-zigbee.yaml`](shelly-1pm-mini-gen4-zigbee.yaml) | as 1PM | tested (web UI) |
| Shelly 2PM Gen4 | [`shelly-2pm-gen4-zigbee.yaml`](shelly-2pm-gen4-zigbee.yaml) | 2 relays, 2 switch inputs, per-channel current, power, energy, voltage, temperature, 2 faults | tested (UART) |
| Shelly EM Mini Gen4 | [`shelly-em-mini-gen4-zigbee.yaml`](shelly-em-mini-gen4-zigbee.yaml) | voltage, current, power, energy (CT clamp), temperature | builds, untested |

"Tested" means flashed over UART, paired with Zigbee2MQTT, and switched on the bench, using an
earlier revision of these configs. The 1PM Mini Gen4 was installed through the stock web UI from
the zip and paired. The other web UI zips are untested. Reports for the untested models are welcome.

## Behavior

- **Relays** use the Zigbee on/off cluster with the outlet device type, one endpoint per relay
  (endpoint 1, and endpoint 2 on the 2PM), so they can be added to Zigbee groups.
- **Sensors and inputs** are generic analog/binary input clusters, sent on every change.
- **Switch inputs** toggle their relay. **The onboard button** toggles relay 1 on a short press;
  a 5 s hold leaves the Zigbee network and starts pairing again (the only action on the EM Mini).
- **Protections**: two readings in a row over a limit (power, current, voltage or temperature)
  turn the relay off and set its `Fault`. Turning the relay back on clears it. Limits are
  substitutions in each model file.
- **Mains frequency** defaults to 50 Hz. Set `line_frequency: "60Hz"` in North America.
- **Updates** need UART or the web UI zip from stock firmware; there is no over-the-air update.

## Building

Needs ESPHome 2026.5 or newer (Zigbee on ESP32 came in 2026.5). Developed with 2026.10.2.

```bash
pip install esphome            # or activate a venv that has it
./build.sh <model>             # UART image: <model>-zigbee-factory.bin
./build.sh <model> --webui     # web UI zip: webui/<model>-zigbee-webui.zip
```

Models: `1`, `1-mini`, `1pm`, `1pm-mini`, `2pm`, `em-mini`. `build.sh` also looks for a venv at
`../.venv`.

## Installing

### UART

Wiring, flash mode and the header pinout are covered in automatous-io's
[Flashing Guide](https://github.com/automatous-io/shelly-1-gen4-matter-thread/blob/main/docs/FLASHING.md).
In short: 3.3 V only, TX to RX, RX to TX, BOOT bridged to GND while powering up.

```bash
./build.sh 1pm --backup kitchen --flash
```

`--backup NAME` reads the full 8 MB chip to `../stock-<model>-<NAME>-<mac>.bin` and verifies it
before anything is written. `--flash` erases the chip and writes the image. `--port` overrides
`/dev/ttyUSB0`. Remove the BOOT bridge and power-cycle afterwards.

### Stock web UI

1. Check `slot` in the device info page (or `Shelly.GetDeviceInfo`). It **must be 1**. On slot 0,
   install any official Shelly update first. The reason is in automatous-io's
   [partition docs](https://github.com/automatous-io/shelly-gen4-esphome/blob/main/docs/PARTITIONS.md#the-slot-rule).
2. Upload `webui/<model>-zigbee-webui.zip` as a firmware file.
3. A stall at 87% means slot 0: nothing was written, go back to step 1.

The zips keep Shelly's stock flash layout, which the installer requires. Zigbee stores its
network data in `nvs`; the `zb_fct` partition ESPHome requires is a placeholder in otherwise
unused flash. If the device does not boot afterwards, recovery is UART only.

## Pairing

1. Enable *Permit join* on the coordinator.
2. Hold the onboard button 5 s. A fresh flash may not retry joining on its own.
3. Pair close to the coordinator; ESPHome notes the ESP32-C6 radio can be marginal.
4. If sensors show `N/A` in Zigbee2MQTT, press *Reconfigure* or re-pair.

Remove and re-pair the device after changing its configuration, so the coordinator sees the
new endpoints.

## Layout

```
build.sh                    build, back up, flash, or package a web UI zip
shelly-*-zigbee.yaml        one file per model: pins, calibration, limits, blocks
common/
  zigbee-base.yaml          chip, logger, Zigbee router, status LED, default settings
  relay.yaml                relay, switch input, button, fault (single-relay models)
  button-reset.yaml         button with pairing reset only (EM Mini)
  meter-bl0942.yaml         BL0942 power meter
  temp-ntc.yaml             board NTC
  temp-internal.yaml        ESP32-C6 die temperature (1 Gen4)
  protect-temp.yaml         overheating trip
  protect-meter.yaml        overvoltage, overcurrent and overpower trip
  *.csv                     stock flash layouts for web UI builds
tools/
  make-esphome-ota-zip.py   web UI zip packager, from automatous-io
```

## License

Configs and scripts: Apache 2.0, see [LICENSE](LICENSE) and [NOTICE](NOTICE). Firmware images
built from this repository include ESPHome's GPLv3 runtime and are distributed under the GPLv3.

Not affiliated with Shelly / Allterco Robotics, ESPHome, or automatous-io.
