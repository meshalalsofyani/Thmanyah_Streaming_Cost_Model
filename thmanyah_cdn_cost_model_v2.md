# CDN Cost Model

**Version:** 2.0
**Scope:** CDN providers for OTT streaming delivery
**Purpose:** Per-match CDN cost calculation, provider rate reference, and traffic distribution logic for the streaming cost calculator

---

## Table of Contents
1. [Overview](#1-overview)
2. [Calculator Logic](#2-calculator-logic)
3. [CDN Providers — Active](#3-cdn-providers--active)
   - 3.1 AWS CloudFront
   - 3.2 Wangsu
   - 3.3 EdgeNext
   - 3.4 Akamai
4. [CDN Providers — Pending](#4-cdn-providers--pending)
5. [Provider Comparison](#5-provider-comparison)
6. [Traffic Distribution Model](#6-traffic-distribution-model)
7. [Reference Data](#7-reference-data)

---

## 1. Overview

The CDN layer sits between MediaPackage (origin) and end viewers. Unlike the Media Elements layer which is always AWS, the CDN layer operates across multiple providers simultaneously. Traffic is distributed across providers based on routing decisions, geographic performance, and cost optimization.

The CDN cost model is maintained as a separate document from the Media Elements cost model. Both documents feed into the same calculator. This document covers CDN-specific logic, rates, and provider details only.

**Architecture position:**
```
MediaPackage (origin)
        │
        ▼
CDN Layer (multiple providers)
        │
        ├──▶ AWS CloudFront  (ME / EU selectable)
        ├──▶ Wangsu
        ├──▶ EdgeNext
        └──▶ Akamai
        │
        ▼
End Viewers — MEA + North Africa
```

---

## 2. Calculator Logic

### 2.1 Two calculation modes

The CDN section of the calculator supports two modes. The user selects which mode applies to their use case.

---

**MODE A — Total GB with percentage distribution**

Use when: forecasting a future match, or when per-provider traffic breakdown is not available.

The user enters:
- Total GB delivered (auto-calculated from the main calculator: viewers × bitrate × watch_time ÷ 8,192)
  — 8,192 is a fixed unit conversion: 1 GB = 8,192 Megabits
- Which CDN providers are active (toggle on/off per provider card)
- % of total traffic routed to each active provider — must sum to 100%

Calculator outputs:
- GB per provider = total GB × provider %
- Cost per provider = GB × provider rate
- Total CDN cost = sum of all provider costs
- Blended effective rate = total CDN cost ÷ total GB

```
Example — 930,204 GB across 3 providers:
  CloudFront (ME):  40% → 372,082 GB × $0.007   = $2,604.57
  EdgeNext:         35% → 325,571 GB × $0.00643  = $2,093.42
  Wangsu:           25% → 232,551 GB × $0.00557  = $1,295.31
  ──────────────────────────────────────────────────────────
  Total:           100% → 930,204 GB              $5,993.30
  Blended rate: $0.00644/GB
```

---

**MODE B — Per-provider GB input**

Use when: actual traffic reports are available from the operator or CDN provider dashboards after a match.

The user enters:
- GB directly for each active provider (independent — no sum required)

Calculator outputs:
- Cost per provider = entered GB × provider rate
- Total GB = sum of all entered values
- Total CDN cost = sum of all provider costs
- Blended effective rate = total CDN cost ÷ total GB

```
Example — Per-provider actuals:
  CloudFront (ME):  372,082 GB × $0.007   = $2,604.57
  EdgeNext:         325,571 GB × $0.00643 = $2,093.42
  Wangsu:           232,551 GB × $0.00557 = $1,295.31
  ──────────────────────────────────────────────────
  Total:            930,204 GB             $5,993.30
```

---

### 2.2 Provider selection — UI design

Each CDN provider is shown as a separate, independently selectable card. Design requirements:

- Each provider is its own card with: name, rate/GB, and an active/inactive toggle
- Only providers with a non-zero entry (% or GB) appear in the cost calculation and provider counts — an active toggle with no entry is ignored
- Mode selector appears at the top of the CDN section. UI names: **"% Distribution Mode"** (Mode A) and **"Per-Provider GB Mode"** (Mode B)
- **CloudFront is a single card with a single input** (% or GB). The ME/EU split is applied automatically from the Advanced Settings value (default 70% ME / 30% EU) and CloudFront's card shows its blended rate (e.g. $0.00610/GB at 70/30). The user never enters ME and EU separately
- The ME/EU split detail is shown in the results: the CDN Layer Breakdown row notes the split, and the CDN Layer service card itemises ME and EU GB and cost
- SCCC, Huawei Cloud and Tencent Cloud appear as normal provider cards at the estimated $0.00839/GB rate, marked "estimated", default off

### 2.3 Validation rules

**Mode A:**
- All active provider percentages must sum to exactly 100%
- Calculator shows a running total and remaining % as the user adjusts inputs
- Highlights in red and blocks calculation if total ≠ 100%

**Mode B:**
- No sum validation — each provider is entered independently
- At least one provider must have a GB value greater than zero

**Both modes:**
- At least one provider must be active

### 2.4 Default distribution

When no custom distribution is entered, the calculator uses these defaults:

| Provider | Default % |
|---|---|
| CloudFront | 40% (split internally: 70% ME / 30% EU) |
| EdgeNext | 35% |
| Wangsu | 25% |

These defaults reflect operational distribution and should be updated when traffic reports provide more precise per-match data.

The CloudFront ME/EU split (default 70% ME / 30% EU) is adjustable in the **Advanced section** of the calculator. Business users use the default. Technical users who know the actual routing split can override it there.

---

## 3. CDN Providers — Active

These providers have  flat rates and are ready for use in the calculator.

---

### 3.1 AWS CloudFront

CloudFront appears as a single card with a single input in the calculator. The ME/EU split is applied automatically from the Advanced Settings split (default 70% ME / 30% EU); the resulting ME and EU allocation is itemised in the results detail.

| Region | Rate/GB | Primary use |
|---|---|---|
| ME (Bahrain) | **$0.007** | Viewer delivery to MEA + North Africa |
| EU (Ireland) | **$0.004** | Origin egress + backup delivery |

Both rates are privately negotiated flat rates — not public AWS tiered pricing.

**Default split:** 70% ME / 30% EU — adjust per match based on routing.

**Advanced split option:** The ME % is entered in the calculator's global Advanced Settings section (collapsed by default); EU % is calculated automatically as the remainder. Business users keep the 70/30 default.

---

### 3.2 Wangsu

| Field | Value |
|---|---|
| Provider | Wangsu (网宿科技) |
| Rate/GB | **$0.00557** |
| Rate type | Flat — contracted |
| Rate type |
| Role | Third-party CDN delivery (role to be  with operator) |
| Status | Active |

**Notes:**
- Lowest rate among  CDN providers
- Rate is consistent across varying traffic volumes —  flat contract

---

### 3.3 EdgeNext

| Field | Value |
|---|---|
| Provider | EdgeNext |
| Rate/GB | **$0.00643** |
| Rate type | Flat — contracted |
| Rate type |
| Role | Third-party CDN delivery (role to be  with operator) |
| Status | Active |

**Notes:**
- Rate is consistent across significantly varying traffic volumes —  flat contract
- Second-highest volume among CDN providers

---

### 3.4 Akamai

| Field | Value |
|---|---|
| Provider | Akamai |
| Rate/GB | **$0.00839** |
| Rate type | Flat — contracted (minor volume variation observed, working rate used) |
| Rate type |
| Role | Third-party CDN delivery (role to be  with operator) |
| Status | Active |

**Notes:**
- Rate used is the higher-volume period rate — conservative and representative of match-heavy periods
- Highest rate among  CDN providers
-  rate card Estimated from operator

---

## 4. CDN Providers — Pending

These providers appear in billing data without a derivable per-GB rate. They are ENABLED in the calculator at an estimated $0.00839/GB (Akamai-level, conservative), marked "estimated" and default off. The estimate will be replaced once billing-derived rates are received.

| Provider | Blocking Reason |
|---|---|
| **SCCC** | CDN traffic volume is  (~1.5–2M GB/month) but cost is bundled into infrastructure — no CDN-specific rate derivable |
| **Huawei** | CDN cost is not separated from full cloud infrastructure; billing unit inconsistency prevents GB calculation |
| **Tencent** | CSS Live Overseas shows 1.2–2.6M GB of streaming delivery but cost is allocated entirely to object storage — no CDN rate derivable |

Each estimated rate will be replaced with the billing-derived rate once received from the operator.

---

## 5. Provider Comparison

| Rank | Provider | Rate/GB |
|---|---|---|
| 1 | CloudFront EU | $0.00400 |
| 2 | Wangsu | $0.00557 |
| 3 | EdgeNext | $0.00643 |
| 4 | CloudFront ME | $0.00700 |
| 5 | Akamai | $0.00839 |

### 5.2 Cost at common traffic volumes

| Volume | CloudFront ME | CloudFront EU | Wangsu | EdgeNext | Akamai |
|---|---|---|---|---|---|
| 100 GB | $0.70 | $0.40 | $0.56 | $0.64 | $0.84 |
| 1 TB | $7.17 | $4.10 | $5.71 | $6.59 | $8.60 |
| 100 TB | $716.80 | $409.60 | $570.57 | $658.56 | $859.97 |
| 500 TB | $3,584 | $2,048 | $2,853 | $3,293 | $4,300 |
| 1 PB | $7,168 | $4,096 | $5,705 | $6,585 | $8,599 |
| 2.5 PB | $17,920 | $10,240 | $14,263 | $16,463 | $21,498 |

### 5.3 Blended rate scenarios

| Scenario | Distribution | Blended rate |
|---|---|---|
| Default | CF 40% (70/30 ME/EU), EdgeNext 35%, Wangsu 25% | $0.00628/GB |
| Cost-optimized | Wangsu 50%, EdgeNext 50% | $0.00600/GB |
| Full mix | CF 40%, EdgeNext 30%, Wangsu 20%, Akamai 10% | $0.00647/GB |
| CloudFront heavy | CF 70% (all ME), EdgeNext 20%, Wangsu 10% | $0.00665/GB |

---

## 6. Traffic Distribution Model

### 6.1 Mode A — step by step

1. Total GB is auto-filled from the main calculator output (viewers × bitrate × watch_time ÷ 8,192)
  — 8,192 is a fixed unit conversion: 1 GB = 8,192 Megabits
2. Toggle on each active CDN provider
3. For CloudFront: enter the % split between ME and EU within the CloudFront card
4. Enter % for each active provider — running total shown, must reach 100%
5. Calculator outputs: cost per provider, total CDN cost, blended effective rate

### 6.2 Mode B — step by step

1. Toggle on each active CDN provider
2. Enter actual GB per provider from operator traffic report
3. Calculator outputs: cost per provider, total GB, total CDN cost, blended effective rate

### 6.3 When to use each mode

| Situation | Mode |
|---|---|
| Forecasting a future match | Mode A |
| Post-match analysis with operator traffic report | Mode B |
| Post-match analysis without traffic report | Mode A with default distribution |
| CDN cost scenario comparison | Mode A — adjust percentages |

---

## 7. Reference Data

### 7.1  provider rates

| Provider | Rate/GB | Rate type |
|---|---|---|
| CloudFront ME | $0.007 |
| CloudFront EU | $0.004 |
| Wangsu | $0.00557 |
| EdgeNext | $0.00643 |
| Akamai | $0.00839 |
| SCCC | $0.00839 | Estimated |
| Huawei | $0.00839 | Estimated |
| Tencent | $0.00839 | Estimated |

### 7.2  traffic volumes

| Provider | Avg GB/month | Avg Cost/month |
|---|---|---|
| CloudFront ME | 2,567,310 GB | $17,971 |
| CloudFront EU | 1,620,439 GB | $6,482 |
| Wangsu | 777,179 GB | $4,330 |
| EdgeNext | 3,839,666 GB | $24,684 |
| Akamai | 905,300 GB | $7,752 |
| **Total** | **9,709,893 GB** | **$61,218** |

### 7.3 Effective blended rate

| Total avg GB/month | Total avg cost | Blended rate |
|---|---|---|
| 9,709,893 GB | $61,218 | **$0.00630/GB** |

---

## 8. Integration with Main Calculator

This document is maintained separately from `thmanyah_streaming_cost_model_v3.md`. Both are loaded into the same calculator interface. The integration points are:

**From main calculator → CDN model:**
- Total GB (calculated from viewers, bitrate, watch time) feeds directly into the CDN section
- Stream duration used to derive total GB
- Match type (Standard / 4K) affects total GB via quality tier bitrates

**From CDN model → main calculator:**
- Total CDN cost is the CloudFront / CDN line in the full pipeline cost summary
- Blended effective rate shown as a reference metric

**Calculator display:**
- Media Elements section (MediaConnect, MediaLive, MediaPackage) — from main document
- CDN section — from this document
- Both sections displayed within the same calculator page
- CDN section has its own Mode A / Mode B toggle independent of the main calculator

---

*Version 2.0 — Updated July 8, 2026*
*Active providers: CloudFront (ME/EU), Wangsu, EdgeNext, Akamai*
*Pending providers: SCCC, Huawei, Tencent*

---

## Developer Note — Merging into index.html

Both `.md` files (`thmanyah_streaming_cost_model_v3.md` and `thmanyah_cdn_cost_model.md`) are loaded as separate sections within the same `index.html`. Neither file is deleted or replaced — they remain as standalone `.md` references.

To merge into the existing `index.html` in VSCode, run the following command in the terminal:

```bash
# From the project root where index.html exists
node -e "
const fs = require('fs');
const { marked } = require('marked');

const main = fs.readFileSync('thmanyah_streaming_cost_model_v3.md', 'utf8');
const cdn  = fs.readFileSync('thmanyah_cdn_cost_model.md', 'utf8');

const mainHtml = marked.parse(main);
const cdnHtml  = marked.parse(cdn);

let html = fs.readFileSync('index.html', 'utf8');

html = html.replace(
  '<!-- INJECT:MAIN_MODEL -->',
  mainHtml
);
html = html.replace(
  '<!-- INJECT:CDN_MODEL -->',
  cdnHtml
);

fs.writeFileSync('index.html', html);
console.log('Done — both models injected into index.html');
"
```

**Prerequisites:** `npm install marked` if not already installed.

**Required placeholders in index.html:**
- Add `<!-- INJECT:MAIN_MODEL -->` where the Media Elements cost model section should appear
- Add `<!-- INJECT:CDN_MODEL -->` where the CDN cost model section should appear

Both sections render as independent tabs or panels within the same page.


---

## 9. Calculator Implementation Notes (v2.1 — July 10, 2026)

Decisions applied in `index_v4.html` that refine this document:

1. **CloudFront single input.** One card, one % / GB entry. ME/EU split (default 70/30) comes from Advanced Settings and is applied automatically; the split is itemised in the results detail card and noted on the breakdown row.
2. **Mode names.** Mode A = "% Distribution Mode", Mode B = "Per-Provider GB Mode" ("Enter actual GB per CDN vendor").
3. **Provider counting.** A provider counts toward the distribution (and appears in results) only when it has a non-zero entry. Toggled-on with no entry = ignored. The breakdown header shows "N providers in distribution" with no mode label.
4. **SCCC / Huawei / Tencent** are active calculator providers at estimated $0.00839/GB, marked "estimated", default off. No "Pending" labels anywhere.
5. **4K upscale delivery** is costed at the blended CDN rate from the user's entered distribution (falls back to the CloudFront blended rate when no distribution has been calculated).
6. **Input conventions.** All GB / viewer fields use thousands separators (1,000,000) everywhere; number-input spinners are removed globally; per-provider % / GB inputs update costs in place (no re-render, so multi-digit typing works).
