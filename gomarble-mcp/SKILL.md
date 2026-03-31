---
name: "gomarble-mcp"
description: "Connect to GoMarble MCP to fetch live Meta Ads and Google Ads data — ROAS, CTR, spend, CPA, creative fatigue, and campaign performance. Use this skill whenever the user asks about ad account performance, wants to pull metrics, or needs to establish a baseline for monitoring."
version: "1.0.0"
author: "GoMarble"
tags:
  - ads
  - meta
  - google
  - performance-marketing
  - mcp
  - analytics
metadata: '{"openclaw":{"requires":{"env":["GOMARBLE_API_KEY"],"bins":["curl"]},"primaryEnv":"GOMARBLE_API_KEY"}}'
---

# GoMarble MCP Connector

## Purpose

This skill teaches OpenClaw how to connect to the GoMarble MCP server and retrieve live advertising performance data from Meta Ads and Google Ads accounts.

Use this skill when the user asks to:
- Pull current campaign performance (ROAS, CTR, CPA, spend)
- Check creative fatigue signals
- Establish a 7-day performance baseline for monitoring
- Compare current metrics against historical averages
- Investigate anomalies in ad account performance

---

## Connection Details

**MCP Endpoint (SSE):** `https://apps.gomarble.ai/mcp-api/sse`
**Auth:** Bearer token — use the value stored in `GOMARBLE_API_KEY`
**Protocol:** Model Context Protocol over Server-Sent Events (SSE)

Do not use REST HTTP calls directly to GoMarble tool routes. The MCP server communicates over SSE only. All tool calls must go through the MCP client connection.

---

## Instructions

### Step 1 — Establish the MCP Connection

Connect to GoMarble MCP using the SSE endpoint with Bearer auth:

```
Endpoint: https://apps.gomarble.ai/mcp-api/sse
Authorization: Bearer YOUR_GOMARBLE_API_KEY
```

### Step 2 — Fetch Available Ad Accounts

After connecting, call the appropriate MCP tool to list accessible Meta Ads and Google Ads accounts. Present account names and IDs to the user if multiple accounts are found. Ask which account(s) to monitor if more than one exists.

### Step 3 — Pull 7-Day Performance Data

For each connected ad account, fetch the following metrics for the trailing 7 days:

**Meta Ads:**
- ROAS (Return on Ad Spend) — per campaign and account-level
- CTR (Click-Through Rate) — per campaign
- Spend — daily and 7-day total
- CPA (Cost Per Acquisition / Cost Per Result)
- Creative fatigue signals — frequency, CTR decay over 7 days
- Campaigns with zero conversions in the last 24 hours

**Google Ads:**
- ROAS — per campaign and account-level
- CTR — per campaign
- Spend — daily and 7-day total
- CPA — per campaign
- Campaigns with zero conversions in the last 24 hours

### Step 4 — Calculate and Store the Baseline

After retrieving the 7-day data, calculate averages for each metric to establish the monitoring baseline:

```
baseline.meta.roas = 7-day average account ROAS
baseline.meta.ctr = 7-day average account CTR
baseline.meta.daily_spend = 7-day average daily spend
baseline.meta.cpa = 7-day average CPA

baseline.google.roas = 7-day average account ROAS
baseline.google.ctr = 7-day average account CTR
baseline.google.daily_spend = 7-day average daily spend
baseline.google.cpa = 7-day average CPA

baseline.last_updated = [current timestamp]
```

Save this baseline to a local file at: `~/.openclaw/gomarble-baseline.json`

If the file already exists, check whether it was last updated more than 7 days ago. If so, offer to refresh the baseline with current data.

### Step 5 — Confirm to the User

After storing the baseline, confirm:
- Which accounts were connected
- The baseline values computed
- That proactive monitoring is now ready (if the gomarble-monitor skill is also installed)

---

## Data the GoMarble MCP Can Return

GoMarble MCP gives access to:

**Meta Ads:**
- Campaign-level performance (impressions, clicks, spend, ROAS, CTR, CPA, frequency)
- Ad set performance
- Creative-level metrics (by ad ID)
- Account-level aggregated metrics
- Date-range sliced data

**Google Ads:**
- Campaign-level performance (impressions, clicks, spend, ROAS, CTR, CPA)
- Ad group performance
- Account-level aggregated metrics
- Date-range sliced data

**Attribution & Date Slicing:**
GoMarble normalizes attribution windows across platforms. When pulling data, always request last-click attribution for consistency unless the user specifies otherwise.

---

## Constraints

- Never hardcode or log the API key. Always read it from the `GOMARBLE_API_KEY` environment variable.
- Do not make direct REST calls to individual GoMarble tool routes — always use the MCP SSE connection.
- Do not attempt to modify or write data back to ad accounts. This skill is read-only.
- If the MCP connection fails, suggest the user verify their API key at apps.gomarble.ai and check that their account has MCP access enabled.
- Baseline data should be refreshed at the user's request or automatically after 7 days.

---

## Error Handling

| Error | Likely Cause | What to Do |
|---|---|---|
| 401 Unauthorized | Invalid or expired API key | Ask user to generate a new key at apps.gomarble.ai |
| 403 Forbidden | MCP access not enabled on account | Direct user to enable MCP in GoMarble settings |
| No accounts returned | Account not connected to GoMarble | Ask user to connect their ad account in the GoMarble dashboard |
| Empty metrics | Date range too narrow or account newly connected | Widen the date range or wait 24h for data to populate |
| SSE connection drops | Network instability | Retry once automatically; notify user if retry also fails |
