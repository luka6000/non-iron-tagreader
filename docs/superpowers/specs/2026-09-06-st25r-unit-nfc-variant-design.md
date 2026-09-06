# M5Stack Unit NFC (ST25R3916) firmware variant

## Goal

Add a second hardware option to this project: the [M5Stack Unit NFC](https://docs.m5stack.com/en/unit/Unit_NFC) (ST25R3916-AQWT), alongside the existing [Unit RFID2](https://docs.m5stack.com/en/unit/rfid2) (WS1850S) support. Both units are Grove I2C, so they plug into the same Atom Lite the project already targets.

ESPHome has no native ST25R component ([confirmed against esphome.io/components](https://esphome.io/components/index.html) — only PN7150/PN7160/PN532/RC522 are built in), so this uses [JohnMcLear/esphome_st25r](https://github.com/JohnMcLear/esphome_st25r) as an `external_components` source.

## Hardware facts

- M5Stack Unit NFC: ST25R3916-AQWT, I2C only, default address `0x50`, Grove 4-pin (GND/5V/SDA/SCL) — **no IRQ or reset line exposed**.
- `st25r_i2c` (from esphome_st25r) defaults to address `0x50` and treats `irq_pin`/`reset_pin` as optional, falling back to register polling — matches this hardware exactly.
- `st25r_i2c`'s `on_tag` / `on_tag_removed` triggers hand back a `std::string` UID, same shape as `rc522_i2c`, so the existing binary_sensor/text_sensor/script/`homeassistant.tag_scanned` block in `non-iron-tagreader-core-nfc-rc522.yaml` ports over with only the driver block changed.

## Files

### `non-iron-tagreader-core-nfc-st25r.yaml`

Copy of `non-iron-tagreader-core-nfc-rc522.yaml`. Same `binary_sensor`/`script`/`text_sensor`/`button` block, unchanged. The driver block becomes:

```yaml
external_components:
  - source:
      type: git
      url: https://github.com/JohnMcLear/esphome_st25r
      ref: 35d2318c1c419ea61dd49ff82e873322ccad022d
    refresh: never
    components: [st25r, st25r_i2c]

st25r_i2c:
  id: nfc_board
  i2c_id: bus_i2c
  address: 0x50
  update_interval: 350ms
  aat_enabled: false
  on_tag_removed:
    - binary_sensor.template.publish: { id: is_tag, state: false }
    - text_sensor.template.publish: { id: status, state: "Tag removed" }
    - script.execute: { id: tagreader_event, action: "tagRemoved", uid: !lambda 'return x;' }
    - script.execute: led_blink
  on_tag:
    - binary_sensor.template.publish: { id: is_tag, state: true }
    - text_sensor.template.publish: { id: status, state: !lambda 'return "Tag "+x;' }
    - text_sensor.template.publish: { id: status, state: "UID tag" }
    - homeassistant.tag_scanned: !lambda 'return x;'
    - script.execute: { id: tagreader_event, action: "tagScanned", uid: !lambda 'return x;' }
    - script.execute: led_ok
```

Pinned to a specific upstream commit (no tagged releases exist yet) with `refresh: never` rather than tracking `main`, so an unrelated upstream change can't silently change our build. The commit to bump to, and why, goes in the PR description — not a YAML comment, matching this project's existing no-comments style in the core-nfc files.

`aat_enabled: false`: Automatic Antenna Tuning needs a varicap network the M5Stack Grove unit doesn't have; leaving it on just adds startup latency for no benefit.

Everything else (`nfcv_enabled`, `nfcb_enabled`, health-check) stays at the component's defaults — the ST25R3916 and the WS1850S it replaces are both multi-protocol, and the health-check bug noted in the upstream repo's `AGENTS.md` is already fixed in the current `st25r.cpp` (`chip_type = ic_identity & 0xF8`), confirmed by reading the source directly rather than trusting the stale doc.

### `non-iron-tagreader-atom-lite-st25r.yaml`

Copy of `non-iron-tagreader-atom-lite.yaml`. Only change: the `core-nfc:` package line points at `non-iron-tagreader-core-nfc-st25r.yaml` instead of the rc522 file. `non-iron-tagreader-atom-lite.yaml` itself is untouched — no risk to the existing rc522 variant or anything referencing it by filename.

### `non-iron-tagreader-st25r.factory.yaml`

Copy of `non-iron-tagreader.factory.yaml`:
- `substitutions.name`: `non-iron-tagreader-st25r`, `friendly_name`: `Non-Iron TagReader (ST25R)`
- `packages.core`: `!include non-iron-tagreader-atom-lite-st25r.yaml`
- `dashboard_import.package_import_url`: `github://modem7/non-iron-tagreader-st25r/non-iron-tagreader-st25r.factory.yaml@main`
- `update.source`: `https://github.com/modem7/non-iron-tagreader-st25r/releases/latest/download/non-iron-tagreader-st25r-firmware.manifest.json` — a GitHub Release permalink instead of a Pages URL, since GitHub Pages is explicitly out of scope for this fork. Works once `publish-firmware.yml` cuts a release; 404s harmlessly until then.
- `esp32_ble.name`: distinct BLE name so it doesn't collide with the rc522 build during improv pairing.

### `non-iron-tagreader-st25r-custom.yaml`

New top-level entry point for technical users who want to compile locally with their own secrets instead of flashing the pre-built factory binary and pairing over BLE improv:

```yaml
substitutions:
  name: "non-iron-tagreader-st25r"
  friendly_name: "Non-Iron TagReader (ST25R)"

packages:
  project: !include non-iron-tagreader-project.yaml
  core: !include non-iron-tagreader-atom-lite-st25r.yaml

esphome:
  name: ${name}
  friendly_name: ${friendly_name}

logger:
  level: DEBUG
  logs:
    component: ERROR

api:
  encryption:
    key: !secret api_password

ota:
  platform: esphome
  password: !secret ota_password

wifi:
  ssid: !secret wifi_ssid
  password: !secret wifi_password
  domain: .home
```

This packages the same `atom-lite-st25r` stack (which still includes the default `non-iron-tagreader-core-comms.yaml`, unchanged), then merges the user's `logger`/`api`/`ota`/`wifi` block on top at the outer level. ESPHome's package merge combines dict keys rather than replacing whole blocks, so the resulting `wifi:` keeps `core-comms.yaml`'s fallback `ap:` *and* gains the primary `ssid`/`password`; `ota:` keeps `platform: esphome` and gains `password`; `logger:` keeps `logs.light: WARN` and gains `logs.component: ERROR` alongside the overridden `level: DEBUG`.

This file is deliberately **not** added to `ci.yml` or `publish-firmware.yml` — it requires a real `secrets.yaml`, which must never exist in the repo or in CI.

**Rejected alternative**: folding the `wifi`/`api`/`ota` snippet directly into the shared `non-iron-tagreader-core-comms.yaml`. That file is packaged by the factory chain too, which `ci.yml` compiles with no `secrets.yaml` present — hardcoding `!secret` references there would break CI and the zero-yaml-edit BLE-improv onboarding flow for both the rc522 and ST25R factory builds. Confirmed with the user; keeping the override in its own top-level file instead.

### `secrets.yaml.example`

```yaml
wifi_ssid: "your-wifi-ssid"
wifi_password: "your-wifi-password"
api_password: "generate-a-32-byte-base64-key"
ota_password: "your-ota-password"
```

`secrets.yaml` is already gitignored (`.gitignore:6`).

### CI / release wiring

- `.github/workflows/ci.yml`: add `non-iron-tagreader-st25r` to the `matrix.file` list (builds `non-iron-tagreader-st25r.factory.yaml`).
- `.github/workflows/publish-firmware.yml`: add `non-iron-tagreader-st25r.factory.yaml` to the `files:` block.
- No changes to `static/index.md` or `publish-pages.yml` — GitHub Pages is explicitly out of scope for this fork per the user.

### README.md

- New BOM entry for the M5Stack Unit NFC alongside the existing RFID2 unit.
- New Firmware section entries for all four new files, explaining factory vs. custom.
- New "Known Limitations & Workarounds" section:
  - **UID format differs from `rc522_i2c`** (dash-separated hex vs. concatenated) — existing HA tag automations keyed to rc522 UIDs will need re-scanning after switching hardware. Inherent to the chip swap.
  - **Polling only, no IRQ** — the Grove connector has no IRQ line; the component falls back to register polling, same approach the rc522 variant already uses. No functional loss.
  - **Pinned to an upstream commit, not a release** — `esphome_st25r` has no tagged releases yet; pinned to a known-working commit SHA with `refresh: never`. Documents how/when to bump it.
  - **Space-B registers (0x40–0x7F) unwritable in the current component** — antenna-tuning correlator config stays at factory defaults. Irrelevant here since `aat_enabled: false` (no varicap network on this hardware to tune anyway).
  - **No hosted installer for this variant** — flash via the ESPHome Dashboard's `dashboard_import`, `esphome run` locally with the custom yaml, or by feeding the GitHub Release's manifest/bin to the generic `web.esphome.io` installer.

## Testing plan

`esphome` isn't installed in this environment and pulling the full ESP-IDF toolchain to compile-test is expensive for this session. Plan:
1. `pip install esphome --break-system-packages`
2. `esphome config non-iron-tagreader-st25r.factory.yaml` and `esphome config non-iron-tagreader-st25r-custom.yaml` (with a throwaway local `secrets.yaml`) — schema/merge validation only, no toolchain download.
3. Push the branch and let the repo's own `ci.yml` do the real compile via `esphome/build-action`.
4. No physical M5Stack hardware available in this environment — cannot verify actual tag reads; this is a known gap, stated explicitly rather than claimed as tested.

## Explicitly out of scope

- GitHub Pages / `static/index.md` / `publish-pages.yml` changes.
- Renaming or modifying any existing rc522-path file (`non-iron-tagreader-atom-lite.yaml`, `non-iron-tagreader-core-nfc-rc522.yaml`, `non-iron-tagreader.factory.yaml`, `non-iron-tagreader-core-comms.yaml`).
- Adding the custom yaml to CI (requires secrets that must not exist in the repo).
