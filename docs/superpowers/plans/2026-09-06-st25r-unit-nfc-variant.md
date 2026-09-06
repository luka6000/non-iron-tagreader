# M5Stack Unit NFC (ST25R3916) Variant Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add a second, fully flashable hardware option to this project for the M5Stack Unit NFC (ST25R3916), alongside the existing RFID2 (rc522) option.

**Architecture:** New sibling YAML files mirror the existing rc522 chain (`core-nfc` → `atom-lite` → `factory`) with the NFC driver block swapped from `rc522_i2c` to `st25r_i2c` (via `external_components` pointing at JohnMcLear/esphome_st25r, since ESPHome has no native ST25R support). A new top-level `-custom.yaml` gives technical users a secrets-based local build without touching the shared `core-comms.yaml`. No existing rc522-path file is modified.

**Tech Stack:** ESPHome YAML, GitHub Actions, `esphome` CLI for config validation.

## Global Constraints

- Never push directly to `main` — this work lands via a PR (already on branch `feature/st25r-unit-nfc-variant`).
- No comments in new YAML beyond what the mirrored original file already has — `non-iron-tagreader-core-nfc-rc522.yaml` has zero comments, so `non-iron-tagreader-core-nfc-st25r.yaml` gets zero comments.
- Do not modify `non-iron-tagreader-atom-lite.yaml`, `non-iron-tagreader-core-nfc-rc522.yaml`, `non-iron-tagreader.factory.yaml`, or `non-iron-tagreader-core-comms.yaml`.
- No GitHub Pages / `static/index.md` / `publish-pages.yml` changes.
- `non-iron-tagreader-st25r-custom.yaml` must never be added to `ci.yml` or `publish-firmware.yml` — it requires a real `secrets.yaml`, which must never exist in the repo or in CI.
- `external_components` pinned to commit `35d2318c1c419ea61dd49ff82e873322ccad022d` of `JohnMcLear/esphome_st25r` with `refresh: never` (no tags exist upstream).
- Git commit messages: plain, no `Co-Authored-By` trailer (per standing user preference).
- Spec: `docs/superpowers/specs/2026-09-06-st25r-unit-nfc-variant-design.md`

---

### Task 1: Core NFC driver, board file, and factory build for ST25R

**Files:**
- Create: `non-iron-tagreader-core-nfc-st25r.yaml`
- Create: `non-iron-tagreader-atom-lite-st25r.yaml`
- Create: `non-iron-tagreader-st25r.factory.yaml`

**Interfaces:**
- Consumes: `non-iron-tagreader-project.yaml` and `non-iron-tagreader-core-comms.yaml` (existing, unmodified — provide `tagreader_event` script, `led_blink`/`led_ok` scripts, `bus_i2c` I2C bus).
- Produces: entity IDs `is_tag`, `status`, `btn_restart`, `nfc_board` (same IDs as the rc522 file, so any downstream automation pattern from the rc522 variant transfers directly). Package name `non-iron-tagreader-atom-lite-st25r.yaml` is consumed by Task 2's custom yaml.

- [ ] **Step 1: Install the ESPHome CLI for local validation**

```bash
pip install esphome --break-system-packages
esphome version
```

Expected: prints something like `Version: 2026.x.x` with exit code 0.

- [ ] **Step 2: Create `non-iron-tagreader-core-nfc-st25r.yaml`**

