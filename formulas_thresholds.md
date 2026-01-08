Below is a **scoring specification** you can implement directly. It is designed for your Airtable structure and assumes you can derive weekly time series from **Weekly forms + Feedback Forms**, and qualitative signals from **Notes + free-text fields** via LLM extraction.

The spec has three layers:

1. **Feature extraction** (raw → normalized)
2. **Sub-scores** (0–100) with **formulas**
3. **Overall score + thresholds** that drive actions

---

# 0) Entities and time window

**Unit of scoring:** `FounderWeek` (one founder, one week)

**Join key:** `Main SU 2025` row (cohort enrollment)
Links to: Founder (Profile Book), Startup (Startups), Weekly form(s), Feedback form(s), Notes, Evaluation.

**Week definition:**

* Use ISO week (Mon–Sun) by submission date (prefer explicit “week” field if present).
* If multiple submissions in a week: use the latest; keep the others for audit.

**Missing week:** If no weekly submission and no feedback in the week → mark `missing_update = 1`.

---

# 1) Stage buckets and normalization

You must stage-normalize traction-related metrics.

Define `stage ∈ {IDEA, MVP, TRACTION, REVENUE}`.

If you have a stage field in Startups, use it; else infer:

* `REVENUE` if revenue > 0 for 2+ consecutive weeks
* `TRACTION` if active_users > 20 for 2+ weeks or pilot/LOI flagged
* `MVP` if product built flag / demo link present or active_users > 0
* else `IDEA`

For stage normalization, you’ll use **piecewise targets** per week:

| Metric (per week)          | IDEA | MVP | TRACTION | REVENUE |
| -------------------------- | ---: | --: | -------: | ------: |
| Customer conversations `C` |    5 |   8 |       10 |       6 |
| Active users `U`           |    0 |   5 |       25 |      50 |
| Revenue `R` (€/$)          |    0 |   0 |       50 |     250 |

These are “target” values, not caps.

---

# 2) Raw inputs (per FounderWeek)

From Weekly/Feedback (use 0 if blank unless noted):

* `C` = number of user/customer conversations (weekly)
* `U` = active users (weekly)
* `R` = revenue (weekly)
* `goals_text` = weekly goals / next steps
* `progress_text` = what happened / what shipped
* `blockers_text` = blockers
* `morale` = if numeric field exists (else null)
* `submission_ts` = timestamp

From Notes (staff qualitative):

* `notes_text` = concatenated notes in the week (or last 14 days if sparse)

From Evaluation (optional weekly or periodic):

* `eval_score` (if exists) mapped to 0–100; else null

---

# 3) LLM-extracted features (strict JSON)

Run LLM on `{goals_text, progress_text, blockers_text, notes_text}` and require:

* `goal_specificity_1_5` (1 vague → 5 measurable)
* `evidence_quality_1_5` (1 claims → 5 numbers/artifacts)
* `experiment_rigor_1_5` (hypotheses/tests)
* `coachability_1_5` (responds to feedback, adjusts)
* `focus_1_5` (1 scattered → 5 tight)
* `blocker_severity_1_5` (1 minor → 5 critical)
* `sentiment_1_5` (1 very low → 5 high)
* `pivot_flag` (bool)
* `scope_creep_flag` (bool)

If LLM fails or texts missing:

* Set these to `null` and handle defaults as described below.

---

# 4) Normalization helpers

### 4.1 Clamp

`clamp(x, a, b) = max(a, min(b, x))`

### 4.2 Saturating ratio → 0..100

For positive metrics vs stage target `T`:
`score_ratio(x, T) = 100 * clamp(x / T, 0, 1.25) / 1.25`

This gives diminishing returns above target.

### 4.3 Likert 1..5 → 0..100

`likert(v) = (v - 1) / 4 * 100`

If `v` is null, use neutral default `50`.

---

# 5) Sub-scores (0–100) with precise formulas

## 5.1 Reliability & Cadence (RC)

Purpose: penalize missing updates and chronic lateness.

Inputs:

* `missing_update` ∈ {0,1}
* `days_late` (if you have due date; else set 0)

Formula:

* `missing_penalty = 35 if missing_update=1 else 0`
* `late_penalty = clamp(days_late * 5, 0, 20)`
* `RC = clamp(100 - missing_penalty - late_penalty, 0, 100)`

If you do not track due dates, RC reduces to missing penalty only.

---

## 5.2 Execution Velocity (EV)

Purpose: how much real work is being done and communicated.

Inputs:

* `goal_specificity_1_5` (LLM)
* `evidence_quality_1_5` (LLM)
* Optional: `shipping_flag` (LLM-inferred from “shipped/released/demo” keywords) → if unavailable, ignore.

Formula:

* `GS = likert(goal_specificity)`
* `EQ = likert(evidence_quality)`
* `EV = 0.55*GS + 0.45*EQ`

Then apply cadence gate:

* `EV = EV * (RC/100)`

This prevents founders with great writing but no submissions from scoring high.

---

## 5.3 Traction Signal (TS) — stage-adjusted

Purpose: reward meaningful market signal without punishing early stage.

Inputs:

* Stage targets: `TC, TU, TR` from table above.
* `C,U,R`

Compute:

* `SC = score_ratio(C, TC)`
* `SU = score_ratio(U, TU)` (if TU=0, treat SU=50 when U=0, SU=100 when U>0)
* `SR = score_ratio(R, TR)` (if TR=0, treat SR=50 when R=0, SR=100 when R>0)

Define special cases:

* If `TU=0`: `SU = 50 if U=0 else 100`
* If `TR=0`: `SR = 50 if R=0 else 100`

Traction score:

* `TS = 0.45*SC + 0.35*SU + 0.20*SR`

Cadence gate (weaker than EV):

