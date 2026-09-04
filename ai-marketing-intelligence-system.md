# AI Marketing Intelligence System
### Architecture & Build Plan (n8n-first, no-code/low-code)

---

## 1. Executive Assessment

**Is the original architecture fundamentally sound?** Partially. The instinct — collect data, analyze it, use an LLM for interpretation, classify, recommend, report — is the right shape of a marketing intelligence system. But the diagram as drawn has one structural error and several missing layers.

**The structural error:** `Google Ads → GA4 → n8n`

This implies GA4 sits *downstream* of Google Ads — that GA4 depends on Ads data, or that Ads data flows into GA4 before anything else happens. That's not how these two systems relate. They are **two independent data sources** that should be pulled **in parallel** and joined later, at the storage/analysis layer, on shared dimensions (date, campaign, source/medium). Chaining them in series creates a false dependency: if the Ads pull fails or is slow, GA4 never runs; if you ever add a third source, the whole thing turns into an unmanageable daisy chain. It also obscures the actual hard problem, which isn't fetching data — it's **reconciling two systems that count conversions differently** (Ads uses its own attribution model; GA4 uses its own, usually data-driven or last-click-non-direct). That reconciliation needs to be an explicit step, not an afterthought of API ordering.

**What's missing entirely:**
- No ingestion/storage layer — the diagram jumps from raw sources straight to "LLM," implying the LLM sees raw data. It shouldn't.
- No deterministic metrics/statistics layer — percentage changes, CPA, ROAS, and anomaly thresholds need to be calculated *before* anything touches an LLM, not inferred by it.
- No data validation/QA step — nothing catches a broken conversion tag or a zero-spend day before it's "interpreted" as a finding.
- No human review gate before recommendations reach the marketer as if they were vetted facts.
- No outcome tracking or feedback loop — without this, the system can never learn whether its recommendations were any good.

**What's redundant as drawn:** `LLM → Classification` as two separate stages. In practice this is one LLM call that returns structured output including a classification field. Splitting it into two n8n workflow stages adds orchestration overhead for no benefit at V1 scale.

**Bottom line:** keep the *intent* of your pipeline, discard the *literal shape* of it. The fix is below.

---

## 2. Recommended Architecture

```text
┌─────────────────────────────────────────────────────────────────┐
│  DATA SOURCES (parallel, independent)                           │
│  Google Ads API        GA4 Data API                              │
└─────────┬───────────────────────┬───────────────────────────────┘
          │                       │
          ▼                       ▼
┌─────────────────────────────────────────────────────────────────┐
│  INGESTION  (n8n, scheduled)                                     │
│  Pull → validate response → tag with run_id/date                │
└─────────────────────────────┬────────────────────────────────────┘
                               ▼
┌─────────────────────────────────────────────────────────────────┐
│  STORAGE — raw layer  (Airtable, see §6/§7)                       │
└─────────────────────────────┬────────────────────────────────────┘
                               ▼
┌─────────────────────────────────────────────────────────────────┐
│  NORMALIZATION  (deterministic)                                  │
│  Common schema, shared keys (date, campaign, source/medium)     │
└─────────────────────────────┬────────────────────────────────────┘
                               ▼
┌─────────────────────────────────────────────────────────────────┐
│  VALIDATION / QA  (deterministic rules)                          │
│  Nulls, zero-spend anomalies, schema drift, duplicate runs      │
└─────────────────────────────┬────────────────────────────────────┘
                               ▼
┌─────────────────────────────────────────────────────────────────┐
│  ANALYTICS  (deterministic math)                                 │
│  CPC, CPA, ROAS, CTR, conv. rate, period-over-period % change  │
└─────────────────────────────┬────────────────────────────────────┘
                               ▼
┌─────────────────────────────────────────────────────────────────┐
│  ANOMALY / CHANGE DETECTION  (statistical, deterministic)         │
│  Thresholds, z-scores/rolling baseline, significance floor       │
└─────────────────────────────┬────────────────────────────────────┘
                               ▼
                    [ Findings — structured, not yet "meaning" ]
                               ▼
┌─────────────────────────────────────────────────────────────────┐
│  LLM INTERPRETATION + CLASSIFICATION  (one structured call)       │
│  Likely cause, taxonomy tag, confidence, draft recommendation    │
└─────────────────────────────┬────────────────────────────────────┘
                               ▼
┌─────────────────────────────────────────────────────────────────┐
│  RECOMMENDATION SCORING  (deterministic formula)                 │
│  Priority = Impact × Confidence × Urgency ÷ Effort               │
└─────────────────────────────┬────────────────────────────────────┘
                               ▼
┌─────────────────────────────────────────────────────────────────┐
│  HUMAN REVIEW  (approval queue)                                  │
└───────────────┬───────────────────────────────┬─────────────────┘
                ▼                               ▼
    ┌───────────────────────┐        ┌───────────────────────┐
    │ ACTION / AUTOMATION    │        │ REPORTING / DASHBOARD  │
    │ (V3, gated by approval)│        │ (all findings, always) │
    └───────────┬───────────┘        └───────────────────────┘
                ▼
┌─────────────────────────────────────────────────────────────────┐
│  OUTCOME TRACKING → FEEDBACK LOOP  (V2/V3)                        │
│  Did the metric move the way we expected after the action?      │
└─────────────────────────────────────────────────────────────────┘
```

**Layers deliberately combined** (don't build these as separate n8n workflows):
- *Normalization + Validation/QA* → one workflow, since QA rules need the normalized schema anyway.
- *LLM Interpretation + Classification* → one LLM call with structured JSON output, not two.
- *Recommendation generation + Prioritization* → the LLM drafts the recommendation and estimates impact/confidence; a deterministic n8n Function node computes the actual priority score. Never let the LLM output a final priority number directly — it will be inconsistent run to run.

---

## 3. Why This Architecture Is Better

1. **Sources are parallel, not chained.** Google Ads and GA4 are pulled independently and joined at the storage layer on `date + campaign + source/medium`. This removes the false dependency and makes it trivial to add Search Console or Meta Ads later without restructuring anything.
2. **The LLM never does arithmetic.** Every number (CPA, ROAS, % change, z-score) is computed deterministically before the LLM sees it. This is the single biggest lever against hallucinated or subtly-wrong recommendations — LLMs are unreliable at precise math and *will* occasionally invent a plausible-looking but wrong percentage.
3. **Anomaly detection is a distinct, deterministic stage.** "Is this different enough to matter" is a statistics question (threshold, rolling baseline, minimum sample size), not a semantic-reasoning question. Only once something clears a real threshold does it become a "finding" worth spending an LLM call on.
4. **One LLM call, structured output.** Interpretation and classification are merged into a single call that returns strict JSON (schema in §8). This halves LLM cost and orchestration complexity versus treating them as separate workflow stages, and avoids the classic failure mode of two LLM calls disagreeing with each other about the same finding.
5. **Human review sits between "recommendation" and "action," always.** Nothing changes a budget, pauses a campaign, or edits targeting without a person clicking approve — even in the most agentic version of this system (§14).
6. **Outcome tracking closes the loop without pretending the LLM "learns."** The system gets smarter by *feeding better historical context into future prompts and priority scoring*, not by retraining anything.

---

## 4. Complete Data Flow

```text
Raw pull (Ads + GA4)
   → Raw storage (immutable, timestamped)
   → Normalized rows (common schema)
   → QA pass (reject/flag broken rows)
   → Computed metrics (CPA, ROAS, CTR, Δ% vs prior period)
   → Anomaly flags (threshold/z-score exceeded?)
   → Finding record created (structured, deterministic fields only)
   → LLM call: finding → {likely_cause, classification, confidence,
                            draft_recommendation, expected_impact}
   → Recommendation record created
   → Deterministic priority score attached
   → Human review queue (approve / reject / edit)
   → Approved → report + (V3) automated action
   → Outcome measured N days later → written back to the finding/
     recommendation record → available as context for future LLM calls
```

The critical discipline: **findings are facts, recommendations are opinions.** A finding ("CPA on Campaign X rose 43% week-over-week, n=212 conversions") is deterministic and never touched by the LLM's judgment. A recommendation ("likely the new landing page, consider reverting") is the LLM's interpretation, always labeled as such, and always reviewable against the underlying finding.

---

## 5. n8n Workflow Architecture

| # | Workflow | Trigger | Purpose |
|---|----------|---------|---------|
| **W1** | Data Ingestion | Schedule (daily) | Pull Google Ads + GA4 **in parallel branches**, tag each pull with a run ID and date range, write to raw storage tables. Fails loud (Slack alert) on auth/quota errors — does not fail silently into W2. |
| **W2** | Normalization & QA | Triggered by W1 success | Map both sources to a common schema; run deterministic checks (nulls, zero-spend days, duplicate run IDs, expected-row-count sanity check); flag/quarantine bad rows rather than silently dropping them. |
| **W3** | Metrics & Anomaly Detection | Triggered by W2 success | Compute CPC/CPA/ROAS/CTR/conv. rate; compare to prior period and trailing baseline; apply thresholds + minimum sample size floor; write **Finding** records only for what clears the bar. |
| **W4** | AI Interpretation & Classification | Triggered when new Findings exist | Assemble context (finding + relevant history from Outcomes table) → single LLM call → structured JSON → validate JSON against schema → write **Recommendation** record. Invalid/malformed LLM output is retried once, then flagged for human review rather than discarded. |
| **W5** | Prioritization & Human Review Queue | Triggered by W4 success | Deterministic Function node computes priority score; pushes new items into the Airtable "Review Queue" view; sends a digest ping to Slack (not full content — just "N new items to review"). |
| **W6** | Reporting & Dashboard Sync | Schedule (weekly + monthly) | Pull approved/all findings for the period → LLM narrative pass (writing only, no new numbers) → push to Looker Studio source table + send formatted Slack/email digest. |
| **W7** *(V2)* | Outcome Tracking | Schedule (e.g., 14 days after any approved action) | Re-pull the relevant metric, compare actual vs. expected impact, write outcome back to the Recommendation record. |
| **W8** *(V3)* | Action Execution | Triggered by explicit human approval click | Executes only pre-approved, whitelisted action types (e.g., pause keyword, adjust budget by ≤X%) via the Google Ads API; logs the action; hands off to W7 for measurement. |

Keep W1–W6 as the entire V1/V2 build. Do not start W8 until W7 has been running long enough to trust the outcome data.

---

## 6. Tool Stack

| Layer | Best option | Why | Alternative | Upgrade trigger |
|---|---|---|---|---|
| Orchestration | **n8n** | Visual, self-hostable or cloud, native nodes for Google Ads, GA4, Airtable, Slack, HTTP | Make | Only switch if you need something n8n genuinely can't do — unlikely here |
| Data sources | **Google Ads API + GA4 Data API** (native n8n nodes / HTTP Request node) | Direct, no middleman cost | Supermetrics/Fivetran connector | When you add 3+ more channels and want managed connectors instead of maintaining auth yourself |
| Storage | **Airtable** | Doubles as database *and* the human-review UI (views, statuses, comments) — no separate approval tool needed at V1 | Google Sheets (prototype only), Supabase/Postgres (V2) | Migrate when you exceed ~30–50k records, need real joins/window functions, or Airtable API rate limits start hurting |
| LLM | **Claude or GPT-4-class model via API (JSON mode / structured output)** | Strong structured-output reliability, good at synthesis + classification | Gemini | Multi-model only if you need cost-tiering (cheap model for routine, strong model for strategic monthly report) |
| Dashboard | **Looker Studio** connected to Airtable/Sheets export | Free, marketer-native, no build cost | Retool (if you want interactive approve/reject in a real UI) | When Airtable's native views stop being "enough" for stakeholders outside your team |
| Alerts / digest | **Slack** (or email via Gmail node) | Push-based, matches how marketers already triage attention | — | — |
| Human approval | **Airtable views + button field → n8n webhook** | Zero extra tooling; a marketer clicks "Approve" in a familiar table UI | Slack interactive buttons | If approvals need to happen fast/on mobile, add Slack buttons in V2 |

**Explicitly not recommended for V1:** BigQuery, Zapier, a custom Postgres server, Supabase. All are good tools — none earn their complexity yet. BigQuery in particular is overkill until you have genuine multi-million-row, multi-year data or need to run it alongside other company data warehousing.

---

## 7. Data Model

Three core tables carry the whole system. Keep it to three for V1 — don't build ten tables you don't have data for yet.

**`Findings`** (one row = one deterministic, evidence-based observation)

| Field | Type | Notes |
|---|---|---|
| Finding ID | auto | |
| Timestamp | date | |
| Data Source | select (Ads/GA4) | |
| Entity / Entity Type | text/select | campaign, ad group, keyword, etc. |
| Metric | select | CPA, ROAS, CTR, conv. rate, spend, impressions |
| Current Value / Previous Value / % Change | number | computed, never LLM-touched |
| Sample Size | number | needed to judge statistical meaningfulness |
| Anomaly Flag | boolean | did it clear the threshold |
| Run ID | text | traceability back to W1 pull |

**`Recommendations`** (one row = one LLM interpretation + human decision)

| Field | Type | Notes |
|---|---|---|
| Linked Finding | link | |
| Classification | select | taxonomy, §8 |
| Likely Cause | text | LLM output |
| Confidence | number 1–5 | LLM output |
| Recommended Action | text | LLM output |
| Expected Impact | text/number | LLM qualitative estimate, mapped to 1–5 |
| Effort | number 1–5 | LLM estimate or human-set default |
| Priority Score | number | **deterministic formula**, never LLM-set |
| Human Review Required | boolean | true unless in the pre-approved safe list |
| Status | select | Pending / Approved / Rejected / Actioned |
| Rejection Reason | text | optional, valuable for feedback loop |

**`Outcomes`** (V2 — one row = what actually happened after a recommendation was acted on)

| Field | Type | Notes |
|---|---|---|
| Linked Recommendation | link | |
| Metric Before / After | number | re-pulled from the same source |
| Expected vs. Actual Direction | select | matched / partial / missed |
| Notes | text | |

**Fields to defer past V1:** don't build "Attribution Issue" sub-fields, geographic/device breakdowns, or a separate Evidence-blob field until you've seen a few weeks of real Findings and know what you actually reference.

---

## 8. AI/LLM Architecture

**What the LLM does:** interprets a pre-computed finding, assigns a taxonomy classification, estimates likely cause/confidence/impact/effort in qualitative terms, and drafts a recommendation and a plain-language explanation for the report.

**What the LLM never does:** compute percentages, CPA/ROAS, statistical thresholds, or the final priority score. All of that is Function-node arithmetic before the finding ever reaches the LLM.

**Input → reasoning → output contract** (single call, per finding or small batch of related findings):

```json
// INPUT to LLM
{
  "entity": "Campaign: Brand - Search",
  "metric": "CPA",
  "current_value": 42.10,
  "previous_value": 29.40,
  "pct_change": 43.2,
  "sample_size": 212,
  "period": "week over week",
  "recent_history": "3 similar CPA spikes in past 90 days, 2 resolved by landing page fix",
  "known_context": ["No tracking anomalies detected this period"]
}
```

```json
// OUTPUT from LLM (strict schema, validated in n8n before storage)
{
  "classification": "Performance Issue",
  "likely_cause": "Landing page change deployed 2 days before CPA increase began",
  "confidence": 4,
  "recommended_action": "Compare converting landing page variant before/after; consider reverting if confirmed",
  "expected_impact": "high",
  "effort": "low",
  "explanation_for_report": "..."
}
```

n8n validates this JSON against the schema; malformed output is retried once, then routed to human review rather than silently discarded or force-parsed. **Never** feed the LLM raw campaign-level rows across dozens of entities in one call and ask it to "find the interesting stuff" — that's exactly the anomaly-detection job that belongs in deterministic W3, and it's also where hallucinated findings creep in.

---

## 9. Human-in-the-Loop Architecture

| Automatically allowed (no gate) | Requires human approval |
|---|---|
| Generate reports, summarize performance | Change budgets |
| Flag anomalies, surface findings | Pause/resume campaigns |
| Draft recommendations | Modify bidding strategy or targeting |
| Classify findings | Add/remove negative keywords |
| Compute priority scores | Launch new campaigns |
| | Modify conversion tracking |

One addition to your original split: **a "tracking failure" finding is never just queued for routine review — it's flagged urgently**, because if conversion tracking is broken, every other finding and recommendation generated downstream is built on bad data. Treat data-quality findings as a hard-override priority tier, separate from the normal Impact × Confidence × Urgency ÷ Effort scoring.

Even in the most agentic version of this system (V3, action execution), the default posture is **propose + one-click approve**, not silent execution — full autonomy is something to earn over months of validated outcome data, not something to design in from day one.

---

## 10. Dashboard & Reporting

- **Real-time/operational dashboard** (Looker Studio): raw + computed metrics, always current, for "how are we doing right now" glance-checks. No LLM content here — just numbers and trend lines.
- **Weekly intelligence report** (Slack/email, LLM-narrated): the findings + recommendations generated that week, written in plain language. This is where insight lives.
- **Monthly strategic report**: same pipeline, longer lookback, a prompt oriented toward "what should change about strategy" rather than "what changed this week."

Improved executive summary template:

```text
MARKETING INTELLIGENCE — WEEK OF [DATE]

Overall Status: ███████░░░  (7/10 — stable, one issue needs attention)

⚠ Data Quality Note: [only appears if tracking/data issues detected —
   always shown first, before anything else, if present]

3 Important Changes
1. CPA on Brand Search up 43% WoW (n=212) — likely cause: landing page change
2. ROAS on Retargeting up 18% WoW — no action needed, monitor
3. Impressions down 22% on Display — pacing issue, budget exhausted early

Top Opportunities
1. Keyword "X" converting at 2x account average, currently budget-capped
2. Audience segment Y underspent relative to performance

Potential Problems
1. Landing page suspected cause of #1 above — needs verification

Recommended Actions (requires your approval)
1. Increase budget cap on keyword "X" by $200/day
2. Revert or A/B test the new landing page variant

Requires Human Review
1. [action items above] — approve/reject in the Review Queue: [link]
```

The one addition versus your draft: a data-quality line that always surfaces first if present, because a marketer acting on recommendations built on broken tracking is the single worst outcome this system could produce.

---

## 11. Feedback Loop

```text
Recommendation (Status: Approved)
   → Action taken (manually, or via W8 in V3)
   → Wait N days (metric-appropriate — e.g. 14 days for CPA, 30 for ROAS)
   → Outcome Tracking workflow re-pulls the metric
   → Compare actual change vs. expected_impact from the Recommendation
   → Write to Outcomes table: matched / partial / missed
   → Future LLM calls receive this history as context
     ("last 3 times we saw a similar CPA spike and reverted the landing
      page, it resolved 2/3 times")
```

This is **not** model retraining — it's giving the LLM better *context* at inference time (recent_history field in §8) and letting the human reviewer see a track record ("this type of recommendation has been right 70% of the time") next to new recommendations. That's enough to materially improve decision quality without any ML infrastructure.

Also record **rejections and reasons** — "rejected: seasonal, not a real issue" is just as valuable as a confirmed win, because it teaches the reviewer (and eventually the system's context) what *not* to flag as urgently next time.

---

## 12. Critical Gaps & Risks

| Risk | Rank | Notes |
|---|---|---|
| Attribution mismatch between Ads and GA4 conversion counts | **Critical** | Different attribution models will show different numbers for the "same" conversion. Pick one source of truth per metric and document it, don't average them. |
| Tracking failures masquerading as performance issues | **Critical** | A broken conversion tag looks exactly like a real CPA spike. This is why data-quality findings get a hard-override priority. |
| Hallucinated causes from the LLM | **Critical** | Mitigated by never letting the LLM see raw data or invent numbers — but "likely cause" is still a guess. Always label it as such in the report, never as fact. |
| Small sample sizes / statistical significance | **High** | A keyword with 4 conversions moving from 2% to 4% CVR is noise, not a finding. Enforce a minimum sample-size floor in W3. |
| Seasonality confused with anomalies | **High** | Compare against the same period last year in addition to week-over-week, where data history allows. |
| Data latency (GA4 conversions can finalize 24–48h late) | **High** | Don't compute "final" metrics on same-day GA4 data; reprocess after a lag window. |
| API rate limits / quota (Google Ads especially) | **Medium** | Schedule pulls during off-peak, batch requests, monitor quota usage in W1. |
| Auth token expiry | **Medium** | OAuth refresh token handling in n8n credentials; alert on auth failure rather than silent skip. |
| Duplicate ingestion runs (double-counted spend/conversions) | **Medium** | Run ID + idempotency check in W2. |
| Schema drift (Google changes API response fields) | **Medium** | QA layer should flag unexpected/missing fields rather than silently mapping nulls. |
| LLM cost creep as finding volume grows | **Medium** | Batch related findings into fewer calls; use a cheaper model for routine classification, reserve stronger models for monthly strategic synthesis. |
| Recommendation drift (system quietly gets worse over time) | **Medium** | This is exactly what the Outcomes table and rejection-reason logging in §11 catch — without it, drift is invisible. |
| Approval fatigue (too many low-value items in the queue) | **Low–Medium** | Priority scoring should suppress anything below a "worth a human's time" floor into a digest-only mention rather than the review queue. |
| Currency/timezone mismatches between Ads account and GA4 property | **Low** | Verify both are configured identically before trusting joined data. |

---

## 13. MVP — Phase 1

**Build:** W1 → W2 → W3 → W4 → W5 → W6 (skip W7/W8 entirely).

- **Data sources:** Google Ads + GA4 only.
- **Storage:** Airtable, three tables (Findings, Recommendations, and a simple raw-pull log).
- **LLM:** one model, one structured-output call per finding.
- **Outputs:** Weekly Slack digest + a Looker Studio dashboard on top of Airtable.
- **Human review:** Airtable view with a status field, no fancy interactive buttons yet.
- **Estimated complexity:** roughly a 2–3 week build for someone comfortable in n8n, most of it in getting the Google Ads/GA4 API auth and field mapping right — not in the LLM step.

**Explicitly do NOT build yet:** outcome tracking, automated actions, cross-channel data sources, a custom dashboard, Slack interactive approval buttons, or a sophisticated taxonomy with a dozen sub-fields. All of these are premature before you've seen a few weeks of real findings.

---

## 14. V2 / V3 Evolution

**V2 — Intelligence:**
- Add W7 (Outcome Tracking) — this is the single highest-leverage V2 addition.
- Add Google Search Console as a third parallel source (fills the gap between "what ad drove the click" and "what organic query almost did").
- Migrate storage to Supabase/Postgres if Airtable is straining (record count, rate limits, or you need real joins for cross-source analysis).
- Add richer classification (seasonal/external-factor tag to reduce false anomaly attribution).
- Add Slack interactive approve/reject buttons to reduce friction.

**V3 — Agentic Operations:**
- Add W8 (Action Execution) for a **small, explicit whitelist** of actions (e.g., pause a keyword, adjust a budget within a capped %) — never an open-ended "do whatever you think is best" action space.
- Add Meta Ads / Microsoft Ads / CRM (HubSpot) as additional parallel sources once the Ads+GA4 core has proven itself.
- Increase autonomy gradually: start every action type at "propose + approve," and only consider auto-executing a specific, narrow action type after months of outcome data show it's reliably correct — and even then, keep a kill switch.

```text
Level 1 — Automated Reporting        Data → Metrics → Report
Level 2 — Marketing Intelligence     Data → Analysis → Findings → Recommendations
Level 3 — Agentic Marketing Ops      Monitor → Detect → Diagnose → Recommend →
                                       Approve → Execute → Measure → Learn
```

Capabilities required at each level: L1 needs only ingestion + deterministic metrics. L2 adds anomaly detection + LLM interpretation + human review (this document's MVP). L3 additionally requires a whitelisted action layer, outcome tracking mature enough to trust, and an explicit rollback/kill-switch mechanism for every automated action type.

---

## 15. Recommended Build Order

1. Set up Google Ads API + GA4 Data API credentials in n8n; build W1 (parallel pulls) writing to two raw Airtable tables. Verify data lands correctly for a week before building anything else.
2. Build W2: normalization to a common schema + QA checks (nulls, duplicate runs, zero-spend flags).
3. Build W3: deterministic metric calculations + period-over-period comparison + a simple threshold-based anomaly flag (start with fixed thresholds, e.g. ">25% change and >30 conversions," refine later).
4. Design and lock the Findings/Recommendations schema in Airtable (§7) before writing any LLM prompts — the schema is the contract the whole system depends on.
5. Build W4: the single structured-output LLM call, with strict JSON validation and a retry-then-flag fallback.
6. Build W5: deterministic priority scoring + Airtable review queue view.
7. Build W6: weekly Slack/email digest + Looker Studio dashboard.
8. Run the full pipeline for 3–4 weeks on real data before touching V2. Use this period to tune anomaly thresholds and see what the taxonomy actually needs.
9. Only then: add W7 (outcome tracking), then evaluate whether Airtable is still comfortable or it's time to migrate storage.
10. Treat W8 (automated action execution) as a distinct future project, not a checkbox on this build — it deserves its own approval-safety design pass once you have outcome data to trust.
