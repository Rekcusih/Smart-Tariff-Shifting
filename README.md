# Smart Tariff Shifting — Day-Ahead Household Electricity Forecasting and Bill/Carbon Analysis
**SDG 7 (Affordable and Clean Energy) · SDG 13 (Climate Action)**

## Project overview
Households usually pay for electricity without knowing *when* it is expensive or *when* the grid is clean. This capstone project builds a **day-ahead** household electricity forecasting model and combines it with Turkey's **three-time tariff (EPDK)** and **hourly grid data (EPİAŞ)** to advise households on when to shift flexible consumption, with an estimated effect on the bill and on the carbon intensity of the electricity used.

**What makes the project different.** It is not a next-hour (short-term) forecaster. It focuses on **day-ahead (24-hour) forecasting and bill optimization**: tomorrow's hourly consumption is forecast at the beginning of the day, priced under the three-time tariff, and shifting recommendations are evaluated against hourly grid price and carbon signals.

**SDG alignment**
- **SDG 7, Target 7.3** (energy efficiency): better planning of household electricity use and lower bills.
- **SDG 13** (climate action): we investigate whether shifting to cheaper hours also means cleaner hours. In our Turkish 2025 data it **does not necessarily** (see *Key finding*); carbon benefits therefore have to be designed for explicitly, not assumed.

Assignment 1 covers project selection, data collection/cleaning and exploratory data analysis (EDA). The forecasting model and user-facing tool are planned for the next stage.

## Team — Group 7
- Burak Arslan 22061025
- Buğra Karatepe 22061032
- Zeynep Erva Akdoğan 22061049
- Zeynep Şura Çetin 22069030

## Datasets
| Source | Used for | Details |
|---|---|---|
| **London Smart Meters** (UK Power Networks, *Low Carbon London*) | Household consumption patterns, forecasting | 5,566 households, half-hourly readings, 23 Nov 2011 – 28 Feb 2014, hourly weather, household metadata. Kaggle: https://www.kaggle.com/datasets/jeanmidev/smart-meters-in-london |
| **EPİAŞ Transparency Platform** | Cleanest and cheapest grid hours in Turkey | Hourly generation by source and day-ahead market clearing price (PTF), 1 Jan – 31 Dec 2025 (8,760 hours). https://seffaflik.epias.com.tr (accessed with the `eptr2` Python client) |
| **EPDK three-time tariff** | Bill calculation | Day 06:00–17:00, peak 17:00–22:00, night 22:00–06:00; prices in `epdk_three_time_tariff.csv` |

Licences: check and cite the licence shown on each source page. The earlier single-household UCI dataset is no longer used.

> **Tariff values must be verified.** Prices in `epdk_three_time_tariff.csv` were compiled from secondary web sources (2026 tables) and must be checked against the official EPDK table for the current quarter before results are reported.

### How the sources are combined
London households and the Turkish grid are **not** matched day by day (different country and period). They are combined at the **hour-of-day** level: household consumption profile (London) × tariff prices (EPDK) × price/carbon signals (EPİAŞ). See *Limitations*.

## Data preparation
**London Smart Meters** (processed block by block):
- Timestamps converted to datetime; readings converted to numeric (text such as `Null` becomes missing).
- Duplicate `(household, timestamp)` rows dropped; negative readings set to missing.
- Missing readings are **not** imputed or replaced with zero.
- Hourly kWh = sum of the two half-hour readings, **only when both are present**.
- A random panel of 500 households (seed 42) is kept as an hourly dataset for the forecasting stage.

**EPİAŞ:** timestamps converted to `Europe/Istanbul`, duplicates dropped, missing hours reported and left missing. Hourly **carbon intensity (gCO₂e/kWh)** is the generation-weighted average of life-cycle emission factors per source (IPCC AR5 medians; fuel oil/naphtha approximate). It is a screening metric for ranking hours, not an official inventory.

**EPDK tariff:** each hour of the day is assigned to day / peak / night and a price per kWh.

## EDA results
| Item | Result |
|---|---|
| Households processed | 5,566 |
| Raw half-hourly rows | 167,817,021 |
| Missing (null) readings | 5,560 |
| Duplicate rows / negative readings | 0 / 0 |
| Mean missing-hour ratio per household | about 0.5% (19 households above 20%, 10 above 50%) |
| Mean consumption per household | 0.43 kWh/h (median 0.35, maximum 4.22) |
| Panel (hourly, 500 households) | about 7.5 million hourly rows |

**Daily pattern (Fig. 1).** Consumption is lowest around 03:00–04:00 (about 0.22 kWh/h) and highest at 19:00 (about 0.64 kWh/h) on both weekdays and weekends. Weekdays have a morning bump at 08:00; weekends have higher midday consumption. In the 500-household weekday profile, **44% of daily kWh falls in the day period, 31% in the peak period and 25% at night.**

**Weather (Fig. 4).** Daily mean consumption falls clearly as temperature rises (about 0.9 kWh/h near 0 °C to about 0.45 kWh/h above 20 °C), so temperature is a useful forecasting feature.

