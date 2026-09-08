# AI-Powered Anomaly & Fraud Detection for MPLADS — Project Blueprint

## 1. Problem, restated tightly

MPLADS (Members of Parliament Local Area Development Scheme) gives each MP ₹5 crore/year to recommend developmental works in their constituency. District Authorities sanction and execute these through Implementing Agencies. Since April 2023, this runs through the **eSAKSHI portal** (mplads.mospi.gov.in).

Where fraud/inefficiency actually shows up in this scheme:
- **Non-utilization / delayed utilization** of allocated funds
- **Works sanctioned but never completed**, or marked complete without verification
- **Cost inflation** — a work costs far more than similar works elsewhere
- **Contractor/agency concentration** — one Implementing Agency wins a suspicious share of works in a district
- **Duplicate or near-duplicate works** — same work re-sanctioned under different names
- **SC/ST quota violations** — MPs are required to allocate 15% to SC areas, 7.5% to ST areas; many don't
- **Geographic mismatch** — works recommended outside the MP's actual constituency/nodal district (this is a real, currently-reported issue)
- **Sudden spikes** in sanctions right before elections or fiscal year-end (classic "use it or lose it" pattern)

Pick **3–4 of these** as your core detection targets for the prototype. Trying to detect all seven in a hackathon timeframe will dilute your demo — depth on a few beats breadth on all.

## 2. Data sources

| Source | What it gives you |
|---|---|
| **eSAKSHI portal** (mplads.mospi.gov.in) | Official live data since Apr 2023 — works, sanctions, fund flow. Check if they expose an open dataset/API or if you'll need to scrape the public dashboard. |
| **dataful.in MPLADS dataset** | Pre-compiled 2017–2023 Rajya Sabha MPLADS data with clean columns: `state, nodal_district, implementing_district, house_name, member_type, mp_name, sanction_amount, date_of_administrative_approval, work_name, unique_work_number, implementing_agency_name, work_status, date_of_receipt_of_work_proposal_from_mp, unit` |
| **data.gov.in** | Search "MPLADS" — historical state-wise release/expenditure/utilization tables |
| **PIB & Lok Sabha/Rajya Sabha Q&A archives (eparlib.nic.in)** | State-wise pending works, utilization percentages — good for validating your anomaly thresholds against real historical numbers |
| **Synthetic augmentation** | Once you have the real schema, generate synthetic "planted fraud" rows (implausible costs, backdated approvals, one agency winning 90% of a district) so you have labeled cases to validate your model against — since real fraud isn't labeled in the raw data |

**Important for judging:** SIH panels specifically probe "is this real data or did you make it up?" Anchor your demo in the real compiled dataset, and clearly label any synthetic fraud cases as synthetic (used only for validation, not as your primary dataset).

## 3. System architecture

```
┌─────────────────┐     ┌──────────────────┐     ┌───────────────────────┐
│  Data Sources     │ →  │  Ingestion &      │ →  │  Feature Store /       │
│  (eSAKSHI, CSVs)  │     │  Cleaning Pipeline│     │  Structured DB        │
└─────────────────┘     └──────────────────┘     └───────────────────────┘
                                                            │
                              ┌─────────────────────────────┼─────────────────────────┐
                              ▼                              ▼                          ▼
                     ┌────────────────┐          ┌────────────────────┐    ┌────────────────────┐
                     │ Rule-Based      │          │ ML Anomaly Models   │    │ NLP on work         │
                     │ Flag Engine     │          │ (IsolationForest,   │    │ descriptions (LLM   │
                     │                 │          │  LOF, DBSCAN,       │    │ for duplicate/       │
                     │                 │          │  Autoencoder)       │    │ semantic matching)   │
                     └────────────────┘          └────────────────────┘    └────────────────────┘
                              └─────────────────────────────┬─────────────────────────┘
                                                             ▼
                                                 ┌────────────────────────┐
                                                 │  Fraud Risk Scoring &   │
                                                 │  Explainability Layer   │
                                                 └────────────────────────┘
                                                             │
                                                             ▼
                                                 ┌────────────────────────┐
                                                 │  Dashboard (React) +    │
                                                 │  API (FastAPI) +        │
                                                 │  Alerts/Reports         │
                                                 └────────────────────────┘
```

## 4. Detection logic — concrete signals to implement

**Rule-based (build these first — fast, explainable, always demo-safe):**
- `utilization_ratio = amount_spent / amount_sanctioned` — flag if low and deadline is near
- `days_since_sanction - expected_completion_days` — flag overdue works still "in progress"
- `agency_work_share_in_district` — flag agencies above a percentile threshold
- `cost_zscore` within work-category peer group (e.g., all "road construction" works) — flag statistical outliers
- SC/ST allocation percentage vs the 15%/7.5% mandated minimum