```yaml
binary_sensor:
  - platform: template
    name: "=Tag="
    id: is_tag
    device_class: vibration
    entity_category: DIAGNOSTIC
    disabled_by_default: False

script:
  - id: wait_input
    then:
      - delay: 3s
      - text_sensor.template.publish:
          id: status
          state: "Waiting for input"

text_sensor:
  - platform: template
    id: status
    name: "Status"
    icon: mdi:ladybug
    entity_category: DIAGNOSTIC
    on_value:
      if:
        condition:
          lambda: 'return id(status).state != "Waiting for input";'
        then:
        - script.stop: wait_input
        - script.execute: wait_input

button:
  - platform: restart
    name: "Restart"
    id: btn_restart
    entity_category: DIAGNOSTIC

external_components:
  - source:
      type: git
      url: https://github.com/JohnMcLear/esphome_st25r
      ref: 35d2318c1c419ea61dd49ff82e873322ccad022d
    refresh: never
    components: [st25r, st25r_i2c]

st25r_i2c:
- id: nfc_board
  i2c_id: bus_i2c
  address: 0x50
  update_interval: 350ms
  aat_enabled: false
  on_tag_removed:
    - binary_sensor.template.publish:
        id: is_tag
        state: false
    - text_sensor.template.publish:
        id: status
        state: "Tag removed"
    - script.execute:
        id: tagreader_event
        action: "tagRemoved"
        uid: !lambda 'return x;'
    - script.execute: led_blink

  on_tag:
    - binary_sensor.template.publish:
        id: is_tag
        state: true
    - text_sensor.template.publish:
        id: status
        state: !lambda 'return "Tag "+x;'
    - text_sensor.template.publish:
        id: status
        state: "UID tag"
    - homeassistant.tag_scanned: !lambda |-
        return x;
    - script.execute:
        id: tagreader_event
        action: "tagScanned"
        uid: !lambda 'return x;'
    - script.execute: led_ok
```

This is a package fragment, not a standalone buildable config — it can't be validated on its own. It's exercised in Step 5 once the factory file exists.

- [ ] **Step 3: Create `non-iron-tagreader-atom-lite-st25r.yaml`**

Identical to `non-iron-tagreader-atom-lite.yaml`, with only the `core-nfc:` package line repointed:

