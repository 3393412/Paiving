# PaiVing — Chatuchak Park Running Monitor

> Real-time environmental assessment for outdoor running at Chatuchak Park, Bangkok.
> Integrates indoor IoT sensor data with outdoor environmental APIs to compute
> a scientifically-grounded running condition score (0–100).

---

## 🎯 Project Overview

**Question answered:** *"Is it safe to go running at Chatuchak Park right now?"*

PaiVing combines two data sources:
- **Primary:** Indoor IoT sensor (PM2.5/PM10/Light, every 1 minute)
- **Secondary:** IQAir (AQI), OpenAQ (outdoor PM2.5), OpenWeatherMap (weather), UV API

The result is a single running score with full breakdown, trend analysis,
and 5-day forecast — all in one HTML file.

---

## 🚀 How to Run

### Open directly in browser
1. Download `dashboard.html`
2. Open it in any modern browser (Chrome, Firefox, Edge)
3. Enter your Node-RED host and prefix in the config bar
4. Press **⟳ Refresh**
### Default configuration
Host:   https://iot.cpe.ku.ac.th
Prefix: /red/b6710545717

## 📡 Data Sources

### Primary Source — Indoor IoT Sensor
| Field | Detail |
|---|---|
| Sensor | PM2.5 / PM10 / PM1.0 particulate matter + KY-018 light |
| Frequency | Every **1 minute** |
| Storage | `sensor_data` table on `iot.cpe.ku.ac.th` |
| Used for | Trend detection, Coefficient of Variation (CV%), indoor baseline |

### Secondary Sources — External APIs
| API | Data | Frequency |
|---|---|---|
| IQAir | AQI-US (NowCast), Temperature, Humidity | Every 1 hour |
| OpenAQ v3 | PM2.5 µg/m³ latest + 24h average (sensor 2076226) | Every 1 hour |
| OpenWeatherMap | Wind speed, Rain probability, 5-day forecast | Every 1 hour / 24 hours |
| currentuvindex.com | UV Index | Every 1 hour |

## 🏃 Running Condition Score

The core output of PaiVing is a **score from 0 to 100**, computed from
6 environmental factors using published scientific standards.

### How it works

The score starts at **100** and deducts penalties based on current conditions.

```
Score = max(0, 100 − Σ penalties)
```

### Scoring Factors

| # | Factor | Source | Standard Used | Max Penalty |
|---|---|---|---|---|
| 1 | AQI-US | IQAir | US EPA AQI categories | −45 pts |
| 2 | PM2.5 | OpenAQ | EPA breakpoints (µg/m³) | −10 pts |
| 3 | Temperature | IQAir | ACSM — tropical-adapted | −20 pts |
| 4 | Heat Stress | IQAir | Steadman (1979) Heat Index | −8 pts |
| 5 | UV Index | UV API | WHO UV Index categories | −15 pts |
| 6 | Rain | OWM | User-toggleable | −15 pts |

### Penalty Thresholds

**1 AQI-US** — primary factor (source: US EPA AQI, NowCast method)
| AQI Range | Category | Penalty |
|---|---|---|
| 0 – 50 | Good | 0 |
| 51 – 100 | Moderate | −10 |
| 101 – 150 | Unhealthy for Sensitive Groups | −20 |
| 151 – 200 | Unhealthy | −35 |
| > 200 | Very Unhealthy / Hazardous | −45 |

**2 PM2.5** — support factor (source: OpenAQ, real µg/m³)
| PM2.5 (µg/m³) | Category | Penalty |
|---|---|---|
| ≤ 12.0 | Good | 0 |
| 12.1 – 35.4 | Moderate | −3 |
| 35.5 – 55.4 | Unhealthy for Sensitive Groups | −6 |
| > 55.4 | Unhealthy | −10 |

> AQI + PM2.5 combined penalty is capped at **50 pts**.
> AQI and PM2.5 are **different metrics** — AQI is a dimensionless index
> computed via 12-hour NowCast weighting. PM2.5 is a raw physical
> concentration. They are never averaged together.

**3 Temperature** — tropical-adapted (ACSM guidelines, Bangkok context)
| Temperature | Penalty |
|---|---|
| < 30°C | 0 |
| 30 – 33°C | −5 |
| 34 – 36°C | −10 |
| 37 – 39°C | −15 |
| ≥ 40°C | −20 |

> Penalty starts at **30°C**, not 35°C, because Bangkok's daily high
> averages 30–35°C. Using temperate-climate thresholds would mean
> almost no temperature penalty is ever applied.

**4 Heat Stress** — humidity combined with temperature (Steadman 1979)
| Condition | Penalty |
|---|---|
| Humidity < 75% OR temp < 28°C | 0 |
| Humidity ≥ 75% AND temp ≥ 28°C | −3 |
| Humidity ≥ 80% AND temp ≥ 30°C | −5 |
| Humidity ≥ 85% AND temp ≥ 30°C | −8 |

> Humidity **alone** causes no penalty. The penalty only applies when
> high humidity and high temperature occur together — because heat stress
> occurs when sweat cannot evaporate, which requires both conditions
> simultaneously (Steadman, 1979; ACSM Heat Illness Position Stand, 2007).

**5 UV Index** — WHO Global Solar UV Index categories
| UV Index | Category | Penalty |
|---|---|---|
| 0 – 2 | Low | 0 |
| 3 – 5 | Moderate | −5 |
| 6 – 7 | High | −10 |
| 8 – 10 | Very High | −13 |
| ≥ 11 | Extreme | −15 |

**6 Rain Probability** — user-toggleable (some runners ignore rain)
| Rain Probability | Penalty |
|---|---|
| < 20% | 0 |
| 20 – 49% | −5 |
| 50 – 69% | −10 |
| ≥ 70% | −15 |

### Score Labels

| Score | Label | Recommendation |
|---|---|---|
| 80 – 100 | ✅ Excellent | Safe to run |
| 60 – 79 | 🟡 Good | Run with standard precautions |
| 40 – 59 | ⚠️ Moderate | Shorten run, hydrate frequently |
| 20 – 39 | 🔴 Poor | Avoid strenuous activity |
| 0 – 19 | ☠️ Hazardous | Do not run outdoors |
