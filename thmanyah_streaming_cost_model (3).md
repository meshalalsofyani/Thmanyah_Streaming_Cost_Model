# Thmanyah Streaming Cost Model
**Document Date:** June 21, 2026  
**Scope:** AWS CloudFront CDN · MediaConnect · MediaLive · MediaPackage  
**Purpose:** Per-match cost forecasting and monthly infrastructure cost modeling

---

## Table of Contents
1. [Architecture Overview](#1-architecture-overview)
2. [Cost Model Principles](#2-cost-model-principles)
3. [CloudFront CDN](#3-cloudfront-cdn)
4. [MediaConnect](#4-mediaconnect)
5. [MediaLive](#5-medialive)
6. [MediaPackage](#6-mediapackage)
7. [Per-Match Cost Formula](#7-per-match-cost-formula)
8. [Worked Example — May 12 Al Hilal vs Al Nassr](#8-worked-example--may-12-al-hilal-vs-al-nassr)
9. [Monthly Cost Summary](#9-monthly-cost-summary)
10. [Calculator Inputs Reference](#10-calculator-inputs-reference)

---

## 1. Architecture Overview

The full live streaming pipeline flows as follows:

```
MCR (Ireland)
     │
     ▼
AWS MediaConnect (Frankfurt)
     │  Signal transport layer
     │  Carries raw video from MCR to encoding
     ▼
AWS MediaLive (Frankfurt)
     │  Encoding and transcoding layer
     │  Produces ABR ladder renditions (FTA, SPL/HDR, 4K)
     ▼
AWS MediaPackage (Frankfurt)
     │  Packaging and origin layer
     │  Packages HLS/DASH, serves as CDN origin
     ▼
AWS CloudFront CDN
     │  Content delivery layer
     │  Two active regions:
     ├── EU (Ireland)   → primarily origin egress + ops traffic
     └── ME (Bahrain)   → primary viewer delivery to MEA + North Africa
     ▼
End Viewers (MEA + North Africa only)
```

**Key architectural notes:**
- All encoding infrastructure is in EU Frankfurt
- Viewers are exclusively in MEA and North Africa — no other regions are served
- EU CloudFront handles origin egress and was used as a surrogate delivery region during the AWS ME incident in May 2026
- ME CloudFront is the primary viewer-facing delivery region
- At high volumes, EU delivery costs ~50% less per GB than ME delivery ($0.004 vs $0.007/GB)

---

## 2. Cost Model Principles

### 2.1 Billing basis
Every AWS service in this pipeline bills independently. A single match incurs cost at every layer simultaneously. Total match cost = sum of all four layers.

### 2.2 Discount
All services are covered under an **AWS Enterprise Discount Program (EDP)** at approximately **20%** applied uniformly. All rates and costs in this document are **post-discount net figures** as they appear in the final AWS bill. Pre-discount public rates are not referenced.

### 2.3 Fixed vs variable costs
Some costs are fixed monthly infrastructure regardless of match activity. Others are purely variable per match. The model separates these clearly.

| Layer | Cost type |
|---|---|
| MediaConnect baseline flows | Fixed monthly |
| MediaConnect match-day flows | Variable per match day |
| MediaLive | Variable per channel-hour |
| MediaPackage | Variable per GB |
| CloudFront | Variable per GB |

### 2.4 Regional cost asymmetry
ME (Bahrain) delivery costs **1.75x more per GB** than EU (Ireland) delivery at current contracted rates. This is a permanent AWS regional pricing difference, not a temporary condition. Any significant shift of delivery traffic from ME to EU reduces CDN cost meaningfully.

---

## 3. CloudFront CDN

### 3.1 Contracted flat rates
These are final contracted net rates. No tiered pricing applies — the rate is flat regardless of monthly volume.

| Region | Rate per GB | Notes |
|---|---|---|
| ME (Bahrain) | **$0.007/GB** | Primary viewer delivery region |
| EU (Ireland) | **$0.004/GB** | Origin egress + backup delivery |

### 3.2 Excluded line items
The following CloudFront cost components are excluded from the model as negligible (under $50/month combined):
- HTTP/HTTPS request charges (~2–3% of total, under $30/month)
- Data transfer out to origin (under $3/month per region)

### 3.3 Cost formula
```
CloudFront cost = (GB delivered via ME × $0.007) + (GB delivered via EU × $0.004)
```

### 3.4 Reference data — April and May 2026

| Month | ME GB | ME cost | EU GB | EU cost | Total |
|---|---|---|---|---|---|
| April 2026 | 3,079,836 GB | $21,558 | 727,230 GB | $2,909 | **$24,467** |
| May 2026 | 2,054,784 GB | $14,383 | 2,513,647 GB | $10,055 | **$24,438** |

**Note on May 2026:** EU delivery volume spiked to 2.51 PB due to an AWS ME region incident that forced routing of MEA viewer traffic through EU CloudFront as a fallback. Under normal operations, ME carries the majority of delivery traffic.

### 3.5 Regional delivery cost comparison

| Scenario | GB | ME cost | EU cost | Saving if routed via EU |
|---|---|---|---|---|
| 1 TB delivered | 1,024 GB | $7.17 | $4.10 | $3.07 (43%) |
| 1 PB delivered | 1,048,576 GB | $7,340 | $4,194 | $3,146 (43%) |

**Important caveat:** Routing all delivery through EU instead of ME reduces cost but increases latency for MEA viewers and creates a single point of failure. This trade-off must be validated with AWS before any architecture change.

### 3.6 Viewer-hour cost anchor *(estimate)*
Derived from May 12 Al Hilal vs Al Nassr match data.

- Actual GB per viewer per match: **2.46 GB** (based on 930,204 GB ÷ 378,210 viewers)
- Implied average bitrate: **~3 Mbps** (vs 8 Mbps peak — ABR stepping down accounts for the difference)

| Delivery scenario | Rate/GB | Cost per viewing hour *(estimate)* |
|---|---|---|
| ME only | $0.007 | **$0.01722/hr** |
| EU only | $0.004 | **$0.00984/hr** |
| Blended 70% ME / 30% EU | $0.0061 | **$0.01500/hr** |

```
Viewing hour cost = GB per viewer per hour × regional rate
GB per viewer per hour = actual bitrate (Mbps) × 3,600 ÷ 8,192
At 3 Mbps actual: 3 × 3,600 ÷ 8,192 = ~1.318 GB/viewer/hour
```

---

## 4. MediaConnect

### 4.1 What it is
MediaConnect is the signal transport layer. It carries the raw unencoded video signal from the MCR in Ireland to MediaLive in Frankfurt. It does not encode or process content — it is a managed secure pipe.

**Signal path:**
```
MCR (EU Ireland) ──[free ingest]──▶ MediaConnect (EU Frankfurt) ──[$0.008/GB]──▶ MediaLive
```

### 4.2 Billing components

| Component | Unit | Net rate | Notes |
|---|---|---|---|
| Standard RunningFlow | per flow/hour | **$0.1934/hr** | Billed whether content is flowing or not |
| LargeInstance RunningFlow | per flow/hour | **$1.368/hr** | Required for UHD/4K high-bitrate feeds |
| Inbound data (Ireland → Frankfurt) | per GB | **$0.00/GB** | Free — intra-AWS EU transfer |
| Regional transfer (Frankfurt internal) | per GB | **$0.008/GB** | MediaConnect → MediaLive hop within Frankfurt |

**Cost per flow per day:**
- Standard flow: $0.1934 × 24 = **$4.64/day**
- LargeInstance flow: $1.368 × 24 = **$32.83/day**

### 4.3 LargeInstance and 4K matches
When a match includes 4K delivery, MediaLive runs UHD HEVC channels (CH10_SPL) which require high-bitrate signal feeds. These feeds require LargeInstance flows in MediaConnect. Therefore:

> **If match type = 4K → include LargeInstance flow cost in MediaConnect**

Exact number of LargeInstance flows per 4K match is a working assumption of **1–2 flows** pending confirmation.

| 4K match duration | 1 LargeInstance flow | 2 LargeInstance flows |
|---|---|---|
| 4 hours | $5.47 | $10.94 |
| 6 hours | $8.21 | $16.42 |
| 8 hours | $10.94 | $21.88 |

### 4.4 Fixed baseline vs variable cost
Based on daily cost analysis across April and May 2026:

| Cost component | Value | Basis |
|---|---|---|
| Fixed baseline (always-on flows) | **$3,257/month** | ~19 flows running 24/7 at end of May — working assumption |
| Variable match-day increment | **$112/match day** | Average cost above baseline on active match days |

**Flow count assumptions *(working assumptions — not independently confirmed):***

| Scenario | Assumed concurrent flows | Daily cost | Monthly cost |
|---|---|---|---|
| Minimum observed (May 10 — AWS incident) | ~9 flows | $53/day | ~$1,620 |
| **Baseline (end of May — low season)** | **~19 flows** | **$108/day** | **~$3,257** |
| Average across April–May active periods | ~28 flows | $163/day | ~$4,890 |
| Peak observed (Apr 11 — heavy match load) | ~56 flows | $322/day | ~$9,794 |

**How flow count was derived:** Daily cost from AWS Cost Explorer ÷ 24 hours ÷ $0.1934/hr (standard flow rate) = implied concurrent flows. This is an approximation — days with LargeInstance flows will show a higher implied count since LargeInstance costs more per hour. Actual flow count must be confirmed via AWS Console → MediaConnect → Flows.

**Working assumption note:** The baseline of 19 flows and the $112/match-day increment are derived from billing patterns across April–May 2026. Exact flow counts have not been independently confirmed in the AWS Console. These figures should be validated and updated when flow inventory is confirmed.

**Monthly MediaConnect forecast formula:**
```
Monthly MediaConnect cost = $3,257 + ($112 × number of match days in month)
```

**Example:** Month with 15 match days = $3,257 + ($112 × 15) = **$4,937**

### 4.5 Reference data — April and May 2026

| Month | RunningFlow hrs | LargeInstance hrs | Regional GB | Gross | EDP discount | Net |
|---|---|---|---|---|---|---|
| April 2026 | 21,435 | 75.7 | 201,521 GB | $7,327 | -$1,465 | **$5,862** |
| May 2026 | 12,668 | 413.0 | 146,282 GB | $5,232 | -$1,046 | **$4,186** |

**Note:** April had significantly more RunningFlow hours (~29 concurrent flows avg) vs May (~17 avg), consistent with April having more match days. May had more LargeInstance hours (413 vs 76) consistent with Finals at higher resolutions.

---

## 5. MediaLive

### 5.1 What it is
MediaLive is the encoding and transcoding layer. It takes the raw signal from MediaConnect and encodes it into multiple ABR renditions for delivery. It bills per channel-hour based on codec, resolution, and pipeline type — for both input and output independently.

### 5.2 Channel inventory

**Active production channels: 14 channels (CH01–CH11)**

| Channel group | Channels | Type | Status |
|---|---|---|---|
| CH01–CH09 FTA | 9 channels | FTA SDR H.264 | Active production |
| CH01–CH09 SPL | 9 channels | Premium HDR10 H.265 | Active production |
| CH10_SPL | 1 channel | 4K UHD H.265 | Active — 4K matches only |
| CH11_SPL | 1 channel | 4K UHD H.264 | Active — used as part of 4K configuration |
| CH12–CH15, CH20 | 5 channels | Various | Testing channels — excluded from cost model |

**Operating pattern:** Channels start approximately **1 hour before stream** and run for approximately **5–6 hours** from stream start. Default stream duration used in model: **6 hours**. Calculator allows custom duration input.

### 5.3 FTA channel configuration (CH01–CH09 FTA, standard SDR)

**Input:**
| Type | Rate/hr |
|---|---|
| HD AVC 20-50mbps Standard Pipeline | **$0.5340/hr** |

**Output — 3 renditions:**
| Rendition | Resolution | Codec | Bitrate | AWS billing type | Rate/hr |
|---|---|---|---|---|---|
| LOW | 432p | H.264 QVBR | 1.2 Mbps max | SD AVC <10mbps 30-60fps | **$0.6660/hr** |
| MEDIUM | 720p | H.264 QVBR | 3.5 Mbps max | SD AVC <10mbps 30-60fps | **$0.6660/hr** |
| HIGH | 1080p | H.264 QVBR | 8 Mbps max | HD AVC <10mbps 30-60fps | **$1.3320/hr** |

**Total FTA channel cost per hour:**
```
$0.5340 (input) + $0.6660 + $0.6660 + $1.3320 (outputs) = $3.198/hr per FTA channel
```

### 5.4 SPL/Premium channel configuration (CH01–CH09 SPL, HDR10)

**Input:**
| Type | Rate/hr |
|---|---|
| HD HEVC 20-50mbps Standard Pipeline | **$1.0620/hr** |

**Output — 2 video renditions:**
| Rendition | Resolution | Codec | Max Bitrate | AWS billing type | Rate/hr |
|---|---|---|---|---|---|
| MEDIUM | 720p | H.265 QVBR | 7 Mbps | HD HEVC 30-60fps | **$5.3280/hr** |
| HIGH | 1080p | H.265 QVBR | 15 Mbps | HD HEVC 30-60fps | **$5.3280/hr** |

**Output — 4 audio renditions (per channel):**
| Audio track | Language | Codec | Bitrate | Rate/hr |
|---|---|---|---|---|
| audio_arabic | Arabic (ara) | AAC 128kbps | 128kbps | **$0.2640/hr** |
| audio_english | English (eng) | AAC 128kbps | 128kbps | **$0.2640/hr** |
| audio_original | Original (ols) | AAC 128kbps | 128kbps | **$0.2640/hr** |
| audio_arabic2 | Arabic 2 (fra) | AAC 128kbps | 128kbps | **$0.2640/hr** |

**Total SPL channel cost per hour:**
```
$1.0620 (input) + $5.3280 + $5.3280 (video outputs) + $1.0560 (4×audio) = $12.774/hr per SPL channel
```

### 5.5 4K channel configuration (CH10_SPL + CH11_SPL)

**CH10_SPL — UHD H.265 (primary 4K channel)**

| Component | Resolution | Codec | Bitrate | AWS billing type | Rate/hr |
|---|---|---|---|---|---|
| Input | UHD | H.265 QVBR | 20-50mbps | UHD HEVC 20-50mbps Std | **$6.3900/hr** |
| Output LOW | 540p | H.265 QVBR | 5 Mbps | HD HEVC 30-60fps | **$5.3280/hr** |
| Output MED | 720p | H.265 QVBR | 3.5 Mbps | HD HEVC 30-60fps | **$5.3280/hr** |
| Output HIGH | 1080p | H.265 QVBR | 8 Mbps | HD HEVC 30-60fps | **$5.3280/hr** |
| Output MAX | 2160p (4K) | H.265 QVBR | 12 Mbps max | UHD HEVC 30-60fps | **$21.312/hr** |

**CH10_SPL total per hour:** $6.390 + $5.328 + $5.328 + $5.328 + $21.312 = **$43.686/hr**

**CH11_SPL — UHD H.264 (secondary 4K channel)**

| Component | Resolution | Codec | Bitrate | AWS billing type | Rate/hr |
|---|---|---|---|---|---|
| Input | UHD | H.264 QVBR | 20-50mbps | UHD AVC 20-50mbps Std | **$3.1920/hr** |
| Output LOW | 540p | H.264 QVBR | 5 Mbps | SD AVC 30-60fps | **$0.6660/hr** |
| Output MED | 720p | H.264 QVBR | 3.5 Mbps | SD AVC 30-60fps | **$0.6660/hr** |
| Output HIGH | 1080p | H.264 QVBR | 8 Mbps | HD AVC 30-60fps | **$1.3320/hr** |
| Output MAX | 1440p | H.264 QVBR | 10 Mbps | HD AVC 30-60fps | **$1.3320/hr** |

**CH11_SPL total per hour:** $3.192 + $0.666 + $0.666 + $1.332 + $1.332 = **$7.188/hr**

### 5.6 How MediaLive actually bills — critical clarification

**MediaLive channels run 24/7 continuously.** They do not start and stop per match. The cost is always-on infrastructure billed by the hour regardless of whether a match is happening or not.

This means:
- The hourly rate card (sections 5.3–5.5) is correct for understanding what each channel costs per hour
- But per-match cost cannot be calculated as `rate × match hours` — channels are running all day regardless
- The correct per-match cost is derived from: **monthly bill ÷ number of matches**

**Daily baseline (no-match days):**
From Cost Explorer daily data, on days with no matches (Apr 16–18):
- Daily cost: **$569.86/day**
- Monthly fixed baseline: **~$17,096/month**
- This represents always-on channels with no active match encoding

**Match-day increment:**
On days with matches, cost rises above the $569.86 baseline:
- Average single-match day increment above baseline: **~$1,346/match day**
- May 12 (2 matches): $2,095.48 − $569.86 = $1,525.62 above baseline → **$762.81 per match increment**

### 5.7 Per-match cost — correct model

**Method: monthly total ÷ match count (recommended for forecasting)**

| Month | Net total | Matches | Cost per match |
|---|---|---|---|
| April 2026 | $72,585 | 95 | **$764/match** |
| May 2026 | $58,402 | 83 | **$703/match** |
| **Average** | | | **$733/match** |

**Recommended default for calculator: $733/match**

This is the most accurate per-match cost because it reflects the real billing pattern — 24/7 always-on infrastructure amortized across the number of matches in the month.

**The hourly rate card is still useful for:**
- Understanding which channel types drive cost
- Estimating the incremental cost of adding a new channel type
- Comparing cost impact of 4K vs standard matches

**4K match incremental cost above standard:**
When a 4K match runs, CH10_SPL and CH11_SPL add hours on top of the standard channel set:
```
4K incremental rate = CH10_SPL + CH11_SPL = $43.686 + $7.188 = $50.874/hr
4K incremental cost per match (6hrs) = $50.874 × 6 = $305.24
```

| 4K stream duration | Incremental cost above standard match |
|---|---|
| 4 hours | **$203.50** |
| 6 hours | **$305.24** |
| 8 hours | **$406.99** |

### 5.8 Daily cost pattern — April and May 2026 *(from Cost Explorer)*

| Scenario | Daily cost | Notes |
|---|---|---|
| No-match baseline (Apr 16–18) | **$569.86/day** | Always-on channels, no active matches |
| End of May flat (May 26–30) | **$2,059.89/day** | Consistent match/channel pattern |
| Single match day avg (May) | **~$1,878/day** | May monthly avg |
| Peak day (Apr 4) | **$4,663/day** | Multiple concurrent high-traffic matches |
| May 12 (2 matches) | **$2,095.48/day** | Confirmed 2-match day |

### 5.9 Reference data — April and May 2026

| Month | Input cost | Output cost | Gross | EDP discount | Net | Matches | Per match |
|---|---|---|---|---|---|---|---|
| April 2026 | $9,911 | $80,656 | $90,731 | -$18,146 | **$72,585** | 95 | **$764** |
| May 2026 | $9,402 | $63,272 | $73,003 | -$14,600 | **$58,402** | 83 | **$703** |

**Key insight:** Output cost is ~88% of total MediaLive cost. HD HEVC 30-60fps (SPL premium output) is the single largest cost driver — $34,546 in April and $23,746 in May, representing 37–43% of the entire MediaLive bill.

---

## 6. MediaPackage

### 6.1 What it is
MediaPackage is the packaging and origin layer. It receives encoded streams from MediaLive (ingest), packages them into HLS/DASH manifests, and serves them to CloudFront on request (origination). Currently used for basic packaging only — no DVR, SSAI, or DRM at this layer.

### 6.2 Billing components

| Component | Direction | Public rate | Net rate (post-20% EDP) | Notes |
|---|---|---|---|---|
| Ingest bytes | MediaLive → MediaPackage | $0.045/GB | **$0.036/GB** | Always present when channels are running |
| Origination bytes | MediaPackage → CloudFront | $0.060/GB | **$0.048/GB** | Driven by CloudFront viewer requests |

**Combined cost per GB passing through MediaPackage:**
```
$0.036 (ingest) + $0.048 (origination) = $0.084/GB combined
```

### 6.3 Ingest vs origination relationship
Ingest volume = how much MediaLive encodes → driven by channel-hours and rendition count. Always present when channels are running, regardless of viewer count.

Origination volume = how much CloudFront pulls from MediaPackage → driven by CloudFront edge cache warming. Much smaller than CDN delivery because CloudFront caches efficiently and serves thousands of viewers from a single origin pull.

**The scale difference — May 12 example:**
```
MediaLive encodes → ~2,876 GB passes through MediaPackage (ingest + origination)
                                    ↓
                    CloudFront edge caches pull ~820 GB from MediaPackage (origination)
                                    ↓
                    CloudFront delivers 930,204 GB to all viewers
```
CloudFront multiplies the content ~324x through caching. MediaPackage origination is only ~0.9% of total CDN delivery volume.

### 6.4 How MediaPackage bills — critical clarification for the calculator

**MediaPackage bills daily — not per match.** A single daily bill covers all matches that ran on that day. There is no per-match isolation in the AWS bill.

This means the calculator must handle MediaPackage in two distinct modes:

---

**MODE 1 — Historical match (daily cost is known)**

Use when: you have the actual Cost Explorer daily figure for that day.

```
Formula:
MediaPackage cost for match = daily total × viewer share %

Example — May 12 Al Hilal vs Al Nassr:
Daily confirmed: $483.18
Viewer share: 95% (second match had ~5% of viewers)
Match cost: $483.18 × 0.95 = $459.02

IMPORTANT: Do NOT divide by match count first and then apply percentage.
Apply the viewer share % directly to the daily total.
Correct:   $483.18 × 0.95 = $459.02 ✓
Incorrect: ($483.18 ÷ 2) × 0.95 = $229.41 ✗ — mixes two methods
```

**Required inputs for this mode:**
- Daily MediaPackage cost (from Cost Explorer)
- Number of matches that day
- Viewer share % for this match (if only one match: 100%)

---

**MODE 2 — Future match forecast (daily cost unknown)**

Use when: forecasting cost for an upcoming match.

```
Formula:
MediaPackage cost = (ingest GB × $0.036) + (origination GB × $0.048)

Default values (derived from April–May 2026 daily data):
Ingest: 1,230 GB × $0.036 = $44.28
Origination: 820 GB × $0.048 = $39.36
Default total: $83.64 per match

Note: This is an average across all matches including low-viewership ones.
High-profile matches (Finals, Al Hilal vs Al Nassr tier) will cost significantly more.
May 12 Al Hilal vs Al Nassr actual: $459.02 — 5.5x the average default.
Adjust GB inputs upward for high-viewership matches.
```

**Required inputs for this mode:**
- Ingest GB (default: 1,230 GB, adjustable)
- Origination GB (default: 820 GB, adjustable)

---

### 6.5 Per-match GB reference table *(derived from daily Cost Explorer data)*

| Data source | Per match cost | Per match GB | Notes |
|---|---|---|---|
| April single-match days (12 days avg) | $107 | ~1,278 GB | Lower — more concurrent matches in April |
| May single-match days (13 days avg) | $179 | ~2,130 GB | Higher — Finals with more viewers |
| May 12 Al Hilal vs Al Nassr (confirmed) | $459.02 | ~5,465 GB | High-viewership confirmed figure |
| **Default forecast average** | **$83.64** | **~2,051 GB** | **Use for average match forecasting** |

**Per-match GB split (60% ingest / 40% origination):**

| Component | Default GB | Rate | Default cost |
|---|---|---|---|
| Ingest | 1,230 GB | $0.036 | **$44.28** |
| Origination | 820 GB | $0.048 | **$39.36** |
| **Total per match** | **2,051 GB** | | **$83.64** |

### 6.6 Cost formula summary

```
MODE 1 — Historical (daily cost known):
MediaPackage cost = daily cost × viewer share %
DO NOT divide by match count first

MODE 2 — Forecast (daily cost unknown):
MediaPackage cost = (ingest GB × $0.036) + (origination GB × $0.048)
Default: (1,230 × $0.036) + (820 × $0.048) = $83.64
```

### 6.7 Reference data — April and May 2026

| Month | Ingest GB | Ingest cost | Origination GB | Origination cost | Gross | EDP discount | Net |
|---|---|---|---|---|---|---|---|
| April 2026 | 78,119 GB | $3,515 | 40,515 GB | $2,430 | $5,946 | -$1,189 | **$4,757** |
| May 2026 | 36,500 GB | $1,642 | 41,068 GB | $2,464 | $4,106 | -$821 | **$3,285** |

### 6.8 Daily cost reference *(from Cost Explorer)*

| Scenario | Daily cost | Notes |
|---|---|---|
| No-match baseline | ~$22/day | Infrastructure only, no matches |
| Single match day (avg) | ~$197/day | Average across April–May |
| Two-match day (May 12 confirmed) | **$483.18/day** | 2 matches, 95/5 viewer split |
| Peak day (Apr 4) | $1,618/day | Unknown cause — investigate |

**Monthly fixed MediaPackage baseline:** ~$22 × 30 = **~$660/month**

---

## 7. Per-Match Cost Formula

### 7.1 Complete pipeline cost per match

```
Total match cost = MediaConnect + MediaLive + MediaPackage + CloudFront
```

### 7.2 Input variables for the calculator

| Variable | Description | Default | Notes |
|---|---|---|---|
| `stream_hours` | Total stream duration | 6 hrs | Used for 4K incremental calculation only |
| `match_type` | Standard or 4K | Standard | 4K adds incremental channel cost |
| `medialive_cost_per_match` | MediaLive cost per match | $733 | Based on monthly avg ÷ match count |
| `is_4k` | Is this a 4K match | No | Adds CH10_SPL + CH11_SPL incremental |
| `large_instance_flows` | MediaConnect LargeInstance flows | 1–2 | Only if is_4k = Yes |
| `ingest_gb` | GB ingested into MediaPackage | 1,230 GB | Adjustable |
| `origination_gb` | GB originated to CloudFront | 820 GB | Adjustable |
| `delivery_gb_me` | GB delivered via ME CloudFront | Manual | Primary viewer traffic |
| `delivery_gb_eu` | GB delivered via EU CloudFront | Manual | Origin + backup traffic |
| `match_days_in_month` | For monthly MediaConnect baseline | — | Used for monthly forecast only |

### 7.3 Standard match — cost breakdown

| Layer | Formula | Default cost |
|---|---|---|
| MediaLive | monthly avg ÷ matches | **$733.00** |
| MediaConnect (variable) | $112 × 1 match day | **$112.00** |
| MediaPackage ingest | 1,230 GB × $0.036 | **$44.28** |
| MediaPackage origination | 820 GB × $0.048 | **$39.36** |
| CloudFront ME | delivery_gb_me × $0.007 | variable |
| CloudFront EU | delivery_gb_eu × $0.004 | variable |
| **Fixed components total** | | **$928.64** |
| **Variable (CloudFront)** | | GB-dependent |

### 7.4 4K match additions

| Addition | Formula | Cost |
|---|---|---|
| MediaLive 4K incremental (6hrs) | $50.874/hr × 6hrs | $305.24 |
| MediaLive 4K incremental (4hrs) | $50.874/hr × 4hrs | $203.50 |
| MediaLive 4K incremental (8hrs) | $50.874/hr × 8hrs | $406.99 |
| MediaConnect LargeInstance (1 flow, 6hrs) | $1.368/hr × 6hrs | $8.21 |
| MediaConnect LargeInstance (2 flows, 6hrs) | $1.368/hr × 6hrs × 2 | $16.42 |

**Total 4K match at 6 hours (standard + 4K incremental):**

| Component | Cost |
|---|---|
| Standard match fixed | $928.64 |
| 4K MediaLive incremental | $305.24 |
| 4K MediaConnect LargeInstance (1–2 flows) | $8.21–$16.42 |
| **Total 4K fixed** | **$1,242.09–$1,250.30** |
| CloudFront | GB-dependent |

### 7.5 Monthly fixed baseline
Regardless of match activity, the following cost exists every month:

| Component | Monthly fixed cost |
|---|---|
| MediaLive baseline (always-on channels) | **~$17,096/month** |
| MediaConnect baseline (~19 flows 24/7) | **$3,257/month** |
| MediaPackage baseline (infrastructure ingest) | **~$660/month** |
| CloudFront (minimal ops traffic) | **~$200–400/month** |
| **Total monthly fixed** | **~$21,013–21,413/month** |

---

## 8. Worked Example — May 12 Al Hilal vs Al Nassr

### 8.1 Match details
| Parameter | Value |
|---|---|
| Date | May 12, 2026 |
| Stream window | 7:00 PM – 1:00 AM (6 hours) |
| Match type | Standard (FTA + SPL) |
| Peak concurrent viewers | 308,120 |
| Total unique viewers | 378,210 |
| Avg watch time per device | 114.76 minutes |
| Total viewing hours | 723,896 hours |
| Avg peak bitrate | 8 Mbps |
| Actual implied bitrate | ~3 Mbps (ABR stepped down) |
| GB per viewer | 2.46 GB |

### 8.2 Confirmed daily costs — May 12 (both matches combined)
*Source: AWS Cost Explorer daily view*

| Service | Confirmed daily cost |
|---|---|
| CloudFront | **$4,593.43** |
| Elemental MediaLive | **$2,095.48** |
| Elemental MediaPackage | **$483.18** |
| Elemental MediaConnect | **$126.38** |
| **Total May 12** | **$7,298.48** |

**Note on the $5,622.81 figure:** This number was provided earlier as the Al Hilal vs Al Nassr CloudFront cost specifically. However the confirmed Cost Explorer daily total for CloudFront on May 12 is $4,593.43 covering both matches. The $5,622.81 source is unclear and has been retired — **$4,593.43 is the confirmed figure.**

**Note on GB delivered:** 930,204 GB was the confirmed delivery for Al Hilal vs Al Nassr specifically. The second match on May 12 had ~5% of that viewership, so total day GB ≈ 930,204 × 1.05 = ~976,714 GB.

### 8.3 Per-match split — 95% / 5% viewer ratio

The second match on May 12 had approximately 5% of Al Hilal vs Al Nassr viewers. Cost is allocated proportionally by applying the viewer share % directly to the daily total.

**Calculator instruction — MediaPackage for historical matches:**
```
CORRECT:   daily cost × viewer share % = $483.18 × 0.95 = $459.02 ✓
INCORRECT: (daily cost ÷ match count) × viewer share % = ($483.18 ÷ 2) × 0.95 = $229.41 ✗
INCORRECT: use section 6 default $83.64 for a known historical match ✗

Rule: for historical matches where daily cost is known,
always apply viewer share % directly to daily total.
Section 6 defaults ($83.64) are for future forecasting only.
```

| Service | Daily total | Al Hilal vs Al Nassr (95%) | Second match (5%) |
|---|---|---|---|
| CloudFront | $4,593.43 | **$4,363.76** | $229.67 |
| MediaLive | $2,095.48 | **$1,990.71** | $104.77 |
| MediaPackage | $483.18 | **$459.02** | $24.16 |
| MediaConnect | $126.38 | **$120.06** | $6.32 |
| **Total** | **$7,298.48** | **$6,933.55** | **$364.92** |

### 8.4 CloudFront GB and effective rate — Al Hilal vs Al Nassr

| Metric | Value |
|---|---|
| GB delivered (confirmed) | 930,204 GB |
| CloudFront cost allocated (95%) | $4,363.76 |
| Effective rate/GB | **$0.00469/GB** |

**Why $0.00469/GB vs contracted $0.007 ME / $0.004 EU:**
May 12 fell during the AWS ME region incident. A significant portion of traffic routed through EU (Ireland) at $0.004/GB instead of ME (Bahrain) at $0.007/GB. The blended effective rate of $0.00469 suggests approximately **55% EU / 45% ME** routing on that day.

### 8.5 Per-viewer cost breakdown — Al Hilal vs Al Nassr

| Metric | Value |
|---|---|
| Total unique viewers | 378,210 |
| Peak concurrent viewers | 308,120 |
| Avg watch time | 114.76 mins |
| Total viewing hours | 723,896 hrs |
| GB per viewer | 2.46 GB |
| Actual implied bitrate | ~3 Mbps (vs 8 Mbps peak) |
| **CloudFront cost per viewer** | **$0.01154** |
| **Full pipeline cost per viewer** | **$0.01833** |
| **CloudFront cost per viewing hour** | **$0.00603** |
| **Full pipeline cost per viewing hour** | **$0.00958** |

---

## 9. Monthly Cost Summary

### 9.1 April 2026 actuals

| Service | Net cost |
|---|---|
| CloudFront | $24,467 |
| MediaConnect | $5,862 |
| MediaLive | $72,585 |
| MediaPackage | $4,757 |
| **Total** | **$107,671** |

### 9.2 May 2026 actuals

| Service | Net cost |
|---|---|
| CloudFront | $24,438 |
| MediaConnect | $4,186 |
| MediaLive | $58,402 |
| MediaPackage | $3,285 |
| **Total** | **$90,311** |

### 9.3 Cost share by service

| Service | April share | May share |
|---|---|---|
| MediaLive | 67.4% | 64.7% |
| CloudFront | 22.7% | 27.1% |
| MediaPackage | 4.4% | 3.6% |
| MediaConnect | 5.4% | 4.6% |

**MediaLive consistently dominates at ~65–67% of total pipeline cost.** Any cost optimization effort should start here — specifically the HD HEVC 30-60fps SPL output renditions which represent 37–43% of the entire MediaLive bill.

---

## 10. Calculator Inputs Reference

### 10.1 Quick rate card

| Service | Component | Net rate |
|---|---|---|
| CloudFront | ME (Bahrain) delivery | $0.007/GB |
| CloudFront | EU (Ireland) delivery | $0.004/GB |
| MediaConnect | Standard flow | $0.1934/hr |
| MediaConnect | LargeInstance flow (4K) | $1.368/hr |
| MediaConnect | Regional transfer | $0.008/GB |
| MediaConnect | Fixed monthly baseline | $3,257/month *(working assumption)* |
| MediaConnect | Match-day variable | $112/match day *(working assumption)* |
| MediaLive | FTA channel | $3.198/hr (reference only) |
| MediaLive | SPL channel | $12.774/hr (reference only) |
| MediaLive | Standard match (FTA+SPL) | $15.972/hr (reference only) |
| MediaLive | CH10_SPL (4K primary) | $43.686/hr (incremental) |
| MediaLive | CH11_SPL (4K secondary) | $7.188/hr (incremental) |
| MediaLive | 4K incremental rate | $50.874/hr (incremental) |
| MediaLive | **Per-match default (24/7 amortized)** | **$733/match** |
| MediaLive | Monthly baseline (always-on) | ~$17,096/month |
| MediaPackage | Ingest | $0.036/GB |
| MediaPackage | Origination | $0.048/GB |
| MediaPackage | Combined | $0.084/GB |

### 10.2 Per-match reference costs (default values)

| Match type | MediaLive | MediaConnect | MediaPackage | CloudFront | Total fixed |
|---|---|---|---|---|---|
| Standard | $733.00 | $112.00 | $83.64 | GB-dependent | **$928.64 + CloudFront** |
| 4K (6hrs incremental) | $733 + $305.24 | $112 + $8.21–$16.42 | $83.64 | GB-dependent | **$1,242.09–$1,250.30 + CloudFront** |

**MediaPackage default per match:**
- Ingest: 1,230 GB × $0.036 = $44.28
- Origination: 820 GB × $0.048 = $39.36
- Total: **$83.64** *(manual override available)*

**MediaLive default per match: $733**
- Based on (April $72,585 ÷ 95) + (May $58,402 ÷ 83) ÷ 2 = $733
- Reflects 24/7 always-on channel cost amortized per match

### 10.3 Items pending validation

| Item | Status | Action needed |
|---|---|---|
| Exact MediaConnect flow count | Unknown | Check AWS Console → MediaConnect → Flows |
| LargeInstance flows per 4K match | Working assumption (1–2) | Confirm with ops team |
| MediaLive per-match cost | Derived from monthly avg ÷ matches | Refine as more months of data available |
| MediaPackage GB per match | Derived from daily Cost Explorer data | Refine with match-specific data |
| Alibaba CDN rates and structure | Pending | Obtain billing records |
| MediaConnect match-day increment ($112) | Working assumption | Validate when flow count is confirmed |
| Apr 4 spike ($1,618 MediaPackage, $4,663 MediaLive) | Unknown cause | Check match schedule for Apr 4 |

---

*Document prepared by Thmanyah Streaming Infrastructure — June 21, 2026*  
*All costs are post-EDP-discount net figures as billed by AWS*  
*Working assumptions are clearly marked and subject to revision*