![Hourly consumption profile](SDG7%20Assignment1/outputs/figures/fig1_hourly_profile.png)
![Temperature vs consumption](SDG7%20Assignment1/outputs/figures/fig4_temperature_vs_kwh.png)

**Turkish grid, 2025 (Fig. 5–6).** Estimated mean carbon intensity is 442 gCO₂e/kWh (range 233–591) and the mean low-carbon share is 39%.

| Hour group | Estimated carbon intensity | PTF |
|---|---|---|
| Cleanest: 18:00–20:00 | about 408–409 gCO₂e/kWh | highest (about 3,150–3,250 TL/MWh) |
| Dirtiest: 04:00–06:00 | about 479–481 gCO₂e/kWh | medium (about 2,530–2,720 TL/MWh) |
| Cheapest PTF: 10:00–14:00 | about 432–442 gCO₂e/kWh | lowest (12:00: about 1,501 TL/MWh) |

![Turkey carbon intensity and price by hour](SDG7%20Assignment1/outputs/figures/fig5_turkey_carbon_price_by_hour.png)

## Key finding: cheap hours are not automatically clean hours
In the 2025 data the grid is on average **cleanest in the evening (about 16:00–21:00), which overlaps the expensive peak tariff window (17:00–22:00)**, and dirtiest in the early morning, which is inside the cheap night tariff window. Consequently, an illustrative scenario that moves 20% of peak-hour consumption (assumption) gives:

| Target hours | Bill change | Estimated carbon change |
|---|---:|---:|
| Price-only (00:00–05:00) | −6.03% | +0.81% |
| Price + carbon balanced (00, 01, 15, 16, 22, 23) | −5.21% | +0.34% |

Bill savings are possible under the assumption, but they do not automatically reduce carbon; moving load out of the evening can even raise estimated emissions. These are assumption-based scenarios on average-emission estimates, **not measured savings**. The next stage should treat the bill/carbon trade-off explicitly (e.g. user-selectable weights, or recommendations such as shifting from the evening peak to early-afternoon hours) rather than assume both goals align.

## Planned model and tool (next stage)
- **Forecast:** day-ahead hourly consumption with a Random Forest baseline (hour, weekday, month, weekend/holiday, same-hour lags at 1, 2, 3 and 7 days, previous-day mean/max, temperature), evaluated on a **chronological split** against previous-day and previous-week baselines. An earlier single-household prototype (UCI data) informed this design and is archived.
- **Optimizer:** price the forecast under the three-time tariff and recommend moving flexible load to hours chosen by a user-selected balance of price and grid carbon.
- **Output:** estimated bill change and estimated carbon change per recommendation.

## Repository files
| File | Purpose |
|---|---|
| `README.md` | Project description |
| `SDG7 Assignment1/SDG7_Assignment1_EDA.ipynb` | Data download, cleaning and EDA (Google Colab) |
| `SDG7 Assignment1/epdk_three_time_tariff.csv` | Tariff windows and prices (verify before use) |
| `SDG7 Assignment1/Concept_Note.md` | Concept note text for the template |
| `SDG7 Assignment1/outputs/` | Figures and cleaned outputs exported by the notebook |
| `archive_uci_prototype/` | Earlier single-household (UCI) prototype, reference only |

The 500-household hourly panel (`london_hourly_panel_sample.csv.gz`, about 36 MB) is regenerated by the notebook and is not stored in the repository.

## How to run in Google Colab
1. Open https://colab.research.google.com/ and upload `SDG7_Assignment1_EDA.ipynb`.
2. Upload `epdk_three_time_tariff.csv` to the Colab Files panel (`/content/`).
3. Run the cells from top to bottom. London data is downloaded with `kagglehub`; if Kaggle asks for a login, download the ZIP manually and unzip it to `/content/london_smart_meters`.
4. For the Turkish grid section, register at EPİAŞ and enter your credentials when prompted (never commit them). Set `USE_EPIAS = False` to skip it.
5. `MAX_BLOCKS = 10` gives a quick test run; `None` processes all households (about 10 GB of half-hourly files).

## Limitations and responsible use
- London consumption (2011–2014, UK) is used to model household patterns; Turkish households differ in climate, heating/cooling and routines. A Turkish smart-meter dataset would be needed to validate the tool for Turkish households.
- Turkish grid data (2025) and London data (2011–2014) cover different periods; they are combined at hour-of-day level only.
- Carbon intensity is an estimate from the average generation mix; imports/exports and marginal emissions are not modelled.
- Tariff prices change quarterly and must be updated; taxes and fees are excluded from the stored prices.
- The shifting scenario depends on an **assumed** flexible share of consumption. Bill or carbon savings have not been demonstrated on real households.
- Some London households have large gaps and should be excluded before modelling.
- Household consumption data can reveal personal routines and must be handled privately in any future user-facing tool.

## References
- UK Power Networks, *SmartMeter Energy Consumption Data in London Households* (London Datastore); Kaggle mirror `jeanmidev/smart-meters-in-london`.
- EPİAŞ Transparency Platform: https://seffaflik.epias.com.tr
- EPDK tariff tables: https://www.epdk.gov.tr
- United Nations — Goal 7: https://sdgs.un.org/goals/goal7 · Goal 13: https://sdgs.un.org/goals/goal13
