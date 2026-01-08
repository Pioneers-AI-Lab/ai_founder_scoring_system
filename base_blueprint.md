Below is a pragmatic blueprint to turn your Airtable dataset (Main SU 2025 + Weekly Forms + Feedback + Notes + Evaluations + Sessions + Exit) into an **AI-driven founder scoring system** that is operationally useful (triage, coaching allocation, advancement decisions) and defensible (auditable, bias-aware).

---

## 1) Define the scoring objective precisely

You should not start with “rank founders.” Start with **a set of program decisions you actually make**, then build scores that predict/enable those decisions.

Typical accelerator decisions:

1. **Who needs help this week?** (triage)
2. **Who is on track to pass the next gate?** (advancement likelihood)
3. **Who should get 1:1 time / intros?** (resource allocation)
4. **Who is at risk of dropping out?** (retention risk)
5. **Who is improving fastest?** (trajectory)

Translate these into **4–6 sub-scores**, then combine into an overall score.

---

## 2) Create a scoring framework (what you score)

Use a **two-layer system**:

### Layer A — Quantitative scorecards (transparent, reliable)

Derived from Weekly Forms and structured fields.

Recommended sub-scores (0–100 each):

1. **Execution Velocity**: cadence + delivered outputs + iteration pace
2. **Traction Signal**: users, conversations, revenue (stage-adjusted)
3. **Learning & Iteration**: evidence of hypothesis testing, change in plan
4. **Clarity & Focus**: coherence of goals, constraints, next steps
5. **Engagement & Coachability**: responsiveness, incorporation of feedback
6. **Risk & Friction**: blockers, morale drop, inconsistency, churn risk

### Layer B — LLM-derived “Qualitative Insights” (structured, explainable)

From Notes and free-text fields in Weekly/Feedback/Exit:

* Coachability indicators
* Specificity vs vagueness
* Customer obsession (problem interviews quality)
* Evidence quality (numbers, artifacts, links)
* Narrative coherence (“why now / why this”)

The key is: **LLM output should be structured into features**, not used as the score directly.

---

## 3) Normalize by stage (avoid punishing early-stage founders)

Founders at “idea” will look “worse” than founders already with revenue unless you normalize.

Stage buckets:

* Idea / Pre-product
* MVP
* Early traction
* Revenue / Growth

For each bucket define expected ranges for:

* user interviews/week
* active users
* revenue
* shipping cadence

Then compute a **stage-adjusted traction score** (percentile within bucket) rather than raw numbers.

---

## 4) Build the feature layer (your “founder telemetry”)

### A) Core relational join (what links to what)

Your join key is your cohort junction (`Main SU 2025`) which links:

* founder profile (`Pioneers Profile Book`)
* startup (`Startups`)
* weekly updates (`Weekly forms`)
* feedback (`Feedback Forms`)
* staff notes (`Notes`)
* eval scores (`Startup Evaluation`)
* exit outcome (`Exit Form`)

You’ll create a weekly “FounderWeek” record per founder.

### B) Example features (high signal)

**Execution Velocity**

* `weekly_submission_rate` (weeks submitted / weeks in program)
* `days_late_avg`
* `goal_completion_ratio` (if you can infer last week goals vs this week done)
* `artifact_count` (links to demo, repo, landing page updates if present)

**Traction Signal (stage-adjusted)**

* `customer_convos_7d`
* `active_users_7d`
* `revenue_7d`
* `activation_event` (first paying user, first pilot, first retention signal)

**Learning & Iteration**

* `hypothesis_statements_count` (LLM: count explicit hypotheses)
* `experiment_mentions_count` (A/B test, pricing test, interview)
* `pivot_flag` (LLM: whether direction changed)
* `insight_density` (LLM: “new learning” vs repeated narrative)

**Clarity & Focus**

* `goal_specificity` (LLM: 1–5; specific measurable goals)
* `scope_creep_flag` (LLM: too many simultaneous priorities)
* `next_steps_quality` (LLM: concrete, sequenced plan)

**Engagement & Coachability**

* `feedback_incorporation` (LLM compares feedback → next week actions)
* `attendance_rate` (if you can infer from Session_Event participation)
* `response_latency` (if available)

**Risk & Friction**

