# Lab 2b – High-Load Cybersecurity with Morpheus Lite

**Students:** Tamara Karanovich & Adi Tavakoli  
**Date:** 22 November 2026  
**Platform:** Morpheus Lite v0.3.4 (`lab2b-morpheus-lite-high-load`), Windows + Docker Desktop, Python 3.12, deterministic XAI  
**Tests:** `python -m pytest -q` → **14 passed**

---

## 1. Setup

Components were started in the required order, each in its own terminal: Redpanda + Console (Docker) → detector → orchestrator (started telemetry only after the `ready … partitions=morpheus.alerts:0` message) → Streamlit dashboard → telemetry generator (one profile per run).

| Component | Role | Address |
|---|---|---|
| `telemetry_generator.py` | Producer (load profiles) | – |
| Redpanda (Kafka) | Event bus / buffer | `localhost:9092` |
| `morpheus_lite_detector.py` | Detection stage | – |
| `agent_orchestrator.py` | Investigation, RAI, XAI, Meta-AI stage | – |
| Streamlit dashboard | High-load observability | http://localhost:8501 |
| Redpanda Console | Topic inspection | http://localhost:8080 |

**Setup notes:** the virtual environment was created with Python 3.12, and `kafka-python<3` was installed after `requirements.txt` (kafka-python 3.x could not connect to the broker in Lab 2a).

For each profile we waited until the generator finished and the pipeline drained (Generated = Detected = Orchestrated), refreshed the metrics, and downloaded the CSV. All CSV files are in the `csv/` folder.

---

## 2. Measurement table

Values are the dashboard values at the end of each run (rates and latencies are recent values over the last part of the run). Peak backlogs and peak rates come from the downloaded CSVs.

| Measure | A – Baseline | B – Normal | C – Burst | D – Sustained | E – Attack surge |
|---|---|---|---|---|---|
| Run ID | `567b9d32` | `a68877f8` | `8f70298e` | `52b2a758` | `8fd1bba0` |
| Generated events/s | 1.0 | 25.0 | 87.4 (peak 174) | 95.5 (peak 184) | 77.8 (peak 235) |
| Detector events/s | 1.0 | 25.0 | 17.0 (peak 48) | 14.0 (peak 44) | 15.4 (peak 40) |
| Orchestrator alerts/s | 1.0 | 26.3 | 17.6 (peak 30) | 14.0 (peak 28) | 15.4 (peak 27) |
| Detector utilization | 3.1% | 47.8% | 72.9% (peak ~100%) | 82.9% (peak ~100%) | 71.2% (peak ~100%) |
| Orchestrator utilization | 3.2% | 77.1% | 63.8% (peak ~100%) | 54.1% | 55.5% |
| Peak ingress backlog / capacity | 3 / 500 | 4 / 500 | **1020 / 500** | **3936 / 500** | **4106 / 500** |
| Peak alert backlog / capacity | 1 / 500 | 17 / 500 | 347 / 500 | 97 / 500 | 39 / 500 |
| Detector queue wait | 7.2 ms | 13.1 ms | 39,972 ms | 308,802 ms | 253,999 ms |
| Detector processing latency | 30.8 ms | 19.1 ms | 42.8 ms | 59.4 ms | 46.3 ms |
| Orchestrator queue wait | 5.3 ms | 404.8 ms | 14,642 ms | 8.6 ms | 18.6 ms |
| Orchestrator processing latency | 32.1 ms | 29.4 ms | 36.4 ms | 38.7 ms | 36.0 ms |
| Full-pipeline E2E latency | 70.3 ms | 463.6 ms | 55,182 ms | 308,876 ms | 254,039 ms |
| Final system status | NORMAL | HIGH LOAD | HIGH LOAD (OVERLOADED during run) | HIGH LOAD (OVERLOADED during run) | HIGH LOAD (OVERLOADED during run) |
| Generated / Detected / Orchestrated | 97 / 97 / 97 * | 500 / 500 / 500 | 2000 / 2000 / 2000 | 5000 / 5000 / 5000 | 5000 / 5000 / 5000 |
| Total run time (sending → drained) | 96 s * | 20 s → 20 s | 33 s → 93 s | 48 s → ≈ 6 min | 76 s → 336 s |

\* The baseline CSV was downloaded 96 s into the 120-event run, before the last events were sent. The run was stable (no backlog), so the values are representative.

**Note on final status:** the dashboard status is computed from the last seconds of the run. For C, D and E the status was **OVERLOADED** while the backlog was growing (see the overload screenshots) and changed to HIGH LOAD only after the queues had drained.

---

## 3. Screenshots

**Throughput (E – attack surge).** The generator (light blue) sends 50–170 events/s for about 76 s, while the detector and orchestrator (dark blue and red, near the bottom) stay at about 15–25 events/s and continue working until about 336 s.  
![Throughput](throughput.png)

**Backlog (E – attack surge).** The ingress backlog (light blue) rises to about 4,100 events while the generator is sending, then falls slowly until about 336 s as the detector works through it. The alert backlog (dark blue) stays near 0, showing that the detector, not the orchestrator, is the bottleneck.  
![Backlog](backlog.png)

