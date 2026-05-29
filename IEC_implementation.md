# Indirect Evaporative Cooling (IEC) — Implementation Guide

## What IEC Is (Simplified)

The existing model has:
- **Direct evaporative / adiabatic cooling** (`PUE_WUE_AE_Chiller`) — water sprayed directly into supply air, adding moisture
- **Waterside economizer** (`PUE_WUE_Chiller_Watereconomier`) — cooling tower water pre-cools chilled water via a heat exchanger

**IEC is the middle ground**: outside air is wetted on the *secondary* side of a heat exchanger, cooling the *primary* (supply) air **without adding moisture to it**. This is how Google, Meta, and Microsoft cool most of their hyperscale facilities.

```
 PRIMARY AIR (dry side)          SECONDARY AIR (wet side)
 +-----------------------+       +-----------------------+
 | Server return air --->|--HX-->|<--- Outside air       |
 | (hot, recirculated)   | wall  |   + water spray       |
 | Cooled supply air <---|       |   Exhausted (hot,     |
 | (no moisture added)   |       |    humid) --->        |
 +-----------------------+       +-----------------------+
```

The key physics: primary air can approach the **wet-bulb temperature** of the secondary (outside) air, but its humidity ratio stays constant.

---

## Core Equations

There are only **5 key equations** to implement. All use variables already present in the model.

### 1. Wet-Bulb Effectiveness (the single most important parameter)

```
epsilon_wb = (T_primary_in - T_primary_out) / (T_primary_in - T_wb_secondary_in)
```

- `epsilon_wb` is a design parameter, typically **0.6-0.8** for commercial IEC units (up to 0.9+ for premium units)
- `T_wb_secondary_in` = wet-bulb of outside air (already computed with `HAPropsSI('Twb',...)`)

Rearranged to get **supply air temperature**:

```python
T_sa = T_ra - epsilon_wb * (T_ra - T_wb_oa)
```

where `T_ra` is the return air temperature (primary inlet) and `T_wb_oa` is the outdoor wet-bulb.

### 2. Cooling Capacity

```python
Q_IEC = m_sa * cp_air * (T_ra - T_sa)   # kW
```

This is the sensible cooling delivered. Since IEC doesn't change supply air humidity, there's no latent component on the primary side.

### 3. When IEC Is Sufficient vs. Needs a Backup Chiller

```python
if T_sa_iec <= T_sa_setpoint:
    # IEC handles full load
    Chiller_heat_removed = 0
else:
    # IEC partially cools, chiller trims the rest
    T_sa_actual = T_sa_setpoint
    Q_IEC_partial = m_sa * cp_air * (T_ra - T_sa_iec)  # what IEC can do
    Chiller_heat_removed = Q_total - Q_IEC_partial       # remainder
```

This is analogous to the existing economizer on/off logic but with a **continuous partial-cooling** mode.

### 4. Secondary Air Fan Power

IEC requires a separate fan to push outside air through the wet side:

```python
m_sec = m_sa * secondary_flow_ratio   # typically 0.5-1.0x primary flow
density_oa = 1 / HAPropsSI('Vha', 'T', T_oa+273.15, 'RH', RH_oa/100, 'P', P_oa)
Power_Fan_sec = (m_sec / density_oa) * Fan_Pressure_sec / Fan_e_sec / 1000  # kW
```

This is new power overhead that the existing direct-AE model doesn't have.

### 5. Water Consumption (Secondary Side Only)

```python
# Heat rejected on wet side ~ heat removed from primary side
Q_rejected = Q_IEC + Power_Fan_sec  # include fan heat

# Evaporation
L_v = 2500.5  # latent heat of vaporization, kJ/kg (simplified)
Water_evap = Q_rejected / L_v  # kg/s

# Drift (mist carryover) -- typically 0.001-0.01% of recirculation flow
Water_drift = Water_evap * drift_fraction  # e.g., 0.005

# Blowdown (to prevent mineral buildup)
Water_blowdown = max(Water_evap / (CC - 1) - Water_drift, 0)

# Total
Water_total = Water_evap + Water_drift + Water_blowdown  # kg/s
WUE_IEC = Water_total * 3600 / Power_IT  # L/kWh
```

This is essentially the same cooling tower water model already in `Cooling_Tower()`, but applied to the IEC's secondary air loop.

---

## Proposed Function Signature

Following the existing pattern in `simulation_funs_DC.py`:

