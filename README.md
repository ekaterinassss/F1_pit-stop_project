# Optimization of Pit Stop Strategies in Motorsport Using Telemetry and Simulation Data

## About the Project

Pit stop strategy is one of the key factors influencing race outcomes. It defines the order of tyre changes, compound selection, and resource allocation across the full race distance. Strategic errors can result in significant losses of time and track position, while optimal decisions provide a competitive advantage even when cars are technically comparable.

**Goal:** develop a decision support algorithm that enables:
- evaluation of the effectiveness of alternative pit stop strategies,
- modelling of tyre degradation, race events, and changing race conditions,
- analysis of the influence of external factors (safety car, weather, track evolution) on strategic decisions.

---

## Data

| Source | Coverage | Contents |
|---|---|---|
| [Kaggle F1 (1950–2024)](https://www.kaggle.com/datasets/rohanrao/formula-1-world-championship-1950-2020) | 2011–2024 | Lap times, pit stops, positions, drivers, constructors |
| [FastF1](https://theoehrly.github.io/Fast-F1/) | 2018–2024 | Tyre compounds |
| [Open-Meteo API](https://open-meteo.com/) | 2011–2024 | Temperature, humidity, precipitation, wind speed |
| [Kaggle F1 Race Events](https://www.kaggle.com/datasets/jtrotman/formula-1-race-events) | 2011–2024 | Safety car periods, red flags |

### Dataset Features

```
Race metadata:    year, date, round, circuitRef
Identifiers:      driver_code, constructor, raceId
Lap-level data:   lap, time_lap, pit, time_pit
Tyres & stints:   stint, tyre_life, tyre_compound, stint_time_delta
Position:         position
Race events:      safety_car, red_flag
Weather:          temperature, humidity, precipitation, wind_speed, weather_type
```

---

## Methodology

### Exploratory Data Analysis

The following directions were investigated:

- **Pit stop frequency** — dependence of pit stop count on circuit characteristics
- **Pit stop duration** — trends across seasons and differences between constructors
- **Position change** — relationship between pit stop timing and subsequent position gains or losses
- **Tyre degradation** — analysis of stint lengths and degradation profiles
- **Safety car impact** — how safety car deployment affects position change after a pit stop
- **Tyre compounds** — usage patterns across different compound types

### Hypothesis Testing

| # | Hypothesis | Result |
|---|---|---|
| H1 | Later pit stops → smaller position losses | Confirmed (p < 0.001) |
| H2 | Pit stop under SC → smaller position losses | Confirmed (p < 0.001) |
| H3 | Tyre degradation follows a nonlinear, concave curve | Confirmed (F-test, p < 0.001) |
| H4 | Faster teams → smaller position losses | Not confirmed (p = 0.066) |
| H5 | Optimal number of pit stops varies by circuit | Partially confirmed |

---

## Simulation Environment

A lap-by-lap race simulation reproduces race dynamics at the level of individual laps. Lap time is computed as the sum of five components:

```
Lap time = Base pace
         + Tyre degradation
         − Track evolution
         + Safety car penalty (if applicable)
         + Pit stop time loss (if applicable)
```

### Component Models

| Component | Method | Performance |
|---|---|---|
| Base pace | XGBoost (GroupShuffleSplit by raceId) | MAE = 5.3 s, R² = 0.64 |
| Track evolution | Logarithmic function | R² = 0.984 |
| Tyre degradation (Soft) | XGBoost, per-compound | MAE = 0.502 s/lap, R² = 0.500 |
| Tyre degradation (Medium) | XGBoost, per-compound | MAE = 0.512 s/lap, R² = 0.586 |
| Tyre degradation (Hard) | XGBoost, per-compound | MAE = 0.531 s/lap, R² = 0.522 |
| Safety car | Two-level XGBoost model | ROC-AUC = 0.62 |
| Pit stop duration | XGBoost | MAE = 1.7 s, R² = 0.42 |

**Key features of the environment:**
- Action masking: Pirelli rule (≥2 different dry compounds per race), maximum stint length constraint (95th percentile of historical data per compound)
- Probabilistic strategy evaluation via median across 50–300 Monte Carlo simulations

---

## Strategies

### 1. Baseline
A rule-based strategy: pit when `tyre_life` reaches the 60th percentile of historical stint lengths for the given circuit and compound. Fixed compound sequence: SOFT → MEDIUM → HARD.

### 2. Grid Search
Exhaustive enumeration of all reasonable pit stop strategies, each evaluated via Monte Carlo simulation (50 runs per strategy). Circuits are automatically classified by historical pit stop count:

| Category | Circuits |
|---|---|
| 1-stop (8 circuits) | Monaco, Miami, Jeddah, Imola, Sochi, Monza, etc. |
| 2-stop (22 circuits) | Silverstone, Spa, Bahrain, Baku, Suzuka, etc. |
| 3-stop (4 circuits) | Losail, Hockenheim, Interlagos, Sepang |

### 3. Lookahead Policy
An online algorithm: at each lap, the expected cost of continuing on current tyres is analytically compared against an immediate pit stop over a planning horizon H = min(laps remaining, 20). Adapts to safety car in real time by replacing `pit_time` with −10.0 under SC.

---

## Results

### Silverstone (2 pit stops, 100 simulations)

| Strategy | Median race time |
|---|---|
| Baseline | 81.75 min |
| Grid Search | **81.67 min** |
| Lookahead | **81.67 min** |

### Interlagos (3 pit stops, 50 simulations)

| Strategy | Median race time |
|---|---|
| Baseline | 96.36 min |
| Grid Search | **95.93 min** |
| Lookahead | **95.94 min** |

### Safety Car Scenario Analysis (Interlagos)

| Scenario | Baseline | Grid Search | Lookahead |
|---|---|---|---|
| No SC | 96.355 | **95.919** | 95.944 |
| Early SC (lap 10) | 98.046 | **97.614** | 97.895 |
| Mid SC (lap 30) | 98.131 | 98.060 | **97.976** |
| Late SC (lap 50) | 98.248 | **97.948** | 98.315 |


---
## Author

Submitted by: Ekaterina Sushko,
3rd-year student, "Data Science and Business Analytics" Programme  
Faculty of Computer Science, HSE University

Project Supervisor: Daria Bashminova  
Senior Lecturer, Faculty of Computer Science, HSE University
