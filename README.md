# Section 2. Practical Assignment 1.9

The `ThirdPersonMap` contains one `BP_TrafficLightsManager` and four `BP_TrafficLight` actors. Each traffic light consists of three `BP_Lamp` actors: red, yellow, and green.

The manager controls signal changes and group sequencing. The active group goes through the `Go → Warning → Stop` phases, after which the turn passes to the next group.

## Configuration and Sequence

The group range is configured through the `MinGroup` and `MaxGroup` parameters on the manager. The current setup uses groups `0` and `1`:

- `TrafficLight_0` and `TrafficLight_1` belong to group 0.
- `TrafficLight_2` and `TrafficLight_3` belong to group 1.

Signal colors and durations are stored in the manager.

| Signal | Color (Linear Color) | Duration |
|---|---|---|
| Green | `(0, 1, 0, 1)` | 3 s |
| Yellow | `(1, 0.85, 0, 1)` | 1 s |
| Red | `(1, 0, 0, 1)` | 5 s total per traffic light |

The red duration refers to the total time a traffic light displays red during a complete cycle. It is not the value of the individual `DurationByColor[Red]` timer: the actual red duration depends on the group switching sequence.

For the current group range of 0–1, the cycle is as follows:

| Time | Group 0 | Group 1 |
|---|---|---|
| 0–3 s | Green | Red |
| 3–4 s | Yellow | Red |
| 4–5 s | Red | Yellow |
| 5–8 s | Red | Green |
| 8–9 s | Red | Yellow |
| 9–10 s | Red | Red |

The cycle then repeats, starting with the green signal for group 0.

Only one lamp is illuminated on each traffic light. The lamp matching the active signal receives its corresponding color, while the other two lamps are set to the dark color `(0.02, 0.02, 0.02, 1)`.

## Blueprint Assets

| Asset | Purpose |
|---|---|
| `Content/Blueprints/BP_Lamp` | Lamp actor and color updates |
| `Content/Blueprints/BP_TrafficLight` | Traffic light, group assignment, and lamp management |
| `Content/Blueprints/BP_TrafficLightsManager` | Signal configuration and cycle management |
| `Content/Blueprints/E_LightColor` | Signal enumeration: Red, Yellow, Green |
| `Content/Blueprints/E_TrafficLightState` | Phase enumeration: Go, Warning, Stop |
| `Content/Blueprints/S_LightColor` | Color settings structure containing the `ColorMap` map |
| `Content/Blueprints/Materials/M_TrafficColor` | Lamp material with a vector parameter named `Color` |
| `Content/Blueprints/Materials/M_TrafficColorInstance` | Material instance used by the lamp sphere |

## Implementation

### BP_Lamp

Contains a sphere component named `Lamp` and a `Signal` variable of type `E_LightColor`, marked as `Instance Editable`. The actor has the `Lamp` tag.

The `ChangeColor(Color)` function creates a dynamic material instance and sets its `Color` parameter. The lamp is responsible only for displaying the color it receives.

### BP_TrafficLight

Contains a `Group` variable (`Integer`, `Instance Editable`) and a `Lamps` array.

In the `Construction Script`, child actors with the `Lamp` tag are cast to `BP_Lamp` and added to the array.

The traffic light has three child actors using the `BP_RedLamp`, `BP_YellowLamp`, and `BP_GreenLamp` classes, corresponding to the Red, Yellow, and Green signals.

The `ChangeColor(ActiveSignal, Color)` event iterates through `Lamps`. The lamp whose `Signal` matches `ActiveSignal` receives the specified color. The remaining lamps receive the dark color `(0.02, 0.02, 0.02, 1)`.

### BP_TrafficLightsManager

The main settings are exposed through `Instance Editable`:

- `MinGroup`, `MaxGroup` — inclusive group range.
- `ActiveGroup` — the currently active group.
- `CurrentState` — current phase, initially `Go`.
- `CurrentColor` — current signal, initially `Green`.
- `ColorSettings.ColorMap` — mapping between signals and colors.
- `DurationByColor` — signal durations.
- `FallbackInterval` — default duration when a signal is missing from `DurationByColor`.
- `FallbackColor` — default color when a signal is missing from `ColorMap`.

The `Apply Current Colors` function finds all `BP_TrafficLight` actors and updates their signals. The active group receives the current signal and color, while all other groups receive red.

On `BeginPlay`, the manager initializes the active group and state, applies the initial colors, and starts a one-shot timer.

The timer triggers the `Recolor` event, which advances the phase:

1. `Go` → `Warning`: the signal changes to yellow.
2. `Warning` → `Stop`: the turn passes to the next group. If the group index exceeds `MaxGroup`, it wraps around to `MinGroup` and the signal changes to red.
3. `Stop` → `Go`: the signal changes to green.

After each transition, the manager calls `Apply Current Colors` and starts the next one-shot timer. The timer duration is taken from `DurationByColor` using `CurrentColor`, or from `FallbackInterval` if the corresponding entry is missing.