```yaml
packages:
  project: !include non-iron-tagreader-project.yaml
  core-nfc: !include non-iron-tagreader-core-nfc-st25r.yaml
  core-comms: !include non-iron-tagreader-core-comms.yaml

# These substitutions allow the end user to override certain values
substitutions:
  name: "non-iron-tagreader"
  friendly_name: "Non-Iron TagReader"

esphome:
  name: "${name}"
  friendly_name: ${friendly_name}
  # Automatically add the mac address to the name
  # so you can use a single firmware for all devices
  name_add_mac_suffix: true

  # hello world
  on_boot:
    priority: -100
    then:
    - light.turn_on:
        id: led1
        effect: HelloWorld
    - delay: 1000ms
    - light.turn_off: led1
    - wait_until:
        condition:
          api.connected:
        timeout: 20s
    - text_sensor.template.publish:
        id: status
        state: "Ready"

esp32:
# NOTE: Double-check label on your Atom variant for pin assignments:
# - LED: GPIO27 (some Atom variants use GPIO22 or GPIO19)
# - Button: GPIO39 (some Atom variants use GPIO33)
# - I2C SDA: GPIO26, SCL: GPIO32 (AtomS3 uses GPIO2 and GPIO1)
# Update the pin numbers below if your hardware revision differs.
  board: m5stack-atom
  variant: esp32
  cpu_frequency: 80MHz #sufficient for this application and reduces power consumption
  framework:
    type: esp-idf

# debug:

# To be able to get logs from the device via serial and api.
logger:
  baud_rate: 0
  # level: VERBOSE
  # level: DEBUG
  level: WARN
  logs:
    light: WARN

i2c:
  id: bus_i2c
  sda: 26
  scl: 32
  scan: True
  frequency: 100kHz
  timeout: 13ms

esp32_ble_tracker:
  scan_parameters:
    active: false
bluetooth_proxy:
  active: false

light:
  - platform: esp32_rmt_led_strip
    id: led1
    pin: GPIO27
    chipset: SK6812
    num_leds: 1
    rgb_order: grb
    restore_mode: ALWAYS_OFF
    default_transition_length: 0s
    flash_transition_length: 0s
    effects:
      - strobe:
          name: HelloWorld
          colors:
            - state: true
              brightness: 100%
              red: 100%
              green: 0%
              blue: 0%
              duration: 250ms
            - state: true
              brightness: 100%
              red: 0%
              green: 100%
              blue: 0%
              duration: 250ms
            - state: true
              brightness: 100%
              red: 0%
              green: 0%
              blue: 100%
              duration: 250ms
            - state: true
              brightness: 100%
              red: 100%
              green: 100%
              blue: 100%
              duration: 250ms

binary_sensor:
  - platform: gpio
    id: toggle
    pin:
      number: GPIO39
      inverted: true
    on_multi_click:
    - timing:
        - ON for at most 1s
        - OFF for at most 0.25s
        - ON for at most 1s
        - OFF for at most 0.25s
        - ON for at most 1s
      then:
        - script.execute: led_blink
        - text_sensor.template.publish:
            id: status
            state: "clickTriple"
        - script.execute:
            id: tagreader_event
            action: "clickTriple"
            uid: ""
            uri: ""
            artist: ""
            playlist: ""
    - timing:
        - ON for at most 1s
        - OFF for at most 0.25s
        - ON for at most 1s
        - OFF for at least 0.25s
      then:
        - script.execute: led_blink
        - text_sensor.template.publish:
            id: status
            state: "clickDouble"
        - script.execute:
            id: tagreader_event
            action: "clickDouble"
            uid: ""
            uri: ""
            artist: ""
            playlist: ""
    - timing:
        - ON for at most 1s
        - OFF for at most 0.25s
        - ON for at least 1.5s
      then:
        - script.execute: led_blink
        - text_sensor.template.publish:
            id: status
            state: "clickHold"
        - script.execute:
            id: tagreader_event
            action: "clickHold"
            uid: ""
            uri: ""
            artist: ""
            playlist: ""
    - timing:
        - ON for 0.5s to 2s
        - OFF for at least 50ms
      then:
        - script.execute: led_blink
        - text_sensor.template.publish:
            id: status
            state: "clickLong"
        - script.execute:
            id: tagreader_event
            action: "clickLong"
            uid: ""
            uri: ""
            artist: ""
            playlist: ""
    - timing:
        - ON for at most 0.5s
        - OFF for at least 260ms
      then:
        - script.execute: led_blink
        - text_sensor.template.publish:
            id: status
            state: "clickSingle"
        - script.execute:
            id: tagreader_event
            action: "clickSingle"
            uid: ""
            uri: ""
            artist: ""
            playlist: ""

script:
  - id: led_blink
    then:
    - light.turn_off: led1
    - light.turn_on:
        id: led1
        brightness: 30%
        red: 100%
        green: 100%
        blue: 100%
        flash_length: 50ms
  - id: led_ok
    then:
    - light.turn_off: led1
    - light.turn_on:
        id: led1
        brightness: 100%
        red: 0%
        green: 100%
        blue: 0%
        flash_length: 50ms
    - delay: 100ms
    - light.turn_on:
        id: led1
        brightness: 100%
        red: 0%
        green: 100%
        blue: 0%
        flash_length: 50ms
  - id: led_success
    then:
    - light.turn_off: led1
    - light.turn_on:
        id: led1
        brightness: 100%
        red: 0%
        green: 100%
        blue: 0%
        flash_length: 200ms
    - delay: 250ms
    - light.turn_on:
        id: led1
        brightness: 100%
        red: 0%
        green: 100%
        blue: 0%
        flash_length: 200ms
    - delay: 250ms
    - light.turn_on:
        id: led1
        brightness: 100%
        red: 0%
        green: 100%
        blue: 0%
        flash_length: 500ms
```

- [ ] **Step 4: Create `non-iron-tagreader-st25r.factory.yaml`**