**Latency decomposition (D – sustained).** The detector queue wait climbs in a straight line, because each new event waits behind a longer queue. Processing times stay near the bottom of the chart.  
![Latency decomposition](latency.png)

**During overload (attack surge, 1999 of 5000 detected, ingress backlog 3001/500)**  
![Overload](attack_surge_overload.png)

---

## 4. Bottleneck per profile

| Profile | Bottleneck | Evidence |
|---|---|---|
| **A – Baseline** | None | Both stages ≈3% utilization, backlogs ≤ 3, latency 70 ms is almost all processing. |
| **B – Normal** | **Orchestrator** | Highest utilization (77% vs 48%), alert backlog peaked at 17 while ingress stayed at 4, orchestrator queue wait 405 ms vs detector 13 ms. Everything was still processed (500/500/500). |
| **C – Burst** | **Detector** (first), orchestrator close behind | Ingress backlog exceeded capacity (1020/500 at 33 s) before the alert backlog peaked (347/500 at 41 s). Detector peak utilization ~100%. |
| **D – Sustained** | **Detector** | Ingress backlog 3936/500 while the alert backlog never exceeded 97. Orchestrator queue wait only 8.6 ms: it was waiting for the detector. |
| **E – Attack surge** | **Detector** | Ingress backlog 4106/500, alert backlog only 39. Orchestrator processed every alert it received immediately (queue wait 18.6 ms). |

**Why the bottleneck moved from the orchestrator (B) to the detector (C–E):** at light load the detector needed ≈19 ms per event and the orchestrator ≈29 ms, so the orchestrator was the slower stage. Under heavy load the detector slowed to 43–59 ms per event, while the orchestrator stayed at 36–39 ms. All components share one CPU, and the detector, which receives every event first, was hit hardest by the contention.

**Observation: the detector forwarded every event as an alert.** In every run Alerts = Detected (100%), even though only 12–65% of the events were labelled suspicious. A likely cause is the shipped `data/fingerprint_profiles.json`: every user's `login_hour` history is ≈6:00 UTC with a standard deviation of ≈0.07, so events generated at a different hour get a very large fingerprint z-score (≥ 3), which raises the risk score to ≥ 80. Because alerted events are not treated as "trusted normal", the profiles are never updated, so the effect persists. As a result there was no early filtering, and the orchestrator received the full event stream in every profile.

---

## 5. Reflection (≈480 words)

**1. Which stage saturated first?** Under normal load the orchestrator was closest to its limit (77% utilization, 405 ms queue wait, alert backlog 17). Under burst, sustained and attack-surge load the detector saturated first: its ingress backlog exceeded the 500-event capacity (1020, 3936 and 4106), while the alert backlog stayed below capacity and the orchestrator's queue wait in D and E was under 20 ms. A stage after the bottleneck cannot be overloaded, because the bottleneck limits the flow it receives.

**2. Queue wait or processing time?** Queue wait dominated in every loaded run. Processing time stayed between 19 and 60 ms per stage, but queue wait grew from about 400 ms (B) to 40 s (C) and 309 s (D). In D, 99.98% of the 308,876 ms end-to-end latency was the detector queue wait. The latency chart shows this as a straight rising line: each new event waited longer than the one before.

**3. Why can latency rise before 100% CPU?** Arrivals are uneven. Even at 77% average utilization, short spikes (the generator peaked at 73 events/s in B) create a queue, and every event behind it waits. The closer a stage runs to its capacity, the longer each spike takes to clear. In addition, all processes share one CPU, so contention made the detector slower exactly when load was highest (19 → 59 ms per event).

**4. Why should attack surge stress the orchestrator more?** With 65% suspicious events, many more events become alerts and need the expensive investigation, RAI, XAI and Meta-AI analysis. In our runs this effect did not appear: the detector already forwarded 100% of events in every profile, and it was the slower stage, so the orchestrator's load was limited by the detector rather than by the attack ratio.

**5. Early filtering.** Filtering out clearly benign events early protects the expensive analysis stage and reduces latency for real threats. Our runs show the cost of no filtering: every event reached the orchestrator. However, overly aggressive filtering is dangerous: an attacker who makes malicious activity look normal, or hides it inside a flood of benign events, could be dropped before analysis, creating false negatives that are never investigated.

**6. Graceful degradation.** Yes. Kafka buffering meant no event was lost, but the last events in D were analysed about 5 minutes late, which gives an attacker time to act. A better design degrades gracefully: under overload it performs quick triage on everything, prioritises high-risk alerts for full analysis, and defers or samples low-risk events, as long as every degraded decision is logged so it can be reviewed later.

**7. Which metric should trigger backpressure?** The ingress backlog ratio (backlog / capacity) combined with queue wait. In our runs the backlog crossed 50% of capacity within seconds of overload, before utilization numbers clearly showed a problem, and queue wait directly measures the delay that matters for security response. Utilization alone is a lagging and averaged signal.