* `TS = TS * (0.7 + 0.3*RC/100)`

---

## 5.4 Learning & Iteration (LI)

Purpose: are they running experiments, learning, and adapting?

Inputs:

* `experiment_rigor_1_5`
* `pivot_flag`

Formula:

* `ER = likert(experiment_rigor)`
* `pivot_bonus = 8 if pivot_flag=true else 0` (small positive; pivoting is not always bad)
* `LI = clamp(ER + pivot_bonus, 0, 100)`

Cadence gate:

* `LI = LI * (RC/100)`

---

## 5.5 Focus & Clarity (FC)

Purpose: reduce wasted cycles from scattered priorities.

Inputs:

* `focus_1_5`
* `scope_creep_flag`

Formula:

* `F = likert(focus)`
* `scope_penalty = 15 if scope_creep_flag=true else 0`
* `FC = clamp(F - scope_penalty, 0, 100)`

Cadence gate:

* `FC = FC * (RC/100)`

---

## 5.6 Coachability & Engagement (CE)

Purpose: identifies founders who will benefit from coaching and execute on it.

Inputs:

* `coachability_1_5`
* Optional: attendance rate (if you have it). If not, omit.

Formula:

* `CO = likert(coachability)`
* `CE = CO`

Cadence gate:

* `CE = CE * (0.6 + 0.4*RC/100)`

---

## 5.7 Risk & Resilience (RR)

Purpose: capture burnout/blockers risk.

Inputs:

* `blocker_severity_1_5`
* `sentiment_1_5` (or morale)
* `missing_update`

Formula:

* `BS = likert(blocker_severity)` (higher = worse)
* `SE = likert(sentiment)` (higher = better)

Convert blockers to “good”:

* `BS_good = 100 - BS`

Base:

* `RR = 0.6*BS_good + 0.4*SE`

Penalties:

* if `missing_update=1` then `RR = RR - 10` (extra risk signal)

Final:

* `RR = clamp(RR, 0, 100)`

---

# 6) Overall Founder Score (OFS) and Momentum

## 6.1 OFS (0–100)

Weights are tuned for accelerators: execution + learning dominate early, traction grows later (but stage normalization already helps).

```
OFS =
  0.10*RC +
  0.22*EV +
  0.20*TS +
  0.18*LI +
  0.12*FC +
  0.08*CE +
  0.10*RR
```

If any LLM feature is null:

* use neutral 50 for the corresponding likert mapping
* keep the score, but set `data_quality_flag = 1` for review.

## 6.2 Momentum (MOM)

Use a short moving baseline.

Let:

* `OFS_4w_avg` = average OFS over previous 4 weeks (or all prior weeks if fewer than 4)
* `Δ = OFS - OFS_4w_avg`

Define:

* `MOM = clamp(50 + 2.5*Δ, 0, 100)`

Interpretation:

* +10 points vs baseline → MOM = 75
* -10 points vs baseline → MOM = 25

---

# 7) Thresholds that drive actions (operationally useful)

You want thresholds that map to **clear interventions**.

## 7.1 Weekly Triage Buckets (based on OFS and RR)

**Green (on track)**

* `OFS ≥ 70` and `RR ≥ 60`

**Yellow (monitor)**

* `55 ≤ OFS < 70` OR `45 ≤ RR < 60`

**Red (intervention required)**

* `OFS < 55` OR `RR < 45` OR `missing_update=1 for 2 weeks in last 3`

Interventions:

* Red → mandatory 1:1 + ask for specific deliverable next week
* Yellow → targeted office hours + check-in
* Green → optional support, focus on leverage (intros, PR)

---

## 7.2 “Needs Help” Trigger (blockers + clarity)

Trigger when:

* `blocker_severity_1_5 ≥ 4` AND `FC < 55`
  Action:
* assign mentor + unblock plan, force 3-step next week plan

---

## 7.3 “High Leverage” Trigger (momentum)

Trigger when:

* `MOM ≥ 75` AND `EV ≥ 70`
  Action:
* give intros, distribution support, investor conversations

---

## 7.4 “Coaching ROI” Trigger (coachable but low traction)

Trigger when:

* `CE ≥ 70` AND `TS < 50`
  Action:
* focus on GTM coaching and experiment design (they will execute)

---

## 7.5 “Data Quality / Honesty Check”

Trigger when any:

* `evidence_quality_1_5 ≤ 2` AND `claims of traction present` (LLM flags mismatch)
* `missing_update=1` for 2+ weeks
* Large contradictions week-to-week (optional LLM flag)

Action:

* require evidence artifacts (links, screenshots, dashboard exports)

---

# 8) Output fields to store (recommended)

Per FounderWeek store:

* All raw metrics: `C,U,R,stage,missing_update`
* All LLM metrics + evidence quotes
* Sub-scores: `RC,EV,TS,LI,FC,CE,RR`
* `OFS, MOM`
* `triage_bucket ∈ {GREEN,YELLOW,RED}`
* Trigger booleans: `needs_help, high_leverage, coaching_roi, data_quality`

This makes the system explainable and easy to filter in Airtable.

---

# 9) Calibration rule (first 2 weeks)

Before you trust it:

1. Pick 15 founders across the cohort.
2. Have staff rate them 1–5 on the same dimensions.
3. Adjust:

   * stage targets (C/U/R)
   * weight of EV vs TS
   * penalties (scope creep, missing updates)
4. Freeze the weights for 4 weeks to avoid “moving goalposts”.

---

If you want, I can turn this spec into:

* A **Zod schema** for the LLM extraction JSON
* A **SQL / Drizzle** implementation for FounderWeek and scoring
* An Airtable-compatible formula approach (limited but workable) plus a script/automation for the LLM extraction.