```yaml
# These substitutions allow the end user to override certain values
substitutions:
  name: "tagreader-st25r"
  friendly_name: "TagReader (ST25R)"

packages:
  # Include all of the core configuration
  core: !include non-iron-tagreader-atom-lite-st25r.yaml

esphome:
  name: ${name}
  friendly_name: ${friendly_name}
  # Automatically add the mac address to the name
  # so you can use a single firmware for all devices
  name_add_mac_suffix: true

  project:
    name: LukaGra.${friendly_name}
    version: dev

ota:
  - platform: http_request
    id: ota_http_request

http_request:

update:
  - platform: http_request
    name: None
    id: update_http_request
    source: https://github.com/modem7/non-iron-tagreader-st25r/releases/latest/download/non-iron-tagreader-st25r-firmware.manifest.json

# This should point to the public location of this yaml file.
dashboard_import:
  package_import_url: github://modem7/non-iron-tagreader-st25r/non-iron-tagreader-st25r.factory.yaml@main
  import_full_config: false

esp32_ble:
  name: TagReaderST25
esp32_improv:
  authorizer: toggle
```

`esp32_ble` names are capped at 13 characters — `TagReaderST25` is exactly 13, distinct from the rc522 build's `nonIronTagRdr`.

- [ ] **Step 5: Validate the factory config**

```bash
cd non-iron-tagreader-st25r
esphome config non-iron-tagreader-st25r.factory.yaml
```

Expected: ESPHome clones `JohnMcLear/esphome_st25r` at the pinned commit into its component cache, resolves all packages, and prints the fully merged config ending with `INFO Configuration is valid!` and exit code 0. If it fails on the `external_components` git clone step, check outbound network access; if it fails on schema validation (e.g. unknown key `aat_enabled`), re-check the pinned commit still exposes that option in `components/st25r/__init__.py`.

- [ ] **Step 6: Commit**

```bash
git add non-iron-tagreader-core-nfc-st25r.yaml non-iron-tagreader-atom-lite-st25r.yaml non-iron-tagreader-st25r.factory.yaml
git commit -m "Add ST25R (M5Stack Unit NFC) core-nfc, board, and factory yaml"
```

---

### Task 2: Custom secrets-based build

**Files:**
- Create: `secrets.yaml.example`
- Create: `non-iron-tagreader-st25r-custom.yaml`

**Interfaces:**
- Consumes: `non-iron-tagreader-project.yaml`, `non-iron-tagreader-atom-lite-st25r.yaml` (Task 1).
- Produces: nothing consumed by later tasks — this is a leaf entry point.

- [ ] **Step 1: Create `secrets.yaml.example`**

```yaml
wifi_ssid: "your-wifi-ssid"
wifi_password: "your-wifi-password"
api_password: "generate-a-32-byte-base64-key"
ota_password: "your-ota-password"
```

- [ ] **Step 2: Create a local, gitignored `secrets.yaml` for validation only**

```bash
cp secrets.yaml.example secrets.yaml
```

`secrets.yaml` is already listed in `.gitignore:6` — confirm before continuing:

```bash
git check-ignore -v secrets.yaml
```

Expected: prints `.gitignore:6:/secrets.yaml	secrets.yaml` (or similar), confirming it will not be committed.

- [ ] **Step 3: Create `non-iron-tagreader-st25r-custom.yaml`**

```yaml
substitutions:
  name: "tagreader-st25r"
  friendly_name: "TagReader (ST25R)"

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

- [ ] **Step 4: Validate the custom config, and confirm the merge behaviour assumed in the spec**

```bash
esphome config non-iron-tagreader-st25r-custom.yaml
```

Expected: exit code 0, ending with `INFO Configuration is valid!`. In the printed resolved config, confirm under `wifi:` that **both** `ap:` (from `core-comms.yaml`) and `ssid:`/`password:` (from this file) are present — this confirms ESPHome's package merge combines dict keys rather than one replacing the other, which is the behaviour the design spec relies on. Do the same check for `ota:` (`platform: esphome` plus `password:`) and `logger:` (`logs.light: WARN` plus `logs.component: ERROR`, with `level: DEBUG` winning over the package's `WARN`).

- [ ] **Step 5: Remove the local test secrets file**

```bash
rm secrets.yaml
git status
```

Expected: `secrets.yaml` does not appear in `git status` output (it never will, since it's gitignored, but this confirms no other stray files were created).

- [ ] **Step 6: Commit**

```bash
git add secrets.yaml.example non-iron-tagreader-st25r-custom.yaml
git commit -m "Add ST25R custom build with user-supplied secrets"
```

---

### Task 3: CI and release workflow wiring

**Files:**
- Modify: `.github/workflows/ci.yml`
- Modify: `.github/workflows/publish-firmware.yml`

**Interfaces:**
- Consumes: `non-iron-tagreader-st25r.factory.yaml` (Task 1) by filename convention (`${{ matrix.file }}.factory.yaml`).
- Produces: nothing consumed by later tasks.

- [ ] **Step 1: Add the ST25R variant to the CI matrix**

In `.github/workflows/ci.yml`, change:

```yaml
      matrix:
        file:
          - non-iron-tagreader
