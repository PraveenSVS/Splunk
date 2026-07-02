# IDLocker Ops — Splunk Alert Catalog

Deploy alerts in Splunk **Search → paste SPL → Save As → Alert**.  
Notifications: **Webhook** → Slack Workflow → `#iam-itc-idlocker` — see [`SLACK_WORKFLOW_SPLUNK.md`](SLACK_WORKFLOW_SPLUNK.md).  
**Fallback:** Email → channel email with `$result.slack_message$`.

**Excel / CSV:** [`IDLocker_Alerts_Configuration.csv`](IDLocker_Alerts_Configuration.csv) — Webhook URL, priority, deploy order, Splunk UI settings.

**App context:** `nike_search` | **Index:** `app-gold` | **Sourcetype:** `log4j:IIQ`

---

## Shared alert settings (every alert)

| Setting | Value |
|---------|--------|
| **Permissions** | Shared in app (or Private while testing) |
| **Trigger** | Number of results **> 0** |
| **Action** | Send email → Slack channel email |
| **Email subject** | `$name$` or use table below |
| **Email message** | `$result.slack_message$` |
| **Expire** | 24 hours (default) |

**Throttle:** enable **1 hour** when the option is available. If no per-field throttle, alert-wide 1h is fine.

---

## Phase 1 — Deploy first (core ops)

| # | Splunk alert name | Description | Cron | Time range | Trigger mode | SPL source |
|---|-------------------|-------------|------|------------|--------------|------------|
| 1 | `idlocker-provisioning-spike` | Fires when any target application has ≥25 provisioning failures in 15 minutes. Indicates an integration outage or batch failure for that app. | `*/15 * * * *` | Last 15 min | For each result | `iam_slack_spike_alerts.spl` → SLACK ALERT 1 |
| 2 | `idlocker-provisioning-baseline-spike` | Fires when provisioning failures for an app are 3× the same 15m window last week AND ≥5 failures. Reduces false positives vs fixed threshold. | `*/15 * * * *` | Last 15 min | For each result | `iam_slack_spike_alerts.spl` → SLACK ALERT 2 |
| 3 | `idlocker-ops-rollup` | Single digest of top failure spikes in the last hour (provisioning, identity refresh, housekeeper, network). One message per run instead of many per-app emails. | `0 * * * *` | Last 60 min | **Once** | `iam_slack_spike_alerts.spl` → SLACK ALERT 5 |

> **Tip:** Deploy **#1** and **#3** first. Use #3 if you want one summary email; use #1 if you want immediate per-app notifications.

---

## Phase 2 — Domain-specific alerts

| # | Splunk alert name | Description | Cron | Time range | Trigger mode | SPL source |
|---|-------------------|-------------|------|------------|--------------|------------|
| 4 | `idlocker-identitizer-burst` | Identity Refresh / Identitizer errors (e.g. missing manager, lock errors) — ≥20 of the same template in 15 minutes. | `*/15 * * * *` | Last 15 min | For each result | `dashboards-and-alerts.spl` → ALERT 3 |
| 5 | `idlocker-cert-network-burst` | Certificate (PKIX/SSL) or connectivity (timeout, unknown host) errors — ≥5 in 15 minutes. Usually means integration or network outage. | `*/15 * * * *` | Last 15 min | For each result | `dashboards-and-alerts.spl` → ALERT 4 |
| 6 | `idlocker-housekeeper-failure` | Any ERROR from SailPoint Housekeeper (Perform Maintenance) in the last hour. Batch maintenance job failure. | `0 * * * *` | Last 1 hour | Once | `dashboards-and-alerts.spl` → ALERT 5 |

---

## Phase 3 — Discovery & long-tail

| # | Splunk alert name | Description | Cron | Time range | Trigger mode | SPL source |
|---|-------------------|-------------|------|------------|--------------|------------|
| 7 | `idlocker-new-error-fingerprint` | Error pattern (logger + normalized message) not seen in the prior 7 days. Catches new failure types before they spike. | `0 * * * *` | Last 1 hour | For each result | `dashboards-and-alerts.spl` → ALERT 2 |
| 8 | `idlocker-error-template-spike` | Any normalized ERROR message spiking ≥10 in 15m and ≥3× its 7-day average. Broad catch-all beyond provisioning. | `*/15 * * * *` | Last 15 min | For each result | `iam_slack_spike_alerts.spl` → SLACK ALERT 3 |
| 9 | `idlocker-edge-case-spike` | Known edge-case buckets (Workday SOAP, account selection, SCIM parse, etc.) spiking vs 7-day baseline. | `*/15 * * * *` | Last 15 min | For each result | `iam_slack_spike_alerts.spl` → SLACK ALERT 4 |
| 10 | `idlocker-novel-edge-error` | Brand-new edge error fingerprint (not seen in 14 days) with ≥2 occurrences in 1 hour. Long-tail discovery. | `0 * * * *` | Last 1 hour | For each result | `iam_edge_alerts.spl` → ALERT 7 |

