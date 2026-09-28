# Smart Factory Power Anomaly Detection and Monitoring

**5th BDAI Data Analytics Competition · Team green그린그림**  

**2nd place out of 10+ finalists**

This project uses RTU power data from 13 equipment types to identify **which equipment and time periods deserve inspection first**. An unsupervised model screens all equipment; IQR, EWMA, and Isolation Forest then examine the selected equipment at five-second resolution. A monitoring dashboard presents the results. An *anomaly* means a departure from the observed electrical pattern, not a confirmed failure or measured energy loss.

![Smart factory power anomaly monitoring dashboard](https://appapppy-smartfactorydashboard.streamlit.app/)

![Dashboard Github](https://github.com/smichelle0804/smart-factory-dashboard-github)

## At a glance

| Item | Description |
  | --- | --- |
  | Data | 13 equipment types; 33,696,013 records; 19 variables |
  | Period | December 1, 2024 – April 30, 2025; about 150 days at five-second intervals |
  | Main signals | Three-phase voltage, current and power factor; active and reactive power; operation status; cumulative energy |
  | Goal | Prioritize equipment and inspection times without ground-truth anomaly labels |
  | Deliverable | Two-stage detection workflow and a Streamlit monitoring prototype |
  
  ## Why detection instead of forecasting?
  
  We first hypothesized that three-phase voltage and current imbalance could precede a future power-factor drop. Logistic Regression, XGBoost, and LightGBM reached **0.63–0.66 ROC-AUC** with a lagged power-factor feature on time-ordered validation data. Removing that feature reduced ROC-AUC to **0.49–0.51**, indicating that the apparent predictive signal largely came from short-term persistence in power factor. **99.24%** of the analyzed anomaly episodes lasted only one five-second observation.

We therefore found **no useful advance signal within the available variables and observation period** and shifted to detecting departures in the current electrical state.

## Workflow

1. **Validate and prepare data.** The report found zero collection gaps, missing values, cumulative-meter rollovers, or voltage-range violations. We used `localtime` for analysis and derived mean power factor, three-phase voltage/current imbalance, and the reactive-to-active-power ratio.
2. **Stage 1 — screen all equipment.** Isolation Forest used **six signals**: active power, reactive power, mean power factor, voltage imbalance, current imbalance, and reactive-to-active-power ratio. Screening **561,613 five-minute aggregate rows** selected the preliminary dryer (equipment 15) for closer inspection.
3. **Stage 2 — examine five-second observations.** For the selected equipment, we compared a power-factor IQR rule (`3 × IQR`), a power-factor EWMA control chart (30-minute smoothing, `3σ`), and Isolation Forest with **four signals**. EWMA ran on the five-second source observations to retain short spikes that a five-minute average could hide.
4. **Review in a dashboard.** The prototype shows equipment rankings, method-level decisions, power-factor trends, hourly event counts, and recent events. It has not been connected to a live RTU stream as a production system.

## Results

| Analysis | Result | Meaning |
  | --- | --- | --- |
  | All-equipment screening | Preliminary dryer (15) ranked first of 13 at **1.62%** | Candidate for prioritized inspection |
  | IQR | **20,317 flagged observations** (0.78%) | Univariate baseline for extreme power-factor values |
  | EWMA | **26,244 flagged observations** (1.01%); **25,991 grouped events** | Time-aware power-factor monitoring |
  | Stage 2 Isolation Forest | **20,736 flagged observations** (0.80%) | Multivariate candidate anomalies |
  | EWMA vs. IQR | Included every IQR-flagged observation plus **5,927** others | Different detection coverage |
  | EWMA vs. Isolation Forest | **16,179** shared; **4,557** IF-only; **10,065** EWMA-only | Complementary candidate periods |
  | High-frequency periods | **369 periods** above **10.93 events/hour** | An initial inspection priority rule |
  
The Stage 1 rate uses **five-minute aggregates across equipment**; Stage 2 counts and rates use **five-second observations of the preliminary dryer**. The **25,991 EWMA events** group consecutive flagged observations, whereas **26,244** counts individual flagged observations. Stage 2 Isolation Forest used `contamination=0.008` to match IQR's approximate detection volume, so its **0.80% rate is not independent evidence of accuracy**. Overlap between methods is not a ground-truth performance measure either.

## Operational use

The proposed flow is **rank equipment → compare method decisions at a timestamp → prioritize repeated or high-frequency periods → inspect on site → compare before and after action**. Hourly aggregation and agreement across methods can help prioritize review instead of sending an alert for every five-second spike. The **10.93 events/hour** threshold is an initial operational choice, set at 1.5 times the observed hourly mean of 7.29; it requires calibration in the field.

## Limitations and next steps

- No fault or maintenance labels were available, so the physical cause and false-positive rate have not been verified. Field inspection and sensor calibration records are needed.
- Mean active power was unusually similar across functionally different equipment, at roughly 3,009 W. Metering and equipment labels need checking. A recorded `operation=1` throughout the dataset does not by itself prove that every machine physically ran around the clock.
- **Energy savings, bill savings, and avoided emissions were not measured.** They require actual before-and-after consumption and billing data.
- Applying the workflow elsewhere requires recalibrating equipment baselines, EWMA settings, Isolation Forest thresholds, and alert rules. Live data ingestion and model update criteria are future work.

*Prepared from the competition report, presentation, and monitoring prototype screenshot.*