```

to:

```yaml
      matrix:
        file:
          - non-iron-tagreader
          - non-iron-tagreader-st25r
```

- [ ] **Step 2: Add a separate release build job for the ST25R variant**

`combined-name` isn't just a label — in the pinned `esphome/workflows/build.yml`, when set it merges every file in that job's `files:` list into **one** combined `manifest.json` (concatenating their `builds` arrays, keeping only the first file's top-level name/version metadata). That's for building the *same* project across multiple ESP32 chip variants, not for combining two different NFC readers — adding `non-iron-tagreader-st25r.factory.yaml` to the existing job's `files:` would corrupt both manifests' metadata into one asset instead of two independent ones. So this gets its own job instead, verified against `upload-to-gh-release.yml`'s source, which downloads and uploads *every* artifact from the run with no name filter, so multiple `build-firmware-*` jobs feeding one `upload-to-release` just works.

In `.github/workflows/publish-firmware.yml`, change:

```yaml
  upload-to-release:
    name: Upload to Release
    uses: esphome/workflows/.github/workflows/upload-to-gh-release.yml@0fdd5e311b7e744069166696072a1a9cbc5fbeb6  # 2026.8.1
    needs:
      - build-firmware
    with:
      version: ${{ (github.event_name == 'release' && github.event.release.tag_name) || (github.event_name == 'workflow_dispatch' && inputs.version) || '' }}
```

to:

```yaml
  build-firmware-st25r:
    name: Build ST25R Firmware
    uses: esphome/workflows/.github/workflows/build.yml@0fdd5e311b7e744069166696072a1a9cbc5fbeb6  # 2026.8.1
    with:
      files: |
        non-iron-tagreader-st25r.factory.yaml
      esphome-version: stable
      combined-name: non-iron-tagreader-st25r-firmware

      release-summary: ${{ github.event.release.body }}
      release-url: ${{ github.event.release.html_url }}
      release-version: ${{ (github.event_name == 'release' && github.event.release.tag_name) || (github.event_name == 'workflow_dispatch' && inputs.version) || '' }}

  upload-to-release:
    name: Upload to Release
    uses: esphome/workflows/.github/workflows/upload-to-gh-release.yml@0fdd5e311b7e744069166696072a1a9cbc5fbeb6  # 2026.8.1
    needs:
      - build-firmware
      - build-firmware-st25r
    with:
      version: ${{ (github.event_name == 'release' && github.event.release.tag_name) || (github.event_name == 'workflow_dispatch' && inputs.version) || '' }}
```

- [ ] **Step 3: Validate both workflow files are still well-formed YAML**

```bash
python3 -c "import yaml; yaml.safe_load(open('.github/workflows/ci.yml'))" && echo OK
python3 -c "import yaml; yaml.safe_load(open('.github/workflows/publish-firmware.yml'))" && echo OK
```

Expected: `OK` printed twice, no exceptions.

- [ ] **Step 4: Commit**

```bash
git add .github/workflows/ci.yml .github/workflows/publish-firmware.yml
git commit -m "Build and release the ST25R factory firmware alongside rc522"
```

---

### Task 4: README documentation

**Files:**
- Modify: `README.md`

**Interfaces:**
- Consumes: nothing (documentation only).
- Produces: nothing consumed by later tasks.

- [ ] **Step 1: Add the M5Stack Unit NFC to the Bill-of-Materials**

Find this block in `README.md`:

```markdown
- RFID unit [M5Stack on AliExpress](https://s.click.aliexpress.com/e/_c3JSZivv) \
<img width="200" src="https://shop.m5stack.com/cdn/shop/products/4_7fde30d8-7a26-46a8-9d48-d11a90fbfb7c_1200x1200.jpg" />
```

Replace it with:

```markdown
- RFID unit (WS1850S) [M5Stack on AliExpress](https://s.click.aliexpress.com/e/_c3JSZivv) — for the standard rc522-based firmware \
<img width="200" src="https://shop.m5stack.com/cdn/shop/products/4_7fde30d8-7a26-46a8-9d48-d11a90fbfb7c_1200x1200.jpg" />

- Unit NFC (ST25R3916) [M5Stack docs](https://docs.m5stack.com/en/unit/Unit_NFC) — alternative unit, use the ST25R firmware below
```

No product image for the Unit NFC is added here — do not fabricate or guess an image URL for it (this session doesn't have a verified working image link for this product). If a real product image URL is wanted later, get it by fetching the linked M5Stack docs page and using an image URL that actually appears there.

- [ ] **Step 2: Add the ST25R firmware files to the Firmware section**

Find this block:

```markdown
# Firmware

- [non-iron-tagreader-atom-lite.yaml](https://github.com/luka6000/non-iron-tagreader/blob/main/non-iron-tagreader-atom-lite.yaml) simple NFC tag reader with passive BLE proxy
```

Replace it with:

```markdown
# Firmware

- [non-iron-tagreader-atom-lite.yaml](https://github.com/luka6000/non-iron-tagreader/blob/main/non-iron-tagreader-atom-lite.yaml) simple NFC tag reader with passive BLE proxy, for the RFID2 (WS1850S) unit
- [non-iron-tagreader-st25r.factory.yaml](non-iron-tagreader-st25r.factory.yaml) pre-built firmware for the M5Stack Unit NFC (ST25R3916), flashed via ESP Web Tools and paired over Bluetooth like the standard build
- [non-iron-tagreader-st25r-custom.yaml](non-iron-tagreader-st25r-custom.yaml) same ST25R hardware, for building locally with your own `secrets.yaml` (copy `secrets.yaml.example`) instead of the pre-built Bluetooth-paired firmware — run `esphome run non-iron-tagreader-st25r-custom.yaml`
```

- [ ] **Step 3: Add a Known Limitations & Workarounds section**

Find this block:

```markdown
# NFC tag reader for HA options
```

Insert immediately before it:

```markdown
# ST25R (M5Stack Unit NFC) Known Limitations & Workarounds

- **Tag UIDs look different from the RFID2 build.** `st25r_i2c` formats UIDs as dash-separated hex (e.g. `04-1A-A7-67-5F-61-80`), while `rc522_i2c` doesn't. If you're switching a device from the RFID2 unit to the Unit NFC, re-scan your tags in Home Assistant's Tags panel — existing tag automations keyed to the old UID strings won't match.
- **Polling only, no IRQ.** The Unit NFC's Grove cable only carries GND/5V/SDA/SCL, so there's no IRQ line. This firmware polls for tags every 350ms instead, the same approach the RFID2 build already uses — there's no functional difference.
- **The ST25R driver is pinned to a commit, not a release.** [JohnMcLear/esphome_st25r](https://github.com/JohnMcLear/esphome_st25r) has no tagged releases yet, so `non-iron-tagreader-core-nfc-st25r.yaml` pins `external_components` to a known-working commit instead of tracking `main`. To pick up upstream fixes, check that repo's commits/CHANGELOG.md and update the `ref:` value.
- **Automatic Antenna Tuning is disabled on purpose.** AAT needs a varicap network the M5Stack Unit NFC's fixed antenna matching doesn't have, so `aat_enabled: false` avoids unnecessary startup delay. As a side effect of the underlying component's current register-write limitation (writes are masked to `addr & 0x3F`, so the 0x40-0x7F "Space B" registers used by AAT's correlator config aren't reachable anyway), this isn't a loss for this hardware.
- **No hosted install page for this variant.** Flash it with the ESPHome Dashboard's "Add Device" → import from this repo's `non-iron-tagreader-st25r.factory.yaml`, run `esphome run non-iron-tagreader-st25r-custom.yaml` locally, or download the manifest/firmware from this repo's [Releases](https://github.com/modem7/non-iron-tagreader-st25r/releases) and feed them to [web.esphome.io](https://web.esphome.io).
```

- [ ] **Step 4: Commit**

```bash
git add README.md
git commit -m "Document the ST25R (M5Stack Unit NFC) variant in the README"
```

---

### Task 5: Push and open the PR

**Files:** none (git/GitHub operations only).

- [ ] **Step 1: Push the branch**

```bash
git push -u origin feature/st25r-unit-nfc-variant
```

- [ ] **Step 2: Open the PR**

```bash
gh pr create --title "Add M5Stack Unit NFC (ST25R3916) firmware variant" --body "$(cat <<'EOF'
## Summary
- Adds a second hardware option (M5Stack Unit NFC / ST25R3916) alongside the existing RFID2 (rc522) support, via JohnMcLear/esphome_st25r as an external_components source (ESPHome has no native ST25R component).
- New files: non-iron-tagreader-core-nfc-st25r.yaml, non-iron-tagreader-atom-lite-st25r.yaml, non-iron-tagreader-st25r.factory.yaml, non-iron-tagreader-st25r-custom.yaml, secrets.yaml.example.
- ci.yml and publish-firmware.yml build/release the new factory variant; the custom yaml is intentionally excluded from CI since it requires secrets.
- No existing rc522-path file was modified, and GitHub Pages/static/index.md was left untouched per request.

## External component pin
external_components is pinned to JohnMcLear/esphome_st25r@35d2318c1c419ea61dd49ff82e873322ccad022d (refresh: never) — no tags exist upstream yet. Bump by updating the `ref:` in non-iron-tagreader-core-nfc-st25r.yaml after checking the upstream CHANGELOG.

## Known limitations
See the new "ST25R (M5Stack Unit NFC) Known Limitations & Workarounds" section in README.md — UID format change from rc522, polling-only (no IRQ line on the Grove connector), AAT disabled (no varicap network on this hardware), and no hosted installer for this variant.

## Test plan
- [ ] `esphome config non-iron-tagreader-st25r.factory.yaml` passes locally
- [ ] `esphome config non-iron-tagreader-st25r-custom.yaml` passes locally with a throwaway secrets.yaml
- [ ] CI (this PR) compiles both factory yaml files successfully
- [ ] Manual hardware test: M5Stack Unit NFC detects a tag and it appears in Home Assistant's Tags panel (not verified in this environment — no physical hardware available)

🤖 Generated with [Claude Code](https://claude.com/claude-code)
EOF
)"
```

- [ ] **Step 3: Report the PR URL back to the user**

The `gh pr create` command prints the PR URL on success — surface it directly.

---

## Self-Review Notes

- **Spec coverage:** every file, config decision (address, update_interval, aat_enabled, pinned commit), CI/release change, README section, and the rejected core-comms.yaml alternative from the spec has a corresponding task/step. The spec's testing plan (pip install esphome, `esphome config` on both new top-level files, rely on CI for the real compile, no physical hardware) is reflected in Tasks 1 and 2 and called out explicitly in the PR test plan.
- **Placeholder scan:** no TBD/TODO; every step has literal file content or literal commands with stated expected output.
- **Type/ID consistency:** `nfc_board`, `is_tag`, `status`, `btn_restart`, `bus_i2c`, `led_blink`, `led_ok`, `tagreader_event` are used identically to the existing rc522 chain throughout Task 1; `non-iron-tagreader-atom-lite-st25r.yaml` (Task 1) is the exact filename Task 2's custom yaml and Task 1's own factory yaml both reference.
