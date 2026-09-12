# vacuum_ad

**Detecting anomalies in accelerator vacuum pressure through physically-informed time-series modeling.**

> **Project Status:** Active Research & Development 🚧
> This repository is currently in the exploratory phase. We are investigating different paradigms for anomaly detection, comparing rigorous Bayesian inference algorithms (e.g., Particle Filters, Jump-Diffusion models) against robust, hand-engineered heuristic baselines. 

## Overview

This project aims to monitor the vacuum pressure inside an accelerator waveguide and automatically flag anomalies, such as sudden gas releases or RF discharges. 

The sensor reports one pressure reading every six hours. Buried in these readings are a handful of real events we want to catch, while the "normal" behavior itself slowly wanders, breathes with a daily thermal cycle, and is heavily rounded off by the analog-to-digital converter (ADC). A fixed threshold on the raw pressure would either miss the small events or drown the system in false alarms. 

## The Core Challenges

Anomaly detection in this environment is not a standard outlier-detection problem. Instead of treating anomalies as corrupted data points (observation outliers), we model them as physical impulses to the system's state (innovation outliers). Several problems arise that need to be tackled:

*   **The Chicken-and-Egg Problem:** To decide whether a reading is anomalous, we need to know the "normal" baseline dynamics. But to estimate those dynamics cleanly, we need to remove the anomalies first.
*   **Transient Identifiability:** The physical relaxation coefficient of the vacuum pump (the rate at which the air is pumped out of the system) is almost invisible when the system is at equilibrium. It only becomes mathematically identifiable during the transient recoveries *after* an anomaly.
*   **Quantized Measurements:** The ADC reports readings with limited significant digits. Pretending these readings are exact might push the model to overestimate noise and discard useful long-time information; the likelihood must acknowledge the bin width of the sensor.
*   **Drift, Jumps and Oscillations:** The physical parameters are subject to both fast and slow oscillations, dependent on temperature (which we assume to be unknown), sensor and device degradation, as well as sudden jumps, such as for the manual change of the number of active pumps.


## Methodology & Roadmap

The project is currently advancing along two parallel tracks. We will evaluate their trade-offs in terms of precision, recall, computational cost, and interpretability. The ultimate goal is to find the most robust and accurate tool to address the task at hand, possibly combining the following approaches.

### Track A: Bayesian Filtering (Switching State-Space Models)

This track is a pure (empirical) Bayesian approach to the problem. We define a generative model for the dynamics with hidden hyperparameters and we infer the information on the anomalous impulses from the data and this prior model.

*   **The Model:** A discretized mass-balance conservation law where the relaxation coefficient $\alpha_t$ and asymptotic pressure $\pi_t$ are allowed to drift:
$$
\begin{aligned}
    P_t &= \alpha_t\,P_{t-1} + (1-\alpha_t)\,\pi_t + w_t + J_t\cdot\mathbf{1}\{s_t=\text{anomaly}\},
    & w_t \sim \mathcal N(0,\sigma_w^2)& \\[2pt]
    J_t &\sim \text{LogNormal}, & & \\[2pt]
    &\dots\;\;\dots\;\;\dots && \\[2pt]
    Y_t &= \text{round}(P_t) & \text{rounding to the first $3$ significant digits}&
\end{aligned}
$$
*   **The Inference:** We are exploring **Rao-Blackwellized Particle Filters (RBPF)**. The particle filter handles the non-linear, non-Gaussian components (discrete anomaly regimes, calendar-based hazard priors, logit-scale coefficient jumps), while an exact 2D Kalman filter analytically tracks the linear-Gaussian baseline states, drastically reducing the variance of the estimates and the number of particles needed.
*   **Why it matters:** This approach solves two major problems in real-world anomaly detection:
    - Robustness to slow degradation: Instead of using fixed thresholds, the model dynamically adapts to the slow, natural wear-and-tear of the vacuum system over months. It learns the "new normal", isolating true sudden anomalies without triggering false alarms.
    - Calibrated confidence on imperfect sensors: It doesn't just flag anomalies; it outputs a precise probability. Using a statistical technique called RPIT, we can mathematically certify this confidence even when dealing with low-resolution, heavily rounded sensor data.

### Track B: Engineered Heuristic Baselines
Advanced statistical models can sometimes be sensitive to mis-specification and unmodeled dynamics in real-world industrial settings. To guarantee reliability, we are simultaneously building rule-based, domain-specific algorithms.

*   **The Approach:** A *divide et impera* split to inductively identify consecutive time intervals with clean exponential decay (no anomalies), finding the anomalies in different stages and looking for regime outliers and jumps. This approach needs hyperparameter tuning, which could be done manually or using Track A.
*   **Why it matters:** It establishes a robust, high-recall baseline that is computationally cheap, inherently stable, easy to debug in a production environment, and highly interpretable.

## Future API Structure

As the research phase stabilizes, the codebase will be unified under a simple API that allows testing both tracks interchangeably. The intended usage pattern will look similar to this:

```python
from vacuum_ad import TimeSeriesData
from vacuum_ad.models import ParticleFilterDetector, HeuristicBaselineDetector

data = TimeSeriesData.from_csv(
    "pressure.csv", time_col="timestamp", pressure_col="pressure"
)

# Track A: Rigorous probabilistic approach
pf_detector = ParticleFilterDetector(n_particles=500, threshold=0.5)
pf_report = pf_detector.fit_predict(data)

# Track B: Robust engineering approach
heuristic_detector = HeuristicBaselineDetector(window_size=28)
heuristic_report = heuristic_detector.fit_predict(data)

# Evaluation, comparison, adaptation to production environment
```

## Contributing / Contact

This methodology is currently being validated on historical logs and simulated data. If you are interested in time-series anomaly detection, applied particle filtering, or accelerator physics, feel free to open an issue or reach out.