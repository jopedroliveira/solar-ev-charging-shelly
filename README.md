# Solar Surplus EV Charging with ShellyProEM50

**Shelly Smart Home Challenge 2026 — Category: Scripting & Logic**

A Home Assistant automation that charges a Tesla exclusively from solar surplus, dynamically adjusting charging current every 2 minutes based on real-time energy data from a ShellyProEM50.

## The problem

With solar panels on the roof and an EV in the garage, the default behavior is wasteful: solar energy gets exported to the grid instead of being used at home. The car sits in the garage all day with a perfectly good battery, while the panels are pushing kilowatts into the grid that could be charging it.

Scheduled charging doesn't solve this — solar production is unpredictable. Clouds, seasons, and household consumption all change how much surplus is available at any given moment. You can't set a fixed schedule for something that shifts every few minutes.

The goal: **use every watt of solar surplus to charge the car, adjusting in real-time** — no energy wasted to the grid, no imports from the grid.

## The solution

A ShellyProEM50 monitors two energy channels:
- **Channel 0** — Solar panel production (also used for voltage reading)
- **Channel 1** — Grid net meter (positive = importing, negative = exporting)

Every 2 minutes, the automation calculates available surplus and adjusts the Tesla's charging current accordingly.

### How it works

```
surplus_W = (-grid_W) + car_W
target_amps = clamp(surplus_W / voltage, 5, 25)
```

The key insight: `car_W` (what the car is already consuming) must be added back to the surplus calculation. Without it, the automation would see the car's own consumption as household load and keep reducing the charging current in a feedback loop.

### Hysteresis

A naive implementation would start charging at 1150W and stop below 1150W — but solar surplus fluctuates constantly (clouds, appliances cycling on/off). This causes the charger to oscillate on/off every 2 minutes, which is bad for the car and wastes energy on repeated ramp-ups.

The solution: **different thresholds for starting and stopping**, creating a dead band where the system holds its current state:

- **Start charging:** surplus ≥ 1150W (~5A × 230V)
- **Stop charging:** surplus < 690W (~3A × 230V)
- **Dead band (690W–1150W):** if already charging, keep charging at minimum 5A. If not charging, don't start.

This means a passing cloud that drops surplus from 1500W to 800W won't stop the charge — only a sustained drop below 690W will.

### Charging modes

| Mode | Trigger | Behavior |
|------|---------|----------|
| **Solar** | Surplus ≥ 1150W (~5A × 230V) | Charges between 5-25A, adjusting every 2 min |
| **Emergency** | Battery < 40% | Charges at 25A from grid until 60% |
| **Idle** | Surplus < 690W (~3A × 230V) | Stops charging, waits for surplus to return |

### Surplus reminder

A companion automation notifies when there's solar surplus being exported to the grid (>1150W for 10+ minutes) but the car isn't plugged in — a gentle nudge to take advantage of free energy. Throttled to one notification per hour.

## Architecture

```
                    ┌─────────────────────┐
                    │    Solar Panels      │
                    └─────────┬───────────┘
                              │ DC
                    ┌─────────▼───────────┐
                    │      Inverter        │
                    └─────────┬───────────┘
                              │ AC
           ┌──────────────────┼──────────────────┐
           │                  │                   │
  ┌────────▼────────┐  ┌─────▼──────┐   ┌───────▼────────┐
  │  ShellyProEM50   │  │  House     │   │  Tesla Wall    │
  │  Ch0: Solar prod │  │  Loads     │   │  Connector     │
  │  Ch1: Grid meter │  │            │   │  (5-25A mono)  │
  └────────┬────────┘  └────────────┘   └───────▲────────┘
           │ WiFi                               │
  ┌────────▼──────────────────────────────────────────────┐
  │                  Home Assistant                        │
  │                                                        │
  │  Every 2 min:                                          │
  │  1. Read grid_W and voltage from ShellyProEM50         │
  │  2. Read current charger_amps from Tesla               │
  │  3. Calculate: surplus = (-grid_W) + car_W             │
  │  4. Calculate: target_amps = clamp(surplus/V, 5, 25)   │
  │  5. Set Tesla charging current to target_amps          │
  │                                                        │
  └────────────────────────────────────────────────────────┘
```

## Hardware

| Device | Role | Entity |
|--------|------|--------|
| **ShellyProEM50** | Energy monitoring | `sensor.shellyproem50_*_energy_meter_0_voltage`, `sensor.shellyproem50_*_energy_meter_1_power` |
| Tesla Wall Connector | EV charger (monophase, 5-25A) | `switch.carro_charge`, `number.carro_charge_current` |
| Tesla (vehicle) | Battery level, charging state, location | `sensor.carro_battery_level`, `sensor.carro_charging`, `device_tracker.carro_location` |

## Setup

### Prerequisites

- Home Assistant with [Shelly integration](https://www.home-assistant.io/integrations/shelly/) (native)
- ShellyProEM50 installed on the electrical panel
- Tesla integration (native or custom) providing charging controls
- Monophase charging setup (the automation assumes single-phase)

### Installation

1. **Create helpers** — Add the two `input_boolean` helpers from [`helpers.yaml`](helpers.yaml) via Settings → Helpers or in your `configuration.yaml`

2. **Import automations** — Copy the contents of [`automation_solar_ev_charging.yaml`](automation_solar_ev_charging.yaml) and [`automation_surplus_reminder.yaml`](automation_surplus_reminder.yaml) into your automations (via UI or `automations.yaml`)

3. **Adapt entity IDs** — Replace the entity IDs with your own:
   - `sensor.shellyproem50_*` → your ShellyProEM50 entities
   - `sensor.carro_*`, `switch.carro_*`, `number.carro_*` → your Tesla entities
   - `device_tracker.carro_location` → your Tesla device tracker

4. **Verify ShellyProEM50 channel mapping:**
   - Channel 0 should read solar production (used for voltage)
   - Channel 1 should read grid net meter (positive = importing, negative = exporting)

### Adapting to your setup

- **Three-phase charging:** Divide surplus by 3 for per-phase amps, adjust the 5-25A clamp to your charger's range
- **Different EV:** Replace Tesla entities with your charger's equivalents (most smart chargers expose current control)
- **Different energy monitor:** Any Shelly energy meter works — adjust entity IDs and verify the sign convention for import/export

## Results

- Solar energy that would be exported to the grid is used to charge the car instead
- The car charges almost exclusively from solar during sunny days
- Grid imports for EV charging reduced to emergency-only situations
- Hysteresis prevents start/stop oscillation during variable cloud cover
- The system is fully automatic — no manual intervention needed

## Files

| File | Description |
|------|-------------|
| [`automation_solar_ev_charging.yaml`](automation_solar_ev_charging.yaml) | Main automation — surplus calculation and charging control |
| [`automation_surplus_reminder.yaml`](automation_surplus_reminder.yaml) | Companion — reminds to plug in when surplus is available |
| [`helpers.yaml`](helpers.yaml) | Required `input_boolean` helpers |

## License

MIT
