# ReSpeaker Lite + ESPHome + Home Assistant: working notes

Notes from a session working on two Seeed ReSpeaker Lite satellites running the
"Brick Assistant" ESPHome config (a fork of formatBCE's
[Respeaker-Lite-ESPHome-integration](https://github.com/formatBCE/Respeaker-Lite-ESPHome-integration)).

Line-number citations below point at the ESPHome `dev` branch and Home Assistant
`dev` as they were when this was written. They will drift, so treat them as "look
near here", not as permanent addresses.

## Devices

| | Device 1 | Device 2 |
|---|---|---|
| `name` | `respeaker-lite-faed68` | `respeaker-lite-2-fae5ec` |
| `friendly_name` | `ReSpeaker-lite-1` | `ReSpeaker-lite-2` |
| `use_address` | `respeaker-lite-faed68.local` | `respeaker-lite-2-fae5ec.local` |

### Layout

| File | What it holds |
|---|---|
| `config/common/brick-assistant-base.yaml` | Everything shared. Pulled from GitHub at build time via `packages:`. |
| `config/devices/respeaker-lite-1.yaml`, `-2.yaml` | ~30 lines each: `name`, `friendly_name`, `ap_ssid` substitutions, the API key secret, and the local `hey_robo_joe` model. |

Copy a device file into the ESPHome config dir (`/data` in the container) to use it.
Changes to the base take effect on the next build after they are pushed to `main`
(the package has `refresh: 1d`; use **Clean build files** in the dashboard to force it).

Device files append to lists in the base, e.g. extra `micro_wake_word: models:`.
To change a field of something the base defines, use `!extend <id>`.

`secrets.yaml` must contain `wifi_ssid`, `wifi_password`,
`respeaker_lite__speaker_api_key` and **`respeaker_ap_password`** (new; the fallback
hotspot password used to be hard-coded in the YAML).

This is a different hardware mapping from upstream's `respeaker-satellite-base.yaml`
(which uses GPIO3 as the button and GPIO4 as a mute output). Don't mix the two.

---

## 1. Barge-in: interrupting the assistant mid-response

### Does it work?

Yes, and the config already had the machinery. Two separate paths:

- **Wake word during a response**: works because `stop_after_detection: false`
  keeps microWakeWord inferring continuously, right through TTS playback.
- **Saying "stop"**: a dedicated `stop` mWW model, armed only while the device
  is talking or a timer is ringing.

None of this would work without acoustic echo cancellation. The ReSpeaker Lite's
XMOS XU316 does AEC in hardware, which is what lets the mic hear you over the
speaker. The mic split is `micro_wake_word` → `channels: 1` + `gain_factor: 4`,
`voice_assistant` → `channels: 0`. That matches formatBCE's upstream base config
exactly. **Do not change it**.

### `voice_assistant: micro_wake_word:` does NOT pause mWW

A common misreading. The key exists only so HA can enumerate and select wake
words; its only uses are `on_set_configuration()` and `get_configuration()`
(`voice_assistant.cpp:1097-1147`). It never stops or starts mWW around the
pipeline.

### Bugs found and fixed

**1. `on_end` disarmed the stop word while audio was still playing.**
With streaming TTS, `end_trigger_` fires when the TTS *stream* ends, not when the
audio buffer drains. The old code did `wait_until: not voice_assistant.is_running`
then immediately `micro_wake_word.disable_model: stop`, so "stop" went deaf for
the tail of every longer reply. Fixed by waiting for the announcement to actually
finish, guarded so a barge-in that started a new turn doesn't get its
freshly-armed model disarmed by the previous turn's pending `on_end`.

**2. `activate_stop_word_once` could deadlock permanently.**
Its first `wait_until: media_player.is_announcing` had no timeout. If the
announcement never arrived (empty/failed TTS, API hiccup) the script blocked
forever, and because scripts default to `mode: single`, every later call was
silently skipped. Stop word dead until reboot. Fixed with `timeout: 15s` plus a
re-check afterwards (the wait can now time out rather than succeed).

Keep it at `mode: single`. Do **not** switch to `restart`: `on_intent_progress`
fires repeatedly and would thrash the model on/off.

**3. `stop` cutoff was 0.2; the model's own manifest says 0.5.**
At 0.2 the residual echo of our own TTS through the AEC was enough to self-trigger
and cut responses off. Settled on **0.4**.

Note this same model silences a ringing timer by voice
(`timer_ringing.on_turn_on` → `id(stop).enable()`), so the cutoff affects that too.

**4. Cosmetic: 1s deaf window at the start of each response**: reduced to 250ms.

### Things that are not bugs

- `request_stop()` in `STREAMING_RESPONSE` already stops the announcement itself
  (`voice_assistant.cpp:724+`), so the explicit `media_player.stop` in
  `on_wake_word_detected` is redundant but harmless.
- Saying "stop" only kills the **announcement** pipeline. It will not stop music
  on the media pipeline; `activate_stop_word_once` only runs from `on_tts_start`
  / `on_intent_progress`. Say the wake word, then "stop".
- If AEC still can't keep up, lower `volume_max` (currently `0.8`). That's the
  usual physical limit on barge-in.

---

## 2. Only ONE wake word is active at a time

The single most important constraint, and it is not obvious.

When HA connects it calls `on_set_configuration()`, which **disables every
non-internal model** and enables only what HA has selected, with
`max_active_wake_words = 1` (`voice_assistant.cpp:1097-1115`).

So with `hey_jarvis` selected in HA, `okay_nabu`, `kenobi`, `hey_mycroft` and
`hey_robo_joe` are all disabled. Disabled is real, not cosmetic: a disabled model
has its TFLite interpreter destroyed and tensor arenas freed on the next inference
call (`streaming_model.h:40-41`). They cost **flash**, not runtime RAM or CPU.

Keeping extra models listed is worth it only so you can switch from the HA
dropdown without reflashing.

### The escape hatch: `internal: true`

`get_wake_words()` filters internal-only models out (`micro_wake_word.cpp:252-255`),
so HA never sees them and never disables them. That's how `stop` stays alive
alongside the selected wake word.

Two consequences:
- Internal models don't persist their enabled state to flash
  (`streaming_model.cpp:326-331`), purely runtime-controlled.
- Only the **first** model in the list is enabled by default
  (`default_enabled = i == 0`, `micro_wake_word/__init__.py:591`), so an internal
  model must be enabled explicitly, e.g. from `api: on_client_connected`.

**To run two wake words simultaneously, the second must be `internal: true`**:
and it then disappears from HA's dropdown.

---

## 3. Custom wake words

### Supported, and already in use

`kenobi` in this config is a community-trained custom model, not a built-in.
A model is just a `.tflite` plus a JSON manifest:

```json
{
  "type": "micro",
  "wake_word": "Hey Robo Joe",
  "author": "you",
  "version": 2,
  "trained_languages": ["en"],
  "micro": {
    "probability_cutoff": 0.7,
    "feature_step_size": 10,
    "sliding_window_size": 5,
    "tensor_arena_size": 30000,
    "minimum_esphome_version": "2024.7.0"
  }
}
```

`author`, `version`, `wake_word`, `trained_languages` and everything under `micro`
are required in v2; `website` is optional (`micro_wake_word/__init__.py:144-163`).

### Source forms accepted by `model:`

```yaml
    - model: hey_jarvis                              # built-in name → esphome/micro-wake-word-models
    - model: my_models/hey_computer.json             # local, relative to the CONFIG DIR
    - model: /data/whatever/hey_computer.json        # local, absolute
    - model: https://example.com/hey_computer.json   # http(s)
    - model: github://user/repo/path/model.json@main # github shorthand
```

A bare name without `.json` is treated as a built-in and resolved against
`github.com/esphome/micro-wake-word-models/.../models/v2/`, so a local file must be
written as a path or end in `.json`.

### Gotchas that will bite

**`feature_step_size` must match across every model in the block.** Hard
config-time check (`micro_wake_word/__init__.py:495-507`); a mismatch fails the
build with `Cannot load models with different features step sizes`. Everything
here uses `10`. This is the most likely cause of a failed build.

**The `.tflite` must sit next to the `.json`.** Resolved as
`manifest_path.parent / manifest["model"]` (`micro_wake_word/__init__.py:464`).
The tflite path never appears in YAML; it comes from the manifest's `"model"` key.

**The manifest's `wake_word` string is what everything sees.** It's the label in
HA's dropdown (not the `id`), and it's what `on_wake_word_detected` lambdas compare
against, e.g. `wake_word != "Stop"`.

**Models are embedded in the firmware at compile time.** There is no runtime
download, so hosting on GitHub buys nothing over local files except a build-time
fetch. Use local files unless you want to share the model.

### Path resolution

`_existing_path()` (`config_validation.py:2012-2016`) tries
`CORE.relative_config_path(value)` first, then beside the declaring YAML document.
`relative_config_path` is `config_dir / path` (`core/__init__.py:849-851`), and
pathlib's `/` discards the left side when the right is absolute:

```
Path('/config') / Path('/data/micro_wake_word/robo-joe/hey_robo_joe.json')
  -> /data/micro_wake_word/robo-joe/hey_robo_joe.json
```

So **an absolute container path works regardless of what the config dir is**:
useful when the model lives outside it.

### Warning: `/data/micro_wake_word/` is ESPHome's own cache directory

`CORE.data_dir` resolves to `/data` when `ESPHOME_IS_HA_ADDON` or
`ESPHOME_DATA_DIR` is set (`core/__init__.py:819-824`), and micro_wake_word then
uses it for git/http model caching (`micro_wake_word/__init__.py:479`):

```python
base_dir = Path(CORE.data_dir) / DOMAIN          # -> /data/micro_wake_word
file = base_dir / h.hexdigest()[:8] / model_config[CONF_FILE]
```

Hand-placed files under there currently survive (ESPHome writes into hash-named
subfolders), but it is ESPHome scratch space. **Recommended: move to a neutral
folder** such as `/mnt/user/appdata/esphome-data/custom_wake_words/robo-joe/`
(container `/data/custom_wake_words/robo-joe/`) and update the one `model:` line.

### Training

- Official: [OHF-Voice/micro-wake-word](https://github.com/OHF-Voice/micro-wake-word).
  Their own README is blunt: training a usable model "requires experimentation"
  and remains "very difficult".
- Easier: [alfiedennen/microwakeword-trainer](https://github.com/alfiedennen/microwakeword-trainer),
  a Colab wrapper with the common failures patched. ~45 min on an A100.
- Pick a phrase of **3–4 syllables** that isn't common in conversation.

---

## 4. Different wake word → different Assist pipeline

**Not supported natively.** HA resolves the pipeline **solely** from the
`select.<device>_assistant` entity, matched by pipeline *name*
(`assist_satellite/entity.py:604-621`):

```python
def _resolve_pipeline(self) -> str | None:
    if not (pipeline_entity_id := self.pipeline_entity_id):
        return None
    ...
    if pipeline_entity_state.state != OPTION_PREFERRED:
        for pipeline in async_get_pipelines(self.hass):
            if pipeline.name == pipeline_entity_state.state:
                return pipeline.id
```

The device *does* send the wake word (`msg.wake_word_phrase`,
`voice_assistant.cpp:356`), but HA uses it **only** for cross-satellite dedup
(`DATA_LAST_WAKE_UP` / `DuplicateWakeUpDetectedError` in `assist_pipeline/pipeline.py`),
the logic that stops both ReSpeakers answering the same "hey jarvis". It never
influences pipeline choice.

### How to build it anyway

Requires a second wake word marked `internal: true` (see §2), then branch in
`on_wake_word_detected`. Drive it **from the device**, not from an HA automation
on `esphome.wake_word_detected`. That is a race you'd usually win and
occasionally lose silently.

`homeassistant.action` supports `on_success` / `on_error` (`api/__init__.py:686-688`),
which removes the race entirely:

```yaml
                - if:
                    condition:
                      lambda: return wake_word == "Kenobi";
                    then:
                      - homeassistant.action:
                          action: select.select_option
                          data:
                            entity_id: select.respeaker_lite_1_assistant
                            option: "Jarvis LLM"
                          on_success:
                            - voice_assistant.start:
                                wake_word: "Kenobi"
                                silence_detection: true
                          on_error:
                            - voice_assistant.start:
                                wake_word: "Kenobi"
                                silence_detection: true
                    else:
                      - homeassistant.action:
                          action: select.select_option
                          data:
                            entity_id: select.respeaker_lite_1_assistant
                            option: "Home Assistant"
                          on_success:
                            - voice_assistant.start:
                                wake_word: !lambda return wake_word;
                                silence_detection: true
```

Notes:
- Hardcode the string in the branch rather than using `wake_word` inside
  `on_success`, because the api component warns that trigger args are stored until the
  response arrives.
- **Set the select in both branches.** It's sticky; without the `else` you stay on
  the LLM pipeline after one alternate-wake-word turn.
- Pipeline names must match exactly. A typo falls through to `return None`, which
  silently uses the preferred pipeline rather than erroring.
- If `on_success` never fires (no action response), fall back to a flat
  `- delay: 250ms` between the switch and the start.
- `voice_assistant.start` defaults `silence_detection: True`
  (`voice_assistant/__init__.py:420`).

**Status: designed, not implemented.** Needs the pipeline names and `select`
entity IDs.

---

## 5. LED behaviour

### How it works (ported from Koala / Voice PE)

One script, `control_leds`, owns the LED. Everything else updates state (the
`voice_assistant_phase` global, a switch, `led_feedback`) and calls
`script.execute: control_leds`. The script is a priority chain; the first match wins:

| Priority | State | Look |
|---|---|---|
| 1 | Booting (HA not yet connected) | blue pulse (fast once Wi-Fi is up) |
| 2 | Wi-Fi or API down | red slow pulse |
| 3 | Siren | red fast pulse, full brightness |
| 4 | Timer ringing | cyan fast pulse |
| 5 | Notification | solid pale cyan |
| 6 | Button feedback (volume limit / "stop" heard) | brief solid blue / red |
| 7 | Waiting / listening | **LED Listening** colour, slow pulse |
| 8 | Thinking | **LED Thinking** colour, `Breathe` |
| 9 | Replying | **LED Replying** colour, `Breathe` |
| 10 | Pipeline error | red fast pulse (held 1 s) |
| 11 | HA connected but no Assist pipeline | orange slow pulse |
| 12 | Timer running | yellow blip once a second |
| 13 | Idle | **LED Idle** colour if on (night light), else off |

### Home Assistant controls

- **LED Brightness** (number, 5–100 %): master brightness for all status animations.
- **LED Idle / Listening / Thinking / Replying** (colour lights, config category):
  colour wheel, brightness and on/off per phase. Brightness multiplies with the
  master. Off = LED stays dark in that phase. These drive no hardware; they're
  colour pickers backed by a do-nothing template output.
- Output brightness is `(master × phase brightness)²`, so the sliders feel linear to
  the eye (the LED itself runs with `gamma_correct: 1.0`).
- Alerts (1–6, 10–12) use fixed colours at `max(master, 35 %)²` so they stay visible.
- Default colours (first flash only; saved HA values win afterwards): Listening sky
  blue, Thinking purple, Replying green, Idle off.
- A 1 s watchdog resets a stuck phase to idle if the assistant has been quiet for 3 s
  (logs `Assistant idle but LED phase N still set`).

The old **LED Light** entity is gone; HA will show it as unavailable, so delete it.

### Why the old LED was buggy

1. **Brightness was applied twice.** ESPHome already scales every pixel an
   addressable effect writes by the light's brightness
   (`AddressableLight::update_state` → `correction_.set_local_brightness`). The old
   effects multiplied by `get_brightness()` again, so 35 % became ~12 %.
2. **Gamma 2.8 on an 8-bit LED** rounds anything under ~14 % of full scale to 0. After
   (1), the "thinking" breath was almost always black or flickering between 0 and 1.
3. **Two lights on one pixel.** The user-facing partition light and `led_internal`
   overwrote each other and their on/off states disagreed.
4. **Barge-in race.** The previous turn's `on_end` switched the LED off just after
   the new wake word lit it.

Fixes: effects scale `current_color` by their animation fraction only;
`gamma_correct: 1.0` with the effects squaring the fraction for a natural-looking
breath; one owner (`control_leds`); and a `wake_pending` flag that the old turn's
`on_end` checks before resetting the phase.

`Breathe` is shared by thinking and replying and isn't restarted when the colour
changes, so the switch happens mid-breath without a hiccup.

---

## 6. Current state of the YAML

Carried over from the earlier barge-in work:

1. `stop` model `probability_cutoff` **0.4**
2. `activate_stop_word_once`: `timeout: 15s` + re-check; 250 ms guard; `mode: single`
3. `on_end` keeps `stop` armed until audio finishes; guarded against barge-in
4. **`hey_robo_joe`** as an HA-selectable wake word (in the device files)

New in the package refactor:

5. Koala-style `control_leds` state machine, per-phase colour lights, LED Brightness
6. VA phases from `on_listening` / `on_stt_vad_start` / `on_stt_vad_end` /
   `on_intent_progress` / `on_tts_start`, plus `on_error` (ignores
   `duplicate_wake_up_detected` and `stt-no-text-recognized`; plays a hint sound on
   `cloud-auth-failed`)
7. **Timer ring fix:** `on_timer_finished` used to run its own sound loop *and* turn
   on `timer_ringing`, whose `on_turn_on` ran a second loop. Both called `play_sound`
   with `priority: true` and could cut each other off. Now a single looping
   announcement (`media_player.repeat_one` + 500 ms playlist delay), as in Voice PE.
8. Timer tick no longer turns the LED off (it used to kill the siren/notification
   colour once a second)
9. New entities: **Next timer**, **Next timer name** (disabled by default),
   **Mic muted**, **Wake sound**, **Button click sounds**, **Disable physical
   controls** (disabled by default), **Restart** (disabled by default)
10. `okay_nabu` / `stop` model URLs moved to `OHF-Voice/micro-wake-word` (upstream did
    the same)
11. AP password moved to `secrets.yaml`; hotspot SSID per device
12. Button debounce (30 ms)

Model list (base + device file):

| # | id | internal | cutoff |
|---|---|---|---|
| 0 | `okay_nabu` | no | 0.7 |
| 1 | `kenobi` | no | 0.7 |
| 2 | `hey_jarvis` | no | 0.7 |
| 3 | `hey_mycroft` | no | 0.8 |
| 4 | `stop` | **yes** | 0.4 |
| 5 | `hey_robo_joe` | no | *(from manifest)* |

`hey_robo_joe` deliberately has **no `probability_cutoff` override**. The manifest
carries the value the training run produced. Add an override only after checking
the logs. Host path: `/mnt/user/appdata/esphome-data/micro_wake_word/robo-joe/`
(see the warning in §3 about relocating this).

---

## 7. Tuning and troubleshooting

`logger:` is at `initial_level: WARN`, so detections aren't logged by default.
Use the **Logger Level** entity to bump it to DEBUG at runtime.

**Tuning a wake word cutoff:**
```
Detected 'Hey Robo Joe' with sliding average probability is 0.82 and max probability is 0.94
```
Say the phrase ~10 times, note the range, set `probability_cutoff` a little below
the lowest reliable value.

**If you instead see:**
```
Wake word model predicts 'Hey Robo Joe', but VAD model doesn't.
```
The wake word *was* recognised but `vad: probability_cutoff` (currently `0.1`)
gated it. Lower the VAD cutoff, not the model cutoff.

**Build fails with `Cannot load models with different features step sizes`**:
a manifest has a `feature_step_size` other than 10.

**Build needs internet**: `okay_nabu`, `kenobi` and `stop` are fetched from GitHub
at compile time. Only `hey_robo_joe` is local.

**Container permissions**: the ESPHome container runs as its own user; model files
must be readable inside the container, not just from Unraid.

**Wake word missing from HA's dropdown**: reload the ESPHome integration. Also
check it isn't marked `internal: true`.

---

## 8. Open items

- [ ] **Relocate the model** out of `/data/micro_wake_word/` (§3)
- [ ] **Shared API encryption key**: both device files use
      `!secret respeaker_lite__speaker_api_key`. Now trivial to split: change the
      secret name in `config/devices/respeaker-lite-2.yaml`.
- [ ] **Wake word → pipeline routing** (§4): designed, needs pipeline names and
      `select` entity IDs.
- [ ] **`hey_robo_joe` isn't wired into the "Wake word sensitivity" select.**
      Same as `kenobi`. Adding it needs quantized uint8 values (`0.85` → `217`).
- [ ] Not ported from Koala/upstream (possible later): alarm clock (`datetime` +
      time sync), Improv BLE provisioning, factory-reset long press.

---

## Reference

- [formatBCE/Respeaker-Lite-ESPHome-integration](https://github.com/formatBCE/Respeaker-Lite-ESPHome-integration)
- [OHF-Voice/micro-wake-word](https://github.com/OHF-Voice/micro-wake-word): training framework
- [alfiedennen/microwakeword-trainer](https://github.com/alfiedennen/microwakeword-trainer): Colab wrapper
- [esphome/micro-wake-word-models](https://github.com/esphome/micro-wake-word-models): built-in models
- [ESPHome micro_wake_word docs](https://esphome.io/components/micro_wake_word/)
- [HA: Wake words for Assist](https://www.home-assistant.io/voice_control/create_wake_word/)