```python
def PUE_WUE_IEC_Chiller(w):
    """IEC with backup water-cooled chiller (hyperscale DC)"""
    Power_IT = 1
    T_oa        = w[0]   # outdoor dry-bulb, C
    RH_oa       = w[1]   # outdoor RH, %
    P_oa        = w[2]   # atmospheric pressure, Pa
    UPS_e       = w[3]   # UPS efficiency
    PD_lr       = w[4]   # power distribution loss ratio
    L_percentage= w[5]   # lighting as fraction of IT
    delta_T_air = w[6]   # supply-to-return air delta_T, C
    Fan_Pressure_CRAC = w[7]
    Fan_e_CRAC  = w[8]
    SHR         = w[9]   # sensible heat ratio

    # --- IEC-specific parameters ---
    epsilon_wb  = w[10]  # wet-bulb effectiveness (0.6-0.8)
    sec_flow_ratio = w[11]  # secondary/primary air flow ratio
    Fan_Pressure_sec = w[12]  # secondary fan pressure, Pa
    Fan_e_sec   = w[13]  # secondary fan efficiency
    drift_frac  = w[14]  # drift fraction
    CC_iec      = w[15]  # concentration cycles for IEC water

    # --- Backup chiller parameters (same as existing) ---
    AT_CT       = w[16]
    Chiller_load= w[17]
    delta_T_water = w[18]
    Pump_Pressure_CW = w[19]
    Pump_e_CW   = w[20]
    delta_T_CT  = w[21]
    Pump_Pressure_CT = w[22]
    Pump_e_CT   = w[23]
    Windage_p   = w[24]
    CC          = w[25]
    Fan_Pressure_CT = w[26]
    Fan_e_CT    = w[27]
    LGRatio     = w[28]
    T_up        = w[29]
    T_lw        = w[30]
    dp_up       = w[31]
    dp_lw       = w[32]
    RH_up       = w[33]
    RH_lw       = w[34]
    pcop        = w[35]
```

---

## Implementation Logic (Pseudocode)

```python
    # --- shared helpers (already in existing code) ---
    Fan_Power = lambda m, rho, dP, eff: m/rho * dP/eff / 1000
    Pump_Power = lambda m, dP, eff, rho: dP*m / (1000*eff*rho)

    # --- total cooling load ---
    Q = Power_IT + (Power_IT/UPS_e - Power_IT) \
      + (Power_IT/(1-PD_lr) - Power_IT) + Power_IT*L_percentage

    # --- psychrometric properties of outside air ---
    Twb_oa = HAPropsSI('Twb','T',T_oa+273.15,'RH',RH_oa/100,'P',P_oa) - 273.15
    density_oa = 1/HAPropsSI('Vha','T',T_oa+273.15,'RH',RH_oa/100,'P',P_oa)

    # --- IEC supply air temperature ---
    T_sa_setpoint = max(T_up, T_lw)      # desired supply temp
    T_ra = T_sa_setpoint + delta_T_air    # return air temp
    T_sa_iec = T_ra - epsilon_wb * (T_ra - Twb_oa)  # what IEC can achieve

    # --- determine operating mode ---
    if T_sa_iec <= T_sa_setpoint:
        # IEC handles full load -- supply at setpoint
        T_sa = T_sa_setpoint
        Q_IEC = Q  # IEC covers everything
        Chiller_heat_removed = 0
    else:
        # IEC can't reach setpoint -- partial IEC + chiller trim
        T_sa = T_sa_setpoint
        Q_IEC = max(Q * (T_ra - T_sa_iec)/(T_ra - T_sa_setpoint), 0)
        Q_IEC = min(Q_IEC, Q)
        Chiller_heat_removed = Q - Q_IEC

    # --- primary air flow ---
    m_sa = Q / (1.01 * delta_T_air)
    d_sa = HAPropsSI('W','T',T_sa+273.15,'RH',50/100,'P',P_oa)  # humidity unchanged
    density_sa = 1/HAPropsSI('Vha','T',T_sa+273.15,'W',d_sa,'P',P_oa)

    # --- latent load (chiller only, IEC doesn't dehumidify) ---
    if Chiller_heat_removed > 0:
        Q_latent = Chiller_heat_removed/SHR - Chiller_heat_removed
        Chiller_heat_removed += Q_latent
        hd_amount = max(Q_latent/2266, 0)
    else:
        Q_latent = 0
        hd_amount = 0

    # === POWER COMPONENTS ===
    # CRAC fan (primary)
    Power_Fan_CRAC = Fan_Power(m_sa, density_sa, Fan_Pressure_CRAC, Fan_e_CRAC)

    # IEC secondary fan (NEW -- not in existing models)
    m_sec = m_sa * sec_flow_ratio
    Power_Fan_sec = Fan_Power(m_sec, density_oa, Fan_Pressure_sec, Fan_e_sec)

    # Chiller + cooling tower (reuse existing logic, only when needed)
    if Chiller_heat_removed > 0:
        COP = COP_gp.predict(...) * (1 + pcop)
        Power_Chiller = Chiller_heat_removed / COP
        # ... pumps, CT fan (same as existing code)
    else:
        Power_Chiller = 0
        # ... zero out chiller-related pumps

    # === PUE ===
    PUE = sum(all_power_components) / Power_IT

    # === WATER COMPONENTS ===
    # IEC evaporation (secondary side)
    Water_IEC_evap = Q_IEC / 2500.5          # kg/s
    Water_IEC_drift = Water_IEC_evap * drift_frac
    Water_IEC_blowdown = max(Water_IEC_evap/(CC_iec-1) - Water_IEC_drift, 0)

    # Cooling tower water (only if chiller active)
    # ... reuse existing Cooling_Tower() function

    # === WUE ===
    Water_total = (Water_IEC_evap + Water_IEC_drift + Water_IEC_blowdown
                   + hd_amount + CT_water_components)
    WUE = Water_total * 3600 / Power_IT

    return PUE, WUE
```

