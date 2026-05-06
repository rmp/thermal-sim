# Compute thermal simulator

A live, single-page thermodynamic simulator for compute systems. Models junction and heatsink temperatures over time given power draw, fan configuration, heatsink properties, and ambient conditions. Useful for sizing cooling, exploring heat-soak behaviour, and stress-testing fan-failure scenarios.

## Physical model

Two-node lumped capacitance. The junction (die + IHS) is treated as a small thermal mass coupled to the heatsink through a fixed conduction resistance `R_θ,jc`; the heatsink itself is a much larger thermal mass that dissipates to ambient through an airflow-dependent convection resistance `R_θ,sa`.

State variables: junction temperature `T_j` and heatsink temperature `T_s`. The governing ODEs are

```
C_j · dT_j/dt = P − (T_j − T_s) / R_θ,jc
C_s · dT_s/dt = (T_j − T_s) / R_θ,jc − (T_s − T_a) / R_θ,sa(CFM)
```

where:

- `P` is power dissipation in watts (5–500 W slider range).
- `T_a` is ambient air temperature.
- `R_θ,jc` is the junction-to-case thermal resistance in K/W (datasheet value, fixed per part).
- `R_θ,sa(CFM)` is the heatsink-to-ambient resistance, an empirical function of total airflow and fin area (see below).
- `C_j` is the junction-side heat capacity in J/K (small — die, solder, IHS).
- `C_s` is the heatsink heat capacity, computed from `mass × specific_heat`.

The two-node split is what produces realistic heat-soak behaviour: the small `C_j` causes a fast initial rise on power-on, while the much larger `C_s` produces the slow drift toward steady state over minutes.

### Steady-state algebraic answer

For any constant `P`, the asymptote is

```
T_j,ss = T_a + P · (R_θ,jc + R_θ,sa)
```

This is computed and displayed alongside the live trace as a sanity check on where the simulation is heading.

### Heatsink heat capacity

```
C_s = m · c_p
```

with `c_p` selected by the material dropdown:

| Material      | c_p (J/kg·K) | Notes                          |
| ------------- | ------------ | ------------------------------ |
| Aluminium 6061 | 897          | Default — most extruded sinks  |
| Copper C110    | 385          | Higher conductivity, lower c_p |
| Mild steel     | 490          | Rare in compute, useful baseline |

Mass slider: 50–2000 g, covering everything from passive low-profile sinks through to oversized server-grade tower coolers.

### Airflow → R_θ,sa model

`R_θ,sa` is modelled as exponential decay from a still-air passive value to a high-airflow asymptote, with surface area scaling:

```
area_scale  = (0.15 / A_sink) ^ 0.6
R_min       = 0.06 · area_scale
R_passive   = 4.00 · area_scale
R_θ,sa(CFM) = R_min + (R_passive − R_min) · exp(−CFM_total / 30)
```

- `CFM_total = n_fans · CFM_per_fan`.
- The exponent `0.6` on area roughly matches the empirical scaling seen in published heatsink performance curves — doubling the fin area gives meaningfully but sub-linearly lower resistance.
- `cfm0 = 30` is the characteristic airflow scale: most of the gain from passive → forced convection happens in the first 30–60 CFM, with diminishing returns above that.
- The asymptotes (4 K/W passive, 0.06 K/W saturated, both at 0.15 m² reference area) are calibrated to typical published curves for mid-sized aluminium tower coolers. They are deliberately representative rather than tied to a specific part.

### Time constants

The dominant time constant of the system is

```
τ_sink = C_s · R_θ,sa
```

This is what the user feels as "thermal mass" — for a 500 g aluminium sink at moderate airflow it sits around 60–120 s, matching real-world burst performance. The junction-side time constant `τ_j = C_j · R_θ,jc` is sub-second and governs the fast spike on power-on.

## Numerical integration

Forward Euler at fixed `dt = 50 ms`. Chosen because:

- The dominant time constants in the model (1 s on the junction side, 30+ s on the sink side) are both much larger than `dt`, so explicit integration is comfortably stable across the full slider range.
- It is trivially cheap, allowing fast-forward sim speeds up to 60× real-time without dropping frames.
- No implicit solve means no library dependency.

The integrator runs as many `dt` steps per animation frame as needed to match `simSpeed × wall_elapsed`, capped at 4000 steps/frame to prevent the loop stalling if the tab is backgrounded and resumes.

## History buffer

Temperature samples are pushed to a rolling history buffer, decimated to one sample per 0.25 simulated seconds, and pruned to the last 20 minutes of simulated time. The chart auto-scales its time window: 60 s → live span → cap at 600 s.

## UI architecture

Single self-contained widget. No build step, no framework. Vanilla JS + a hand-rolled `<canvas>` chart so it stays under 300 lines and renders smoothly during slider drags.

### Controls

**Heat source & component**

- Power (5–500 W)
- Ambient (0–50 °C)
- T_j,max (60–125 °C) — drawn as a dashed red threshold; junction trace shading turns red above it
- R_θ,jc (0.05–1.00 K/W) — datasheet thermal resistance
- C_j (5–200 J/K) — junction-side heat capacity

**Cooling system**

- Fans (0–6) and CFM/fan (0–200) — multiplied to give total airflow
- Heatsink mass (50–2000 g)
- Heatsink area (0.02–0.50 m²)
- Material dropdown (sets c_p)

**Run controls**

- Reset — clears state to ambient and zeros the clock
- Cut fans — instantly drops fan count to 0; useful for fan-failure scenarios
- Pause / Resume
- Sim speed: 1× / 5× / 20× / 60×

### Live readouts

Six metric tiles, all rounded to display precision:

- Junction T_j (warns at 90% of T_j,max, danger above)
- Heatsink T_s
- Steady-state T_j (algebraic asymptote)
- R_θ,sa (current airflow-dependent value)
- τ heatsink (current time constant)
- Total airflow

### Chart

`<canvas>` element, redrawn every frame. Renders:

- Junction trace (red)
- Heatsink trace (amber)
- Ambient (dashed gray reference)
- T_j,max (dashed red threshold)
- Pink wash above T_j,max when the junction trace breaches it

DPR-aware (sharp on Retina), CSS-variable-themed (light/dark mode automatic).

## Limitations & assumptions

Things deliberately not modelled, in roughly decreasing order of impact:

- **Fan curves.** Each fan is a constant CFM source. Real fans have static-pressure–vs–flow curves that interact with sink fin density. For sizing and trend work this is fine; for a specific cooler+fan pairing it under- or over-estimates depending on impedance.
- **Spatial gradients in the sink.** A real heatsink is not isothermal — the base near the die runs hotter than the fin tips. The lumped `C_s` averages this. The error grows for very large sinks at high `P`.
- **Radiation.** Ignored. Below ~80 °C surface temperature in compute enclosures it is a small fraction of total dissipation; included implicitly inside the empirical `R_passive` term for the still-air case.
- **TIM degradation, dust loading, ambient humidity.** All folded into the slider-set `R_θ,jc` and `R_θ,sa` calibration.
- **Power dependency on temperature.** Real silicon leaks more current as it heats, so `P` rises with `T_j`. Treated here as constant — for thermal headroom analysis this is conservative on the cooling side.
- **Throttling response.** Not modelled. The simulator shows what happens to an unthrottled load — `T_j` is allowed to climb past `T_j,max` so you can see how badly cooling is undersized. In a real system DVFS would clamp `P` once `T_j` approaches the limit.
- **Multiple heat sources.** Single-junction model. A board with CPU + GPU + VRMs sharing one airflow path needs a multi-node extension.

## File layout

Single HTML file. Streaming-safe structure:

```
<style>            (≈ 40 lines, kept minimal)
<div class="thermo">
  legend / chart-wrap / metrics / panels (controls)
<script>           (state, model, integrator, render loop, UI bindings)
```

No external dependencies. No network calls. Renders inline in the host chat widget; responsive at the 680 px chat width and collapses cleanly below 600 px.

## Extending

Likely extensions, in roughly decreasing order of usefulness:

1. **Throttling curve.** Trivial: scale `P` by `clamp((T_j,max − T_j) / margin, 0, 1)` once `T_j > T_j,max − margin`. Re-runs and traces would then show the throttling zone explicitly.
2. **Power profile.** Replace the constant `P` slider with a piecewise schedule (idle / burst / sustained) so users can model real workloads against the time-domain response.
3. **Multi-node extension.** Add a second junction (e.g. CPU + GPU) sharing the same airflow path, with separate `R_θ,jc` and `C_j` and a coupled `R_θ,sa` that depends on combined dissipation.
4. **Calibration mode.** Let users enter two `(CFM, R_θ,sa)` points from a real datasheet curve and back-fit the exponential model.
5. **Export.** Dump the history buffer as CSV for further plotting.
