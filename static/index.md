- [Bill-of-Materials](#non-iron-tagreader-bill-of-materials)
- [Quickstart](#quickstart)
- [Installation](#installation)
- [How to use it](#how-to-use-it)

# Non-Iron TagReader for Home Assistant
Easy do‑it‑yourself NFC tag reader kit that requires no soldering.
Snap‑together hardware and simple self‑updating firmware make it perfect for Home Assistant enthusiasts who want a working NFC reader without any special tools.

<img width="1024" alt="8140B74D-A338-42EC-99B6-5AE2DA920D0A_1_105_c" src="https://github.com/user-attachments/assets/b7cdf02e-fb5a-4fdf-8237-ef33800cd697" />

## Why this TagReader

- Uses off‑the‑shelf parts that are easy to buy and ship worldwide
- No soldering iron required
- Many community members report that [Adonno's tagreader](https://github.com/adonno/tagreader) is often unavailable, so this project offers an accessible alternative

<img width="1024" alt="29104098-5D70-4B83-9D7E-D727CBF481E4_1_105_c" src="https://github.com/user-attachments/assets/d175fd05-8561-4544-838a-a852ba9fb2ac" />

# Non-Iron TagReader Bill-of-Materials

- Atom Lite controller [M5Stack Official Store on AliExpress](https://s.click.aliexpress.com/e/_c4rw1K9V) \
<img width="200" src="https://shop.m5stack.com/cdn/shop/products/3_2e4a5d8d-c739-405a-9494-431e2edec8ae_1200x1200.jpg" />

- RFID unit [M5Stack Official Store on AliExpress](https://s.click.aliexpress.com/e/_c43w8sz7) \
<img width="200" src="https://shop.m5stack.com/cdn/shop/products/4_7fde30d8-7a26-46a8-9d48-d11a90fbfb7c_1200x1200.jpg" />

- 3D‑printed enclosure: download the case models [here](https://github.com/luka6000/non-iron-tagreader/tree/main/STLs) or order a print through the JLC3DP service [here](https://jlc3dp.com/3d-models/detail/MX14062-case-for-Non-Iron-TagReader-for-Home-Assistant/?from=VRRKTGMREGSG). For best results, I suggest SLA resin with selectable color and surface finish. You can also check Slant3D Portals service [here](https://teleportpod.com/portal/751bada5-ca11-440a-af45-6d9e2cf4d589?item=5013)

# Quickstart

- Get the enclosure: print or buy. It is also safe to use M5Stack components without any additional case.
- Assemble using snap‑and‑fit connectors. Screws and inserts are optional.
- Flash the firmware with ESP Web Tools below.
- Use the Home Assistant mobile app to onboard the reader via Bluetooth.
- Enjoy using it!

<img width="512" alt="0751A343-CC22-42AC-964D-BEB275F40BFD_1_105_c" src="https://github.com/user-attachments/assets/8b370d38-9ad7-43d7-9deb-2822c7fe524e" />
<img width="512" alt="26D234FC-385A-40DF-85D2-9831F5BE95D2_1_105_c" src="https://github.com/user-attachments/assets/6f2a9b71-5669-4994-b3f9-5994cf393bac" />

# Firmware

- [non-iron-tagreader-atom-lite.yaml](https://github.com/luka6000/non-iron-tagreader/blob/main/non-iron-tagreader-atom-lite.yaml) simple NFC tag reader with passive BLE proxy

# Installation

You can use the button below to install the pre-built firmware directly to your device via USB from the browser.

<style>
  .invisible {
    visibility: hidden;
  }
  .radios li {
    list-style: none;
    line-height: 2em;
  }
</style>
<script type="module" src="https://unpkg.com/esp-web-tools@9/dist/web/install-button.js?module"></script>
<p>Select your version</p>
<ul class="radios">
  <li>
    <label><input type="radio" name="type" value="non-iron-tagreader-firmware"><img width=600 src="https://shop.m5stack.com/cdn/shop/products/7_da00f974-6952-4ad6-9f08-beaab6c888d5_1200x1200.jpg" alt="Non-Iron TagReader Atom Lite">
    </label>
  </li>
</ul>
<p class="button-row" align="center">
  <esp-web-install-button class="invisible"></esp-web-install-button>
</p>
<script>
  document.querySelectorAll('input[name="type"]').forEach(radio =>
    radio.addEventListener("change", () => {
      const button = document.querySelector('esp-web-install-button');
      button.manifest = `./firmware/${radio.value}.manifest.json`;
      button.classList.remove('invisible');
    }
  ));
</script>


Installer powered by [ESP Web Tools](https://esphome.github.io/esp-web-tools/)

# How to use it
- Discover your new TagReader with Home Assistant mobile app
- Open the Tags panel in Home Assistant. You can use the button below \
  [![Open your Home Assistant instance and show your tags.](https://my.home-assistant.io/badges/tags.svg)](https://my.home-assistant.io/redirect/tags/)
- Try scanning any NFC tag
- If you have a writable NTAG and want to also use phone scanning, just write the tagID back to the tag with HA mobile app and this yellow marked button \
  <img width="600" alt="562A0858-5E83-4A17-A422-88401FBECEEC_4_5005_c" src="https://github.com/user-attachments/assets/e3986ec5-b606-4bec-b0fc-bccd30383db2" /> \
  Home Assistant writes a link with tagID so it can be recognized by your phone and pushed to HA for processing.

For more information, check out the Home Assistant documentation for [tags](https://www.home-assistant.io/integrations/tag/).

<img width="1024" alt="4B0793FC-88F4-4ED7-94ED-9044D19C486D_1_102_o" src="https://github.com/user-attachments/assets/4d6b283a-419a-40f8-b3b6-560d2b305715" />

# NFC tag reader for HA options
- Adonno's tagreader [https://github.com/adonno/tagreader]()
- usb direct connected reader - any options here?
- TagTuner NFC music player [https://luka6000.github.io/TagTuner/]()

Feel free to contribute updates for other available DIY projects.

# Disclaimer
All of this is my personal hobby project, available for free download and personal use. If you’d like to support me with a coffee, beer, filament, or electronic parts, feel free to use [paypal.me/lukagra](https://paypal.me/lukagra) or [ko-fi.com/lukagra](https://ko-fi.com/lukagra)

Links to parts listed above are affiliate links, which allow me to earn a small commission from your purchase. Thank you! 🙏

This work, including yaml files, 3d models and documentation, is licensed under \
[Creative Commons (4.0 International License) Attribution—Noncommercial—Share Alike \
<img width="100" src="https://mirrors.creativecommons.org/presskit/buttons/88x31/png/by-nc-sa.png">](http://creativecommons.org/licenses/by-nc-sa/4.0/)

ESPHome components modifications are licensed under ESPHome [license](https://github.com/esphome/esphome?tab=License-1-ov-file#readme)
