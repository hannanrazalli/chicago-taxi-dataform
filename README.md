# Chicago Taxi Trips — Data Analytics Engineering Assessment

## 1. Initial Assessment & Engineering Discovery
Upon receiving this assessment and profiling the bigquery-public-data.chicago_taxi_trips.taxi_trips dataset, I established the following engineering constraints and data realities before writing any code:

*   **Data Nature (Static vs. Live):** The public dataset is a static snapshot ending on December 31, 2023. Therefore, the "last 3 months" requirement cannot be driven by dynamic runtime dates like CURRENT_DATE(). It must be anchored exactly to the dataset's maximum boundary (2023-09-30 to 2023-12-31).
*   **Cost & Compute Constraints:** The raw source table is multi gigabytes. I built a centralized Staging layer to handle data quality filters, time-window bounding, and de-duplication exactly once. All downstream analytical models (Q1, Q2, Q4) query this clean staging table, eliminating redundant table scans and lowering BigQuery compute costs.
*   **The Q3 Architectural Trade-Off:** To analyze US Public Holiday impacts (Q3), a tight 3-month window is too small a sample. I made a deliberate design decision to let Q3 bypass the staging table and read from the raw source directly. To keep this operation cheap, I applied a strict 2-year partition filter (2022-01-01 to 2024-01-01), avoiding a scan of the entire table history back to 2013.

---

## 2. Pipeline Architecture, Methodology, Assumptions & Definitions

taxi_trips (Raw Public Source)
│
├──► staging/stg_taxi_trips (Cleaned, deduped, fixed 3-month window)
│     │
│     ├──► marts/q1_top_earners.sqlx
│     ├──► marts/q2_overworkers.sqlx
│     ├──► marts/q4_insight1_demand.sqlx
│     └──► marts/q4_insight2_peak_hour.sqlx
│
└──► marts/q3_holiday_impact.sqlx (2-year window)

### Staging Layer (Raw Data Refinement)
*   **Data Integrity Rules:** Rows containing impossible physical logic such as zero or negative trip durations (trip_seconds <= 0), negative financial fields (fare < 0), or end timestamps that occur before start timestamps are treated as corrupt data and dropped.
*   **Deduplication Strategy:** A single trip must map to a unique transactional record. Duplicates on unique_key are resolved using QUALIFY ROW_NUMBER() OVER (PARTITION BY unique_key ORDER BY trip_start_timestamp DESC) = 1 before data enters the marts.

### Q1: Top 100 Tip Earners
*   **Definition:** Drivers are ranked by raw total recorded tips within the designated 3-month analysis window, not tips as a percentage of fare, and not tip per trip. This was a direct read of the question's wording, so no further interpretation was layered on top.
*   **Assumption:** "Last 3 months" is taken strictly as the 3 calendar months ending at the dataset's actual max timestamp (2023-12-31), not 90 days or any other approximation.
*   **Reporting Bias Disclaimer:** The dataset documentation states "the tip for the trip. Cash tips generally will not be recorded." An exploratory check on this window confirmed cash tips are recorded far less consistently than card tips [INSERT VERIFIED % HERE — see note below]. This introduces a known reporting bias: drivers operating heavily in cash-dominant zones will rank lower in this model than card-dominant drivers, despite potentially similar or higher actual earnings.

### Q2: Top 100 Overworkers
*   **Definition of a "Shift":** A continuous sequence of trips bound by a recovery gap. Any rest period equal to or exceeding 8 hours terminates the current shift and initializes a new shift.
*   **Shift Duration Logic:** The difference between the final trip's end and the first trip's start within that shift. This acts as a proxy for operational fatigue exposure, not pure driving time.
*   **Safety Threshold:** Benchmarked against commercial transport safety limits (e.g. NYC TLC's 12-hour driving cap), a "long shift" is defined as any shift exceeding 12 continuous hours.
*   **Assumption:** "Last 3 months" is taken strictly as the 3 calendar months ending at the dataset's actual max timestamp (2023-12-31), not 90 days or any other approximation.
*   **Reporting Bias Disclaimer:** There's no fixed industry rule for what percentage of long shifts counts as "regular," so this was treated as a judgment call rather than a hard cutoff baked into the query. The dashboard surfaces pct_long_shifts (long shifts ÷ total shifts) per driver and highlights anything ≥80% as a visual flag. Looking at the actual results, every driver in the top 100 already sits at ≥50% long shifts, meaning for every 10 shifts logged, at least 5 were over 12 hours. That's a strong enough pattern on its own that a stricter cutoff wasn't necessary to make the case.
*   **Fatigue Metric:** The data highlights extreme cases in the fleet, with some drivers logging maximum shift spikes between 40 and 61 hours without a valid 8-hour break.

### Q3: US Public Holiday Impact
*   **Methodology:**  To prevent an "apples-to-oranges" comparison, data is segmented into three distinct buckets: Named Holidays, Weekends, and Regular Weekdays. This isolates the weekend effect so a holiday falling on a weekend isn't misread as a holiday-induced volume drop.
*   **Assumption** A 2-year window (2022–2023) was assumed sufficient to observe a holiday effect, rather than pulling the full 2013–2023 history. Within that window, only the 4 major US holidays (New Year, Independence Day, Thanksgiving, Christmas) were used rather than the full federal holiday calendar chosen as the ones most likely to show a clear demand shift, given the limited time available to validate a longer list. Holiday dates were hardcoded directly in the query rather than sourced from a calendar table or API, which would be the production-grade approach but was out of scope here.
*   **Analytical Finding:** The metrics confirm a clear drop in demand during major holidays. Average weekday volume sits at ~18.6k trips/day, while Christmas Day marks the lowest baseline at ~5.3k trips/day.

### Q4: Bonus Actionable Insights
*   **Insight 1 (Spatial Demand-Supply Gap):** By calculating the ratio of trips per active driver across official pickup community area codes alongside total revenue, the pipeline flags allocation imbalances. Area 76 (O'Hare Airport zone) dominates both trip volume and revenue by a wide margin compared to every other area.
*     Recommendation:  Increase the number of drivers on standby at O'Hare to reduce passenger wait times but avoid oversupplying the zone, since too many idle drivers there would just shift the inefficiency from the passenger side to the driver side.
*   **Insight 2 (Temporal Peak Demand Patterns):** Aggregating trip counts by hour of day (0–23) shows trip volume surging from around midday and peaking sharply around 5 PM.
*     Recommendation: Increase driver availability and incentives during this midday-to-5PM window, and scale incentives back during low-demand overnight hours, to match driver supply to actual rider demand rather than spreading it evenly across the day.

---

## 3. Repository & Visualisation Links

*   **GCP Dataform Codebase:** Organized inside `definitions/` into explicit `staging/` and `marts/` sub-directories to maintain data lineage boundaries.
*   **Interactive Looker Studio Dashboard:** **https://datastudio.google.com/s/v40HKwWK3BE**