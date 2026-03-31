---
name: "gomarble-monitor"
description: "Proactively monitor Meta Ads and Google Ads performance every 3 hours without being asked. Compares live data from GoMarble MCP against stored baseline and sends a WhatsApp alert if ROAS drops, CTR crashes, spend spikes, CPA rises, or any campaign goes dark while still spending. Install this skill to make OpenClaw your always-on ad account watchdog."
version: "1.0.0"
author: "GoMarble"
tags: ["ads", "monitoring", "alerts", "meta", "google", "whatsapp", "automation", "mcp"]
metadata: {
  "openclaw": {
    "requires": {
      "env": ["GOMARBLE_API_KEY"],
      "bins": ["curl"],
      "config": ["message.enabled"]
    },
    "primaryEnv": "GOMARBLE_API_KEY"
  }
}
---

# GoMarble Proactive Ad Monitor

## Purpose

This skill turns OpenClaw into a proactive ad account watchdog. It runs on a 3-hour schedule — without the user asking — fetches live ad performance data from GoMarble MCP, compares it against the stored baseline, and sends a WhatsApp alert if any metric crosses a threshold.

This is different from query-based use of GoMarble. The user does NOT need to ask anything. OpenClaw checks on its own and only surfaces a message when something needs attention.

Use this skill when the user wants:
- Passive background monitoring of their ad accounts
- Instant WhatsApp alerts when performance drops
- To stop finding out about problems hours after they started
- An always-on system that watches accounts 24/7

---

## Prerequisites

Before this skill will work, the following must be set up:
1. `gomarble-mcp` skill is installed and a baseline file exists at `~/.openclaw/gomarble-baseline.json`
2. `GOMARBLE_API_KEY` environment variable is set
3. OpenClaw's message/WhatsApp channel is connected and enabled (`message.enabled` config is true)
4. The user's WhatsApp number is configured in OpenClaw's messaging settings

---

## Instructions

### Monitoring Schedule

Run this monitoring check automatically every 3 hours. Do not wait for the user to ask.

Use OpenClaw's built-in cron/scheduler to trigger this workflow at the following interval:
```
every 3 hours
```

If the scheduler is not available on this OS, inform the user and suggest they use the n8n fallback workflow (see Fallback section below).

---

### Step 1 — Load the Baseline

Read the stored baseline from `~/.openclaw/gomarble-baseline.json`.

If the file does not exist, do NOT proceed with comparison. Instead, send the user a WhatsApp message:

> ⚠️ GoMarble Monitor: No baseline found. Send me "check my ads" to set up monitoring properly.

If the baseline is older than 7 days, send the user a WhatsApp message:

> 📊 GoMarble Monitor: Your baseline is 7+ days old. Send me "refresh baseline" to update it.

Then proceed with the outdated baseline (better to alert than to go silent).

---

### Step 2 — Fetch Current Performance Data

Connect to GoMarble MCP (`https://apps.gomarble.ai/mcp-api/sse`) using Bearer auth (`GOMARBLE_API_KEY`).

Fetch the following for the **last 3 hours** for each connected ad account:

- ROAS (account-level and per active campaign)
- CTR (account-level and per active campaign)
- Spend (actual vs. expected based on daily budget paced to 3-hour window)
- CPA (account-level and per active campaign)
- Conversion count per campaign (to detect zero-conversion campaigns)

"Active campaign" = any campaign that had spend in the last 24 hours.

---

### Step 3 — Compare Against Baseline

Run the following checks. For each, flag if the threshold is breached:

**1. ROAS Drop**
```
Threshold: current ROAS < (baseline ROAS × 0.80)
Meaning: ROAS has dropped 20%+ below 7-day average
```

**2. CTR Crash**
```
Threshold: current CTR < (baseline CTR × 0.85)
Meaning: CTR has dropped 15%+ below 7-day average
```

**3. Spend Spike**
```
Threshold: current 3-hour spend > (expected 3-hour spend × 1.15)
Expected 3-hour spend = (daily budget / 24) × 3
Meaning: Spend is running 15%+ above expected pace
```

**4. CPA Spike**
```
Threshold: current CPA > (baseline CPA × 1.25)
Meaning: CPA has increased 25%+ above 7-day average
```

**5. Zero-Conversion Campaign Still Spending**
```
Threshold: campaign has 0 conversions in last 3 hours AND spend > $0 in last 3 hours
Only flag if: campaign has historically generated conversions (at least 1 conversion in the 7-day baseline period)
```

---

### Step 4 — Decide Whether to Alert

**If NO thresholds are breached:** Do nothing. Do not send any message. Silent is correct behavior when everything is healthy.

**If ONE OR MORE thresholds are breached:** Proceed to Step 5.

Do not send a repeat alert for the same campaign/metric combination within 3 hours. Track which alerts have been sent in a local state file at `~/.openclaw/gomarble-alert-log.json`. Update this file after each alert.

---

### Step 5 — Send WhatsApp Alert

For each threshold breach, send a WhatsApp message in the following format:

```
🚨 Ad Alert — [Platform]: [Campaign Name]

What happened: [Metric] dropped/spiked [X%] vs. your 7-day average.

Current: [value]
Your baseline: [value]

Likely cause: [one specific likely cause based on the metric and context]

Recommended action: [one specific recommended action]

— GoMarble Monitor
```

**Platform options:** Meta Ads / Google Ads

**Alert format examples:**

For ROAS drop:
```
🚨 Ad Alert — Meta Ads: Summer Sale — Retargeting

What happened: ROAS dropped 34% vs. your 7-day average.

Current ROAS: 1.4x
Your baseline: 2.1x

Likely cause: Audience fatigue — frequency is likely elevated on this ad set, reducing purchase intent among people who've seen it multiple times.

Recommended action: Check frequency in Ads Manager. If above 3.5, duplicate the ad set with a refreshed creative or expand the audience.

— GoMarble Monitor
```

For spend spike:
```
🚨 Ad Alert — Google Ads: Brand Keywords — Exact Match

What happened: Spend is running 28% above expected pace for this time window.

Current 3h spend: $340
Expected 3h spend: $265

Likely cause: Competitor bidding activity may have increased auction competition, pushing your bids higher to maintain position.

Recommended action: Check Auction Insights in Google Ads. If competitor impression share is up, consider adjusting target CPA or setting a max CPC cap.

— GoMarble Monitor
```

For zero-conversion campaign:
```
🚨 Ad Alert — Meta Ads: New Customer Acquisition — Broad

What happened: This campaign has spent $47 in the last 3 hours with 0 conversions.

Spend without conversion: $47 over 3 hours
Historical conversion rate: ~1 per $28 spend

Likely cause: The landing page may be down, or a pixel event is misfiring — purchases are happening but not being tracked, or the campaign has entered a learning phase reset.

Recommended action: Check your landing page loads correctly and verify pixel is firing on the confirmation/thank-you page. Also check Meta Events Manager for recent drop in event activity.

— GoMarble Monitor
```

---

### Step 6 — Update the Alert Log

After sending any alert, update `~/.openclaw/gomarble-alert-log.json` with:
```json
{
  "last_check": "[ISO timestamp]",
  "alerts_sent": [
    {
      "timestamp": "[ISO timestamp]",
      "platform": "Meta Ads | Google Ads",
      "campaign": "[campaign name]",
      "metric": "roas | ctr | spend | cpa | zero_conversions",
      "current_value": [number],
      "baseline_value": [number]
    }
  ]
}
```

---

### Step 7 — Log the Check (Even if No Alert)

After every check (alert or no alert), append to a lightweight log at `~/.openclaw/gomarble-check-log.txt`:
```
[ISO timestamp] — Check complete. Alerts sent: [N]. Next check in 3 hours.
```

This gives the user a way to confirm the monitor is running without flooding them with messages.

---

## Fallback — If Proactive Scheduling is Not Available

On some OS configurations, OpenClaw's background task scheduler may not run reliably. If the user reports that monitoring isn't triggering automatically, recommend the n8n fallback workflow.

Tell the user:
> "Proactive scheduling requires OpenClaw's daemon to be running. If you're having trouble, there's a fallback n8n workflow that does the same thing. Comment CLAW on the LinkedIn post or visit the setup guide for the n8n version."

---

## Constraints

- Never send an alert for healthy performance. Silent = good. Noisy = ignored.
- Never repeat the same alert for the same campaign+metric combination within 3 hours.
- Never modify or pause ad campaigns. This skill is read-only.
- Never fabricate data. If GoMarble MCP returns no data for a period, skip the check and log it — do not extrapolate or guess.
- Likely cause and recommended action must be grounded in the actual metric and campaign type. Do not use generic filler like "check your campaigns."
- Keep WhatsApp messages short enough to read in 30 seconds. No walls of text.