---

## Phase 4 — Watchlist (optional)

| # | Splunk alert name | Description | Cron | Time range | Prerequisite | SPL source |
|---|-------------------|-------------|------|------------|--------------|------------|
| 11 | `idlocker-watchlist` | Configurable monitors from `iiq_monitor_watchlist.csv` (Workday SOAP, PassPlay, Identitizer, AD IQService, etc.). | `*/15 * * * *` | Last 15 min | Upload lookup `iiq_monitor_watchlist` to Splunk | `iam_watchlist_alert.spl` |

---

## Test alert (disable after verification)

| Splunk alert name | Description | Cron | SPL source |
|-------------------|-------------|------|------------|
| `idlocker-connectivity-test` | Dummy alert — always returns 1 row. Confirms email → Slack pipeline. **Disable after test.** | `*/5 * * * *` or disable | `iam_slack_spike_alerts.spl` → SLACK ALERT 0 |

---

## Copy-paste descriptions for Splunk UI

Use these in the **Description** field when saving each alert:

| Alert name | Description |
|------------|-------------|
| `idlocker-provisioning-spike` | IDLocker Ops: per-application provisioning failure spike (≥25 in 15m) on index app-gold. |
| `idlocker-provisioning-baseline-spike` | IDLocker Ops: provisioning failures 3× weekly baseline for same 15m window (min 5). |
| `idlocker-ops-rollup` | IDLocker Ops: hourly rollup of top IAM failure spikes across all domains. |
| `idlocker-identitizer-burst` | IDLocker Ops: Identitizer / Identity Refresh error burst (≥20 same template in 15m). |
| `idlocker-cert-network-burst` | IDLocker Ops: certificate or network/connectivity ERROR burst (≥5 in 15m). |
| `idlocker-housekeeper-failure` | IDLocker Ops: SailPoint Housekeeper (Perform Maintenance) ERROR in last hour. |
| `idlocker-new-error-fingerprint` | IDLocker Ops: new normalized error pattern not seen in prior 7 days. |
| `idlocker-error-template-spike` | IDLocker Ops: any ERROR template spiking vs 7-day average (≥10 and ≥3×). |
| `idlocker-edge-case-spike` | IDLocker Ops: classified edge-case error bucket spiking vs baseline. |
| `idlocker-novel-edge-error` | IDLocker Ops: novel edge error fingerprint (new in 14 days, ≥2 in 1h). |
| `idlocker-watchlist` | IDLocker Ops: configurable watchlist monitor match (iiq_monitor_watchlist lookup). |

---

## Deployment order

```
Week 1:  #1 idlocker-provisioning-spike
         #3 idlocker-ops-rollup          (optional — reduces noise)

Week 2:  #4 identitizer  #5 cert/network  #6 housekeeper

Week 3:  #7 new fingerprint  #8 error template spike

Week 4:  #9 edge spike  #10 novel edge  #11 watchlist (if lookup uploaded)
```

---

## Email action template (fallback)

If not using Workflow Webhook, for every alert:

| Field | Value |
|-------|--------|
| **To** | `#iam-itc-idlocker` channel email |
| **Subject** | `[IDLocker] $name$` |
| **Message / Include** | `$result.slack_message$` |

## Webhook action template (Workflow — primary)

See [`SLACK_WORKFLOW_SPLUNK.md`](SLACK_WORKFLOW_SPLUNK.md). Same **pancake URL** on every alert:

| Field | Value |
|-------|--------|
| **Action** | Webhook |
| **URL** | `splunk.webhook.url` in [`config/slack-workflow.properties`](config/slack-workflow.properties) |
| **Payload** | *(none — URL only)* |

Optional footer in SPL `slack_message`:
```
_Dashboard: IDLocker Ops_
```

---

## Tuning

| Symptom | Fix |
|---------|-----|
| Too many emails | Raise thresholds; use rollup (#3) instead of per-app (#1); throttle 2h |
| Missing real outages | Lower thresholds; add baseline alert (#2) |
| Noisy app (e.g. Slack SCIM) | Add `\| where target_application!="Slack"` to SPL |
| Alert never fires | Run SPL manually; confirm `index=app-gold` has data in time range |

---

## SPL file reference

| File | Alerts |
|------|--------|
| [`spl/iam_slack_spike_alerts.spl`](spl/iam_slack_spike_alerts.spl) | SLACK ALERT 0–5 (spike + rollup) |
| [`spl/dashboards-and-alerts.spl`](spl/dashboards-and-alerts.spl) | ALERT 1–5 (provisioning, fingerprint, identitizer, cert, housekeeper) |
| [`spl/iam_edge_alerts.spl`](spl/iam_edge_alerts.spl) | ALERT 6–7 (edge spike, novel edge) |
| [`spl/iam_watchlist_alert.spl`](spl/iam_watchlist_alert.spl) | Watchlist monitor |