* `morale_score` (from structured form if present; else LLM sentiment)
* `blocker_severity` (LLM: 1–5)
* `inconsistency_score` (high variance in updates, contradictory claims)
* `burnout_risk` (morale drop trend + blocker severity)

---

## 5) Use the LLM correctly: “Text → JSON features”

You want the model to produce **auditable features with citations to the text**, not just “I rate them 82/100”.

### Example prompt pattern (per weekly entry)

Input: founder week text fields + relevant staff note excerpts + last week summary
Output: JSON:

```json
{
  "goal_specificity_1_5": 4,
  "evidence_quality_1_5": 3,
  "customer_obsession_1_5": 5,
  "coachability_1_5": 4,
  "blocker_severity_1_5": 2,
  "scope_creep_flag": false,
  "pivot_flag": true,
  "summary_bullets": ["...", "..."],
  "evidence_quotes": [
    {"feature":"customer_obsession_1_5","quote":"..."},
    {"feature":"pivot_flag","quote":"..."}
  ]
}
```

Critical: store `evidence_quotes` so every score is explainable.

---

## 6) Combine into a weekly score (transparent math)

Keep the final score formula simple and stable:

### Sub-scores (0–100)

* Execution Velocity (EV)
* Traction Signal (TS) stage-adjusted
* Learning & Iteration (LI)
* Clarity & Focus (CF)
* Coachability (CO)
* Risk (RK) where higher = better (low risk)

### Example weighting (adjust as needed)

* EV 25%
* TS 20%
* LI 20%
* CF 15%
* CO 10%
* RK 10%

Overall Founder Score (OFS):

```
OFS = 0.25*EV + 0.20*TS + 0.20*LI + 0.15*CF + 0.10*CO + 0.10*RK
```

Add one more metric that matters most in accelerators:

### Trajectory bonus (momentum)

Compute a “delta”:

* `ΔOFS = OFS_this_week - OFS_4wk_moving_avg`
  Then:
* **Momentum Score** = clamp(50 + 10*ΔOFS, 0, 100)

Use Momentum for “who is improving fastest” and avoid over-privileging already-strong founders.

---

## 7) Build the first version without ML training (Week 1–2)

You can get to a functional system quickly:

### Step-by-step

1. **Create a FounderWeek table** (in DB or Airtable)

   * one row per founder per week
2. Compute quantitative features from Weekly Forms + Feedback
3. Run LLM extraction for qualitative features
4. Compute sub-scores + OFS + Momentum
5. Create weekly views:

   * Top momentum
   * High risk / low morale
   * Low submission rate
   * Needs help: high blocker severity + low clarity
6. Staff reviews the outputs weekly and flags errors (for calibration)

This yields value immediately.

---

## 8) Then move to supervised models (Week 3+)

Once you have enough historical cohorts, you can train predictors:

* Predict “advance/pass” outcome (from Startup Evaluation / decisions)
* Predict dropout (Exit Form + inactivity)
* Predict “high impact” (post-program traction)

But even then, keep the *displayed* score interpretable:

* Use ML to produce “probability of advancement”
* Keep the scorecard for day-to-day coaching

---

## 9) Governance: what you must do to avoid bad outcomes

1. **Never score protected attributes** (gender, nationality, etc.)
2. **Stage normalization is mandatory** (prevents unfair ranking)
3. **Use explainability** (store evidence quotes + feature attribution)
4. **Human-in-the-loop** for any adverse action (drop / deprioritize)
5. Track error cases:

   * founders with sparse updates
   * inconsistent self-reports
   * staff notes contradicting founder claims

---

## 10) Deliverables you should implement (concrete)

### A) Tables (minimum viable)

* `Founder` (from Profile Book)
* `Startup`
* `CohortEnrollment` (Main SU 2025)
* `WeeklyUpdate`
* `Feedback`
* `StaffNote`
* `Evaluation`
* `FounderWeek` (derived)
* `FounderScore` (derived)

### B) Output dashboards/views

* Weekly “Red List” (Risk high)
* “Needs Help” (blockers high, clarity low)
* “High Momentum”
* “Quiet but strong” (low visibility, high evidence)
* “Data quality issues” (missing weeks, contradictions)