**ML layer (add once rules work):**
- **Isolation Forest / LOF** on a feature vector per work: `[sanction_amount, cost_zscore, delay_days, agency_concentration, sc_st_flag]` — unsupervised, no labels needed, standard for fraud detection
- **DBSCAN clustering** on agency-district-MP triples to surface unusual clusters (e.g., one agency working across an implausible number of unrelated districts)
- **Autoencoder** (optional, stronger for the "AI-powered" narrative) — train on presumed-normal works, flag high reconstruction error as anomalous

**NLP layer (optional but a strong differentiator):**
- Embed `work_name`/description text (sentence-transformers) and flag near-duplicate works recommended multiple times — catches "same work, reworded, sanctioned twice"

## 5. Tech stack recommendation

- **Backend/API:** Python + FastAPI
- **ML:** scikit-learn (IsolationForest, LOF, DBSCAN), PyOD if you want more anomaly algorithms out of the box, PyTorch/Keras only if you commit to the autoencoder
- **NLP:** sentence-transformers (all-MiniLM) for duplicate/semantic work matching
- **DB:** PostgreSQL (or SQLite for the prototype if time-constrained)
- **Frontend:** React + Tailwind, charts via Recharts, maps via Leaflet/Mapbox for district-level visualization
- **Deployment:** Render/Railway (backend), Vercel (frontend) — pick whatever gives you a live URL fastest

## 6. Development phases — detailed, with "vibe coding" workflow

You're right that this is the way to build fast now. The key discipline with AI-assisted ("vibe") coding in a hackathon is: **one phase = one clear scope = one set of prompts**, and you review/run the output before moving to the next phase. Trying to prompt the whole system in one shot is where vibe-coded projects fall apart under judge questioning — you want to actually understand every layer well enough to defend it live.

**Phase 1 — Problem & Data Recon** *(no code yet)*
Manually explore the datasets, write down your exact schema and your exact anomaly definitions on paper/doc first. Skipping this and jumping straight to prompting an AI coding tool is the #1 reason hackathon teams end up with a system that doesn't map to the real problem statement.

**Phase 2 — Data Pipeline**
Prompt scope: "Write a Python ingestion script that loads [dataset], cleans column X/Y/Z, handles missing dates, and outputs a normalized DataFrame matching this schema: [paste your schema]." Run it against a data sample, fix by hand where the AI gets column-specific quirks wrong (it will).

**Phase 3 — Rule Engine**
Prompt scope per rule, not all at once: "Given this DataFrame schema, write a function that flags works where utilization_ratio < 0.5 and days_to_deadline < 30." Test each rule against known real cases (e.g., states with documented low utilization) to sanity-check thresholds.

**Phase 4 — ML Layer**
Prompt scope: "Fit an IsolationForest on these features [...], return anomaly scores 0–1." Validate against your synthetic planted-fraud rows — this is your only real accuracy check, so build those test cases before this phase.

**Phase 5 — Scoring & Explainability**
Prompt scope: "Combine these rule flags and this anomaly score into a single risk score, and generate a one-line natural-language reason for each flag." This is a good spot to use an LLM call itself (small, templated) rather than hand-written string logic — it scales better across different flag combinations.

**Phase 6 — Dashboard**
Prompt scope: build screen by screen — overview/summary view, then work-list with filters, then a single work's drill-down/explanation view. Don't prompt "build the whole dashboard" — you'll get something generic and hard to steer.

**Phase 7 — Alerts/Reports**
Prompt scope: PDF/CSV export of top-N flagged works; optional email trigger. This is a "nice to have" — build it last and cut it first if time runs short.

**Phase 8 — Test, Deploy, Pitch**
Deploy early (even Phase 3's rule engine deserves a live demo) so you're never staring at a deployment failure the night before. Prepare a backup screen-recorded demo. Structure the pitch as: real problem → real data → what you detect and why it matters → live dashboard → quantified impact (₹ value of flagged works, % of dataset flagged, etc.) → what you'd add with more time (production API integration with eSAKSHI, human-in-the-loop verification workflow, state-wise rollout).

## 7. What impresses SIH judges specifically

- **Explainability over black-box scores** — always show *why* something was flagged
- **Real data, clearly sourced** — cite eSAKSHI/dataful.in, don't fabricate
- **A human-in-the-loop framing** — position this as a decision-support tool for auditors/MoSPI, not an auto-accusation engine (fraud accusations need human review; this positioning also avoids the ethical trap of an "AI declares fraud" system)
- **Quantified impact** — "flagged ₹X crore worth of works as high-risk across Y% of records" is a far stronger closing line than describing the tech stack
- **Scalability story** — mention how this could plug into other similar schemes (state MLA funds, other central sector schemes) since judges often ask about generalizability
