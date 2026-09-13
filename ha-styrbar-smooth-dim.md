# ZHA - IKEA Styrbar Smooth Dimming + Color Presets (Non-IKEA-light safe)

A Home Assistant blueprint that lets an **IKEA Styrbar (Remote Control N2)**, paired
via **ZHA**, control a light — including non-IKEA Zigbee lights (e.g. Aqara LED
strips, generic Tuya bulbs) that don't handle rapid, overlapping brightness commands
well, or don't support Zigbee binding.

## Features

- Short press Up → turn on **at a configurable default white temperature and
  brightness** (own settings, independent of the 6 color/temp presets), with a
  gentle fade-in; Short press Down → turn off, with a gentle fade-out
- Hold Up → smooth dim up using a **native Zigbee Level Control "Move" command**.
  The light's own firmware ramps the brightness in hardware, so it's as smooth as an
  original IKEA bulb — no repeated HA-side commands, no jitter.
- Hold Down → smooth dim down in small steps, with a configurable minimum brightness
  floor so the light never turns off by accident. (Native Move isn't used here
  because it has no floor concept and would turn the light off at 0.)
- Left / Right (short press) → jump to the previous / next preset in a list of **6
  presets — the first 4 are RGB colors, the last 2 are native white temperatures**.
  No helper entity needed — the automation matches the light's current color mode
  and value against the list. Whites use the light's native tunable-white channel
  (`color_temp_kelvin`) rather than an RGB approximation.
- Release button → dimming/Move stops immediately
- All steps, speeds, colors, and thresholds are configurable directly in the Home
  Assistant UI after import — no YAML editing needed.

## Why another Styrbar blueprint?

Most existing Styrbar blueprints target **Zigbee2MQTT**, not ZHA, and use repeated
`brightness_step_pct` commands with a transition time longer than the delay between
steps. That combination makes the automation read a stale `brightness` state
attribute before the previous transition has finished, causing jerky/uneven dimming —
especially noticeable on non-IKEA Zigbee lights that report state more slowly than
IKEA bulbs do, and non-IKEA lights often don't support the same Zigbee binding
shortcuts either.

This blueprint sidesteps the issue for dim-up by sending a single native Zigbee Move
command instead of a step loop. Dim-down still uses a step loop (for the minimum
brightness floor), but keeps `transition` shorter than or equal to the repeat delay
by default so each step reads a fresh value.

## Requirements

- Home Assistant with the **ZHA** integration
- IKEA Styrbar remote (Remote Control N2), paired via ZHA
- A Zigbee light that supports the Level Control cluster (`0x0008`) for dimming, and
  the Color Control cluster (`0x0300`) if you want the color-preset feature

## Installation

1. In Home Assistant: **Settings → Automations & Scenes → Blueprints → Import Blueprint**
2. Paste the raw URL of `blueprints/automation/styrbar_smooth_dim.yaml` from this repo
3. Create a new automation from the blueprint
4. Select your Styrbar remote (as a device), the light entity, and the same light
   again as a ZHA device (needed for the native Move command)
5. Check the light's endpoint ID in **ZHA → Devices → your light → Zigbee Device
   Signature** if it isn't 1
6. (Optional) Tune dim speed, step size, floor, and the 6 preset colors

## Configuration options

| Option | Default | Description |
|---|---|---|
| Endpoint ID | 1 | Zigbee endpoint of the light that exposes Level Control |
| Dim-up speed (units/sec) | 40 | Native Move rate — higher = faster ramp |
| Brightness step per iteration (dim-down) | 3% | Step size for the down-dimming loop |
| Delay between steps (dim-down) | 150 ms | Time between repeat cycles |
| Transition per step (dim-down) | 0.1 s | Should stay ≤ delay to avoid reading stale state |
| Minimum brightness when dimming down | 1% | Floor — light never turns off via holding down |
| Max steps per hold-down (safety limit) | 60 | Safety cap in case a stop event is missed |
| Default color when turning on | 2700K | Native white temperature, applied on short press Up — independent of the presets below |
| Default brightness when turning on | 30% | Applied on short press Up |
| Turn-on transition | 0.5 s | Fade-in duration when turning on |
| Turn-off transition | 0.5 s | Fade-out duration when turning off |
| Color 1 | `[255, 147, 40]` | RGB |
| Color 2–4 | muted blue, sage green, lavender | Soft, non-saturated RGB colors |
| Color 5 (temp) | 2700K | Native white channel via `color_temp_kelvin` |
| Color 6 (temp) | 4000K | Native white channel via `color_temp_kelvin` |
| Color change transition | 1 s | Transition time when switching colors |
| Left/Right arrow `args[0]` values | 257 / 256 | Confirmed via Developer Tools → Events for this Styrbar unit |

## Notes

- Left/Right cycling works by matching the light's *current* `rgb_color` against the
  6 configured presets. If the light is currently on a color that isn't in the list
  (e.g. set manually via the Home app), the first short press will jump to color 1.

## License

MIT