---

## Key Differences from Existing Models

| Aspect | Existing AE model | Proposed IEC model |
|--------|-------------------|-------------------|
| **Moisture in supply air** | Changes (humidification/dehumidification) | Unchanged -- dry heat exchange |
| **Limiting temperature** | Outdoor dry-bulb (direct mix) | Outdoor **wet-bulb** (evaporative limit) |
| **Humidity control logic** | Complex 3-regime air-side economizer | Not needed -- primary air is sealed loop |
| **Secondary fan** | None | Required -- new power component |
| **Water consumption source** | Adiabatic spray into supply + CT | IEC secondary evaporation + CT |
| **Partial cooling** | Binary (AE on/off) | Continuous (IEC always provides some cooling) |

---

## New Input Parameters Needed

| Parameter | Symbol | Typical Range | Unit |
|-----------|--------|---------------|------|
| Wet-bulb effectiveness | `epsilon_wb` | 0.6-0.9 | -- |
| Secondary/primary flow ratio | `sec_flow_ratio` | 0.5-1.2 | -- |
| Secondary fan pressure | `Fan_Pressure_sec` | 200-600 | Pa |
| Secondary fan efficiency | `Fan_e_sec` | 0.5-0.7 | -- |
| Drift fraction | `drift_frac` | 0.001-0.01 | -- |
| IEC concentration cycles | `CC_iec` | 3-10 | -- |

---

## Why This Is Simpler Than the Existing Air-Side Economizer

The `Air_side_economizer()` function (lines 21-234 of simulation_funs_DC.py) has complex branching across temperature and humidity regimes. IEC eliminates most of that because:

1. **No humidity regime logic** -- primary air humidity doesn't change
2. **No humidification/dehumidification** -- no `HD_use`, `DHD_use`, `Steam_use` flags
3. **Single equation for supply temperature** -- `T_sa = T_ra - epsilon_wb * (T_ra - T_wb_oa)`
4. **Graceful degradation** -- as outdoor wet-bulb rises, IEC delivers less cooling continuously rather than switching on/off

The tradeoff is the extra secondary fan power and the fact that IEC effectiveness is a design parameter you'd need to characterize for specific equipment.

---

## Recommended Next Steps

1. **Add the function** `PUE_WUE_IEC_Chiller()` to `simulation_funs_DC.py` following the pattern above
2. **Reuse** `Cooling_Tower()` and `COP_gp` for the backup chiller path
3. **Validate** against published results -- in mild-dry climates (e.g., The Dalles, OR) IEC should show lower PUE than chiller-only and lower WUE than direct adiabatic
4. **Optional enhancement**: add a dry-mode bypass where, when `T_oa < T_sa_setpoint`, the IEC operates without water (dry-bulb effectiveness ~0.5-0.7), saving water in cold weather

---

## References

- [EnergyPlus Evaporative Coolers Engineering Reference](https://bigladdersoftware.com/epx/docs/9-6/engineering-reference/evaporative-coolers.html)
- [Facebook StatePoint Liquid Cooling System](https://engineering.fb.com/2018/06/05/data-center-engineering/statepoint-liquid-cooling/)
- [Research status of evaporative cooling in data centers (ScienceDirect)](https://www.sciencedirect.com/science/article/pii/S2666123321000544)
- [Data centre water consumption (Nature)](https://www.nature.com/articles/s41545-021-00101-w)
- [DOE: Cooling Water Efficiency for Federal Data Centers](https://www.energy.gov/cmei/femp/cooling-water-efficiency-opportunities-federal-data-centers)
- Lei, N. et al. "Climate- and technology-specific PUE and WUE estimations for U.S. data centers using a hybrid statistical and thermodynamics-based approach" Resources, Conservation and Recycling (2022). https://doi.org/10.1016/j.resconrec.2022.106323
