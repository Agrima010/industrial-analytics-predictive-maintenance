# Predictive Maintenance of Railway Air Compressors

Predictive maintenance and anomaly detection for railway air compressors using the MetroPT-3 dataset (Metro do Porto).

Seminar project for **Industrial Analytics (IBS6), WiSe 2026/27**, Hochschule Furtwangen (HFU). **Group 4.**

| Member | Role | Main areas |
|---|---|---|
| Juliana | Data Engineer | Dataset sourcing and access, structure, data quality, preparation |
| Agrima | Data Scientist | ML feasibility, exploratory analysis, models, evaluation, experiments |
| Munira | Business Analyst | Problem definition, stakeholders, value proposition, KPIs, scope, presentations, coordination |

The full business report, with figures and a glossary, is in [docs/index.html](docs/index.html).

---

## Contents

1. [Project definition](#1-project-definition)
2. [Business understanding](#2-business-understanding)
3. [Research question](#3-research-question)
4. [Objectives](#4-objectives)
5. [Dataset](#5-dataset)
6. [Approach](#6-approach)
7. [Success criteria and KPIs](#7-success-criteria-and-kpis)
8. [Scope and limitations](#8-scope-and-limitations)
9. [Milestones](#9-milestones)
10. [Repository structure](#10-repository-structure)
11. [Getting started](#11-getting-started)
12. [References](#12-references)

---

## 1. Project definition

| | |
|---|---|
| **Domain** | Manufacturing / rail operations: *Operations & Service* in Gröger's industrial value-creation cycle; production and after-sale service in Liu's supply-chain view |
| **Topic** | Predictive maintenance of railway air compressors to improve train availability |
| **Asset** | The Air Production Unit (APU) of a Metro do Porto train |
| **Data** | MetroPT-3: 15 sensor signals from one APU, February to September 2020 |
| **Method** | Unsupervised anomaly detection, evaluated against the operator's failure reports |

Many systems on a metro train run on compressed air, including the brakes, the doors and the air-spring suspension. The air comes from the **Air Production Unit (APU)** on the roof of each vehicle. An electric motor drives a compressor, and the air is cleaned, dried and stored in reservoirs. The compressor runs in a cycle: when reservoir pressure drops below about 8.2 bar it starts under load, refills to about 10 bar, then idles or stops.

There is **no backup APU**. When it fails, the train has to be taken out of service at once (Veloso et al., 2022). The most common fault is an **air leak**. A leak usually starts small, and the compressor makes up for the lost air by running longer and more often. Eventually it cannot keep up and pressure falls below safe levels. This project tests whether that gradual change in the sensor data can be detected early enough to act on.

## 2. Business understanding

### The problem

A metro operator runs two operations that depend on each other:

- **Train operations** make sure scheduled trips happen.
- **Maintenance operations** inspect and repair trains so they are available for those trips.

When an APU fails in service, the two collide. A failure leads to an interrupted journey, cancelled trips afterwards, a train out of the fleet, emergency work or overtime in the workshop, and lost passenger trust. Metro do Porto reported more than 170 trips cancelled for this kind of reason in 2017 (Veloso et al., 2022). Fixed alarm thresholds do not help much here, because they only fire once pressure is already too low.

### Maintenance strategies

| Strategy | Principle | Benefit | Drawback |
|---|---|---|---|
| Reactive / corrective | Repair after it breaks | No wasted parts or inspections | Every failure is a surprise in service |
| Preventive | Repair on a fixed schedule | Fewer surprises | Healthy parts replaced too early; failures between visits still missed |
| **Predictive (our focus)** | **Repair when the data shows drift** | **Repairs fit into depot time; parts are used for their full life** | **Only works if warnings come early and can be trusted** |

### Where the value lies

Predictive maintenance only pays off if the warning comes **early enough to act on**. Workshops plan work in blocks of hours or days, so a warning given a minute before failure is useless. A warning that fires for no reason is also useless, because planners learn to ignore it. The business value depends on getting both right: enough lead time and few false alarms.

### Stakeholders

| Stakeholder | Interest |
|---|---|
| Passengers | Fewer cancelled trips and forced changes of train |
| Train operations (dispatchers, drivers) | Enough healthy trains to run the timetable |
| Maintenance team (planners, technicians) | Early, specific alerts with few false alarms |
| Fleet & asset management | Cost, availability, spare parts, depot capacity, reliability (MTBF, MTTR) |

## 3. Research question

> **Can an anomaly-detection model warn early enough, before compressor degradation becomes operationally critical, for maintenance to be planned instead of done reactively?**

The key measure is **lead time**: the time between the first sustained alarm and the start of a failure window. Metro do Porto requires a failure to be detected **at least 2 hours** before the train becomes non-operational, so that it can be taken off the line safely (Veloso et al., 2022; Davari et al., 2021). We use 2 hours as the minimum useful lead time.

## 4. Objectives

**Business objectives**

- Give the maintenance team at least 2 hours' warning before an APU air-leak failure.
- Turn unplanned failures in service into planned depot interventions.
- Keep false alarms low enough that planners trust and act on the alerts.

**Analytical objectives**

1. **Describe** how the APU normally operates: operating states, duty cycles, and signal behaviour around the reported failures.
2. **Diagnose** which signals change before and during air leaks, and explain why using the APU's physics.
3. **Predict** failures with unsupervised anomaly-detection models, compared against a simple statistical baseline.
4. **Prescribe** a maintenance decision rule (alert threshold plus persistence rule) and assess the cost of false alarms against missed failures.

**Course objectives**

- Apply lecture concepts (Gröger's analytics levels and the industrial value-creation cycle) to a real industrial dataset.
- Deliver reproducible experiments, source code and two presentations, with each student's contributions clearly attributable.

## 5. Dataset

**MetroPT-3**, published by INESC TEC and the University of Porto on the [UCI Machine Learning Repository](https://archive.ics.uci.edu/dataset/791/metropt%203%20dataset).

| Property | Value |
|---|---|
| Rows | 1,516,948 |
| Period | 2020-02-01 00:00 to 2020-09-01 03:59 (213 days) |
| Sampling | ~10 s in this CSV (the original logger ran at 1 Hz) |
| Signals | 15: 7 analog (pressures, oil temperature, motor current), 8 digital (valves and switches) |
| Coverage | About 82% of the period; the gaps are mostly while the train is parked or switched off |
| Labels | None. Ground truth comes only from the operator's failure reports |

**Reported failures** (all air leaks, severity "high stress"):

| # | Failure window | Maintenance |
|---|---|---|
| F1 | 2020-04-18 00:00 to 23:59 | not reported |
| F2 | 2020-05-29 23:30 to 2020-05-30 06:00 | printed as 2020-04-30 12:00, which is before the failure; being verified |
| F3 | 2020-06-05 10:00 to 2020-06-07 14:30 | 2020-06-08 16:00 |
| F4 | 2020-07-15 14:30 to 19:00 | 2020-07-16 00:00 |

The source PDF numbers two rows "#1", so we renumber them F1 to F4. The raw CSV (~218 MB) is **not committed** to this repository.

## 6. Approach

We work through Gröger's (2022) four levels of industrial analytics, each building on the one before:

| Level | Question | Work in this project |
|---|---|---|
| Descriptive | What happened? | Signal distributions, operating states (off / idle / load), duty cycles, behaviour around failures |
| Diagnostic | Why did it happen? | Signals that shift before and during leaks; pressure-drop rates, run time, cycle frequency |
| Predictive | What will happen? | Rolling z-score baseline, Isolation Forest, One-Class SVM, autoencoder (as in Davari et al., 2021) |
| Prescriptive | What should we do? | Alert threshold and persistence rule; cost of false alarms vs. missed failures |

**Evaluation rules**

- **No leakage.** Models, scalers and thresholds are fit only on periods assumed to be healthy, such as the ~70 days before F1.
- **Split by time, never at random**, because this is time-series data.
- **Always report the baseline** next to the more complex models.
- **Report per event.** With only four failures, results are reported for each event separately, without over-claiming.
- **Reproducible.** Fixed random seeds, with every experiment's parameters and results logged.

## 7. Success criteria and KPIs

| KPI | Definition | Target / purpose |
|---|---|---|
| **Lead time** (primary) | Failure start minus first sustained alarm | ≥ 2 h, so maintenance can be planned |
| Event recall | Failures detected ÷ known failures | Out of 4. A missed failure is the most expensive outcome |
| False-alarm rate | False alert events per day or per 100 operating hours in healthy periods | Low, so planners trust the alerts |
| Event precision | Valid alerts ÷ all alert events | Avoids unnecessary inspections |
| PR-AUC | Precision–recall area at window level | Supplementary metric for rare events |
| Downtime avoided | Simulated unplanned hours turned into planned work | Links model results to workshop outcomes |
| Maintenance-cost reduction | Simulated reactive cost minus predictive cost | Puts a value on the business case |

## 8. Scope and limitations

**In scope**

- Early detection of air-leak failures on one APU, using sensor data only
- Lead time, false alarms and event recall, measured against the four reported failures
- A simple alert rule a maintenance planner could act on, with its cost trade-offs

**Known limitations**

- Only four failures, all of the same type, so statistical power is low
- One train and one APU, so results may not hold across the fleet
- Failure windows come from coarse report times, and one maintenance date is inconsistent
- About 18% of the period has no data, and this CSV is at 10 s rather than 1 Hz

## 9. Milestones

| Date | Deliverable | Status |
|---|---|---|
| 2026-10-15 | Team and roles, topic approval, glossary, initial business understanding | Done |
| 2026-11-19 | Milestone presentation: domain, scope, objectives, project organisation, business case, dataset, first descriptive analytics | In progress |
| 2027-01-21 | Final presentation: own research questions, lecture concepts applied, solution design and implementation | Planned |

## 10. Repository structure

Current contents:

```
├── README.md
├── EDA.ipynb                                          # exploratory data analysis
├── MetroPT3_DataUnderstanding_Juliana-checkpoint.ipynb # data understanding (Juliana)
└── docs/
    └── index.html                                     # business report (web page)
```

Planned layout as the project grows:

```
data/raw/         # original CSV (not committed)
data/processed/   # parquet and feature tables (not committed)
notebooks/        # numbered notebooks: 01_eda, 02_regimes, ...
src/              # reusable code: loading, features, models, evaluation
config/           # failure windows, thresholds, experiment configs
results/          # metrics, figures, experiment logs
docs/             # glossary, business report, contributions, slides
```

## 11. Getting started

1. Download `MetroPT3(AirCompressor).csv` from the [UCI repository](https://archive.ics.uci.edu/dataset/791/metropt%203%20dataset) and place it in `data/raw/`.
2. Use Python 3.14 with pandas and scikit-learn.
3. Open the notebooks in Jupyter or VS Code.

Load the CSV once and save it as parquet in `data/processed/`. Work from the parquet file after that, since repeated reads of the 218 MB CSV are slow.

## 12. References

- Veloso, B., Ribeiro, R. P., Gama, J., & Pereira, P. M. (2022). The MetroPT dataset for predictive maintenance. *Scientific Data, 9*, 764. https://doi.org/10.1038/s41597-022-01877-3
- Davari, N., Veloso, B., Ribeiro, R. P., Pereira, P. M., & Gama, J. (2021). Predictive maintenance based on anomaly detection using deep learning for air production unit in the railway industry. *IEEE DSAA 2021*. https://doi.org/10.1109/DSAA53316.2021.9564181
- Barros, M., Veloso, B., Pereira, P. M., Ribeiro, R. P., & Gama, J. (2020). Failure detection of an air production unit in operational context. Springer. https://doi.org/10.1007/978-3-030-66770-2_5
- Navinchandran, M., Sharp, M. E., Brundage, M. P., & Sexton, T. B. (2019). Studies to predict maintenance time duration and important factors from maintenance work-order data. *PHM Society Annual Conference 2019*.
- Gröger, C. (2022). Industrial analytics – An overview. *it – Information Technology, 64*(1–2).
- Liu, K. (2022). *Supply Chain Analytics*. Palgrave Macmillan. Code: https://github.com/pysca/SCA
- UCI Machine Learning Repository. MetroPT-3 Dataset. https://archive.ics.uci.edu/dataset/791/metropt%203%20dataset
