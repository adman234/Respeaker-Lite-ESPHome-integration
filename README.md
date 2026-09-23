# ReSpeaker Lite ESPHome integration (Brick Assistant fork)

ESPHome firmware that turns a Seeed ReSpeaker Lite Voice Kit (XIAO ESP32-S3) into a Home
Assistant voice satellite with the same features as the Home Assistant Voice PE.

This is a fork of [formatBCE/Respeaker-Lite-ESPHome-integration](https://github.com/formatBCE/Respeaker-Lite-ESPHome-integration)
by Andrii Mitnovych ([formatBCE](https://github.com/formatBCE)), who wrote the firmware,
the custom `respeaker_lite` component, the blueprints and the case. His work builds on the
[Voice PE firmware](https://github.com/esphome/home-assistant-voice-pe) by the ESPHome team
and on [Seeed's ReSpeaker Lite firmware](https://github.com/respeaker/ReSpeaker_Lite). If
you find it useful, you can [support formatBCE](https://www.buymeacoffee.com/formatbce).
He has since moved on to [Koala Satellite](https://github.com/formatBCE/Koala-Satellite).

The upstream files are kept unchanged so the fork can keep syncing with formatBCE. This fork's
changes live alongside them in their own package and are not intended to be merged upstream.

## What is different from upstream

A separate package, `config/common/brick-assistant-base.yaml`, for ReSpeaker Lite boards
wired the "Brick Assistant" way: the user button on GPIO4, and the XMOS mute button doubling
as volume up. That is a different pin mapping from upstream's `respeaker-satellite-base.yaml`
(button on GPIO3, GPIO4 as a mute output), so do not mix the two. On top of upstream's
features it adds:

- **An LED state machine ported from Koala and Voice PE.** One script owns the LED, with
  per-phase colours (idle, listening, thinking, replying) and a brightness control in Home
  Assistant.
- **Barge-in fixes.** The "stop" wake word stays armed until the reply has finished playing,
  and can no longer lock up until a reboot.
- **Timer fixes.** The ring uses one repeating sound instead of two competing loops, and the
  timer tick no longer turns off other LED states.
- Next-timer and mic-muted sensors, switches for wake and click sounds, a restart button,
  and a hint sound when Home Assistant Cloud authentication fails.
- The fallback hotspot password moved from the YAML into `secrets.yaml`.
- Custom wake word support (a locally trained `hey_robo_joe` model).

Design notes, bugs found and fixed, and wake word gotchas are in
[readme/brick-assistant-notes.md](readme/brick-assistant-notes.md).

## Using it

Each device gets a small file in `config/devices/` that sets its name and hotspot SSID and
pulls the shared package from this repo's `main` at build time:

```yaml
packages:
  brick_assistant:
    url: https://github.com/adman234/Respeaker-Lite-ESPHome-integration
    ref: main
    files: [config/common/brick-assistant-base.yaml]
    refresh: 1d
```

Copy a device file into your ESPHome config folder and adjust it. `secrets.yaml` needs
`wifi_ssid`, `wifi_password`, `respeaker_lite__speaker_api_key` and `respeaker_ap_password`.
Changes to the package reach a device on its next build; use Clean build files in the
ESPHome dashboard to skip the one-day refresh.

For upstream's own pin mapping, use `config/respeaker-satellite-dashboard-example.yaml`
instead, as described in formatBCE's repo.

## Hardware preparation (from upstream)

1. Get Respeaker Lite with ESP32 soldered to it (you may solder it yourself, pins on the back can remain dry, they're not used).
2. [Solder USR to D2 and MUTE to D3 pins](https://wiki.seeedstudio.com/respeaker_button/). _**ATTENTION! This step is mandatory, as without it the buttons on satellite won't work as intended.**_
3. [Flash 48kHz I2S firmware of version **not lower than 1.1.0** to the XMOS board](https://wiki.seeedstudio.com/xiao_respeaker/#flash-the-i2s-firmware) (pay attention to USB port, you need the main board port, not ESP32 one). Make sure you're using 48kHz version, as 16kHz version won't work with this repo. You can use included [firmware file](/respeaker_lite_i2s_dfu_firmware_48k_v1.1.0.bin) to be sure.
4. Flash ESPHome firmware (YAML included, adjust `/config/respeaker-satellite-dashboard-example.yaml` to your needs) to ESP32 (use its port).
5. Add device to Home Assistant.

Step 2 is upstream's wiring. The Brick Assistant package reads the user button on GPIO4
and has no separate mute input, so check the pin table at the top of
`brick-assistant-base.yaml` before soldering.

Since version 2025.2.2 the firmware installs the matching XMOS firmware by itself on first
boot: the ReSpeaker LED flashes yellow while installing and green when done.

## More from upstream

- [Blueprints](readme/blueprints.md) for use with the satellite
- [TTS URI event](readme/tts_uri.md) for sending responses to another media player
- [Daily alarms](readme/alarms.md)
- [3D printable case](casing/Casing.md)
