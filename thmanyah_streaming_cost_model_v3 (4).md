# Thmanyah Streaming Cost Model
**Document Date:** June 28, 2026
**Version:** 3.0
**Scope:** AWS Media Elements (MediaConnect · MediaLive · MediaPackage) · CDN Layer (CloudFront · Alibaba · Others)
**Purpose:** Per-match cost forecasting, monthly infrastructure cost modeling, and calculator logic reference

---

## Model Accuracy Statement

This model is built entirely from historical billing data (April–May 2026), confirmed contracted rates, and real match data from Conviva. Every number is derived from real AWS bills and real viewer data — not theoretical calculations.

**Stated accuracy per service:**

| Service | Accuracy | Confidence | Notes |
|---|---|---|---|
| CloudFront | ±5% | 95% | Contracted flat rate — most reliable component |
| MediaLive | ±50% | 60% | Channel configuration varies — no stable per-match formula |
| MediaConnect | ±25% | 70% | Close to actual on validated matches |
| MediaPackage (small <10K) | ±15% | 80% | From confirmed Apr 10 data |
| MediaPackage (medium 10K–150K) | ±35% | 65% | From monthly bill average |
| MediaPackage (large >150K) | ±5% | 95% | From confirmed May 12 data |
| **Full pipeline** | **±30%** | **70%** | Validated on May 12 at 4.7% off |

**Validated on May 12 Al Hilal vs Al Nassr:** model was 4.7% off confirmed actual of $6,933.55.

**Why MediaLive cannot be precisely modeled:**
MediaLive cost is driven by how many channels are active and for how long — not by viewer count or match count. This varied dramatically across April (single match day Apr 10 cost $4,625 while 3-match day Apr 29 cost only $1,598). Until the transition to full on-demand operation is complete and a clean season bill is available, the weighted average of $736/match is the best estimate.

**Model will be recalibrated:** at the start of each new season with updated billing data.

---

## Table of Contents
1. [Architecture Overview](#1-architecture-overview)
2. [Cost Model Principles](#2-cost-model-principles)
3. [How We Derived Bitrate Per Viewer](#3-how-we-derived-bitrate-per-viewer)
4. [Static Costs — Fixed Monthly Infrastructure](#4-static-costs--fixed-monthly-infrastructure)
5. [Variable Costs — Per Match](#5-variable-costs--per-match)
6. [Media Elements Layer — AWS Only](#6-media-elements-layer--aws-only)
7. [CDN Layer — Multiple Providers](#7-cdn-layer--multiple-providers)
8. [Channel Configuration and Viewer Quality Tiers](#8-channel-configuration-and-viewer-quality-tiers)
9. [Per-Match Cost Formula and Calculator Logic](#9-per-match-cost-formula-and-calculator-logic)
10. [Subscriber vs Non-Subscriber Cost](#10-subscriber-vs-non-subscriber-cost)
11. [4K Upscaling Analysis](#11-4k-upscaling-analysis)
12. [Demo Matches](#12-demo-matches)
13. [Monthly Reference Data](#13-monthly-reference-data)
14. [Calculator Inputs Reference and Rate Card](#14-calculator-inputs-reference-and-rate-card)
15. [Items Pending Validation](#15-items-pending-validation)

---

## 1. Architecture Overview

The pipeline has two distinct layers that are kept separate in the calculator and reporting:

```
MCR (Ireland — Master Control Room)
     │
     ▼
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  MEDIA ELEMENTS LAYER — AWS only, always
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
     │
     ▼
AWS MediaConnect (Frankfurt)
     │  Signal transport — carries raw unencoded video
     │  Ireland → Frankfurt: FREE ($0.00/GB)
     │  Frankfurt internal transfer: $0.008/GB net
     ▼
AWS MediaLive (Frankfurt)
     │  Encoding — produces ABR renditions per channel
     │  FTA (SDR H.264) + SPL/Premium (HDR H.265) + 4K (CH11 UHD H.264)
     │  ALL channels have 4 audio tracks (Arabic, English, Original, Arabic2)
     ▼
AWS MediaPackage (Frankfurt)
     │  Packaging and origin — produces HLS/DASH manifests
     │  Ingest from MediaLive: $0.036/GB net
     │  Origination to CDN: $0.048/GB net
     │
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  CDN LAYER — multiple providers
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
     │
     ├──▶ AWS CloudFront (active)
     │       ME (Bahrain):  $0.007/GB — primary viewer delivery
     │       EU (Ireland):  $0.004/GB — origin egress + backup
     │
     ├──▶ Alibaba CDN (active — rates pending)
     │
     └──▶ Other CDN providers (pending)
     │
     ▼
End Viewers — MEA + North Africa only
```

**Key architectural notes:**
- Media Elements are always AWS — no alternative provider for encoding
- CDN layer has multiple providers — CloudFront is currently primary
- MCR is in Ireland — all encoding infrastructure is in Frankfurt
- Viewers are MEA and North Africa only — no other regions served
- EU CloudFront handles origin egress and served as emergency fallback during AWS ME region incident (May 2026)
- ME CloudFront costs 1.75× more per GB than EU CloudFront at current contracted rates

---

## 2. Cost Model Principles

### 2.1 Two cost types

| Type | Description | Calculator treatment |
|---|---|---|
| **Fixed monthly** | Exists every month regardless of match activity | Read-only — shown for reference, not added to per-match calculation |
| **Variable per match** | Driven by match activity | Primary output of calculator |

### 2.2 EDP Discount
All AWS services carry an Enterprise Discount Program (EDP) discount of approximately **20%**. All rates in this document are **post-discount net figures** as they appear on the final AWS bill. Pre-discount rates are not referenced anywhere in this model.

### 2.3 Baseline derivation
Fixed monthly costs are derived from **May 28–31** — four consecutive days at end of season with no official matches and identical daily costs. This is the most reliable no-match baseline available in the season data.

Variable per-match costs are derived by subtracting the monthly fixed from each month's bill total, then dividing by match count — weighted across April (95 matches) and May (83 matches) = 178 total matches.

### 2.4 June data note
June had no official matches but included significant testing activity. June costs are used only as directional reference — not as a clean baseline. May 28–31 is used as the baseline instead.

### 2.5 MediaLive special note
MediaLive does not follow a predictable per-match cost pattern. The cost is driven by how many channels are active and for how long — which varied significantly across April even when controlling for match count. The $736/match weighted average is the best estimate available and carries a ±50% error range. This will improve once a full season of on-demand billing data is available.

---

## 3. How We Derived Bitrate Per Viewer

This is one of the most important variables in the model. Here is the full derivation, clearly documented.

### 3.1 The theoretical vs actual bitrate gap

FTA 1080p theoretical max bitrate from MediaLive config: **8 Mbps**

Viewers do not always stream at peak quality. ABR (Adaptive Bitrate) automatically adjusts quality based on connection speed. Most viewers land on 720p or lower renditions.

### 3.2 Derivation from real match data

**Step 1 — GB per viewer:**
```
GB per viewer = total GB delivered ÷ total unique viewers
```

**Step 2 — Actual bitrate:**
```
Actual Mbps = GB per viewer × 8,192 ÷ (watch_time_mins × 60)
```

### 3.3 Confirmed data points across 6 matches

| Match | Viewers | Watch time | Total GB | GB/viewer | Actual Mbps |
|---|---|---|---|---|---|
| Apr 7 (2 matches) | 1,980 total | 33.12 mins | 1,887.78 GB | 0.953 | **3.93** |
| Apr 10 FDL (1 match) | 2,650 | 35.93 mins | 2,844.14 GB | 1.073 | **4.08** |
| Apr 24 (2 matches) | 16,310 total | 60 mins | 25,838.66 GB | 1.584 | **3.60** |
| Apr 29 Al-Taawoun vs Al-Ittihad | 88,250 | 61 mins | proportional | 1.837 | **4.11** |
| Apr 29 Al-Nassr vs Al-Ahli | 374,900 | 97 mins | proportional | 1.837 | **2.59** |
| May 12 Al-Hilal vs Al-Nassr | 378,210 | 114.76 mins | 930,204 GB | 2.460 | **2.93** |

**Note on Apr 29 Al-Riyadh vs Al-Qadsiah (9,970 viewers, 19 mins):** excluded as an outlier — 19 minutes is too short to produce a reliable bitrate average (showed 13.2 Mbps which is physically implausible as a sustained average).

### 3.4 Pattern and auto-switching thresholds

The data shows a clear pattern: larger audiences have lower average bitrate. This is because large matches attract more viewers on weaker connections who get stepped down to lower renditions by ABR.

| Viewer range | Data points | Avg actual Mbps | Default used in calculator |
|---|---|---|---|
| < 10,000 | Apr 7 (3.93), Apr 10 (4.08) | 4.00 | **4.0 Mbps** |
| 10,000–150,000 | Apr 24 (3.60), Apr 29 Al-Taawoun (4.11) | 3.86 | **3.9 Mbps** |
| > 150,000 | Apr 29 Al-Nassr (2.59), May 12 (2.93) | 2.76 | **2.8 Mbps** |

**The calculator auto-selects the correct Mbps based on viewer count entered.** The user can always override manually.

### 3.5 How GB is calculated in the model

```
GB = viewers × bitrate_Mbps × (watch_time_mins × 60) ÷ 8,192

Where bitrate_Mbps auto-selects:
  viewers < 10,000  → 4.0 Mbps
  viewers 10K–150K  → 3.9 Mbps
  viewers > 150,000 → 2.8 Mbps

Example — Al-Hilal vs Al-Nassr (378,210 viewers, 114.76 mins):
  GB = 378,210 × 2.8 × (114.76 × 60) ÷ 8,192
     = 378,210 × 2.8 × 6,885.6 ÷ 8,192
     = 891,228 GB (vs actual 930,204 GB — 4.2% off)
```

**Important — 2.46 GB per viewer is NOT a constant:**
GB per viewer varies per match based on watch time and viewer size. The 2.46 GB figure from May 12 is specific to that match only (114.76 min avg watch time, 2.93 Mbps actual bitrate). It is a derived output from that specific match — not a model input or a default value. Do not use 2.46 as a fixed default anywhere in the calculator. Always calculate from bitrate × watch time.

---

## 4. Static Costs — Fixed Monthly Infrastructure

These costs exist every month regardless of match count. They are **read-only** in the calculator. The methodology for each number is shown for transparency.

**Baseline source:** May 28–31 daily average — confirmed no-match days, end of season.

**Why May 28–31:**
These 4 days showed identical costs to the cent across all three services — confirming no match activity. They represent the full channel set running at end of season with zero viewer activity. This is the cleanest no-match baseline available within the season. June was not used as baseline because it included testing activity that made it unreliable as a pure fixed-cost reference.

| Service | Daily baseline | Monthly fixed | How derived |
|---|---|---|---|
| MediaLive | $2,059.89/day | **$61,797/month** | May 28–31 identical cost — all channels always-on |
| MediaConnect | $108.53/day | **$3,256/month** | May 28–31 identical cost — ~19 flows active |
| MediaPackage | $23.01/day | **$690/month** | May 28–31 average — infrastructure ingest only |
| CloudFront ops | ~$200/month | **~$200/month** | Estimated from low-traffic days |
| **Total fixed** | | **~$65,943/month** | |

**MediaLive fixed cost note:**
The $61,797/month reflects channels running continuously at end of season. As transition to full on-demand operation completes (channels only start when a match begins and stop after), this fixed cost will drop significantly — toward only the idle channel fee of $0.01/hr per channel (~$216/month for 15 channels). The model will be updated once a full on-demand season is available.

**MediaConnect fixed cost note:**
~19 flows were Active (billed) during May end-of-season baseline. AWS only bills for Active flows — Standby flows are free. There are 60 total flows across 15 channels (4 flows per channel). The 19 active baseline flows represent flows kept running between matches during the season.

---

## 5. Variable Costs — Per Match

These costs scale with match activity. These are the primary outputs of the calculator.

**Derivation methodology:**
```
Variable monthly = total monthly bill − (baseline daily × days in month)
Variable per match = variable monthly ÷ number of matches that month
Weighted average across April (95) + May (83) = 178 total matches
```

**Validated against May 12 two-match day:**

| Service | Model (2 matches) | Actual daily | Difference |
|---|---|---|---|
| MediaLive (2 × $736) | $1,472 | $2,095 | 29.8% — within ±30% ✅ |
| MediaConnect (2 × $19.26) | $147 | $126 | 16.4% ✅ |
| MediaPackage (large tier) | $459 | $483 | 5.0% ✅ |

---

## 6. Media Elements Layer — AWS Only

### 6.1 MediaConnect

**What it is:** Signal transport. Carries raw unencoded video from MCR (Ireland) to MediaLive (Frankfurt). Does not encode — purely a managed pipe.

**Billing components:**

| Component | Gross rate | Net rate (post-20% EDP) | Notes |
|---|---|---|---|
| Active RunningFlow | $0.2418/hr | **$0.1934/hr** | Per flow — Standby is free |
| LargeInstance flow (4K) | $1.71/hr | **$1.368/hr** | Required for CH11 4K UHD signal |
| Inbound Ireland→Frankfurt | $0.00/GB | **Free** | Intra-AWS EU transfer |
| Regional transfer Frankfurt | $0.01/GB | **$0.008/GB** | MediaConnect → MediaLive internal hop |

**Flow structure — confirmed from AWS Console (60 total flows):**

Each channel has exactly 4 flows:
- Main-Flow_SPL
- FTA-Main-Flow_SPL
- Backup-Flow_SPL
- FTA-Backup-Flow_SPL

| Match type | Channels active | Flows | Flow type |
|---|---|---|---|
| Standard | FTA + SPL = 2 channels | **8 flows** | Standard RunningFlow |
| 4K | FTA + SPL + CH11 = 3 channels | **8 standard + 4 LargeInstance** | Mixed |

**Why 4K uses LargeInstance:** CH11_SPL carries UHD signal which requires higher bandwidth capacity. LargeInstance flows handle high-bitrate feeds.

**Per-match variable cost:** **$19.26/match** (weighted average from bills)
**Fixed monthly baseline:** $3,256/month (~19 flows Active between matches)

**Reference data:**

| Month | Active flow hrs | LargeInstance hrs | Regional GB | Gross | EDP | Net |
|---|---|---|---|---|---|---|
| April | 21,435 | 75.7 | 201,521 | $7,327 | -$1,465 | **$5,862** |
| May | 12,668 | 413.1 | 146,282 | $5,232 | -$1,046 | **$4,186** |
| June | 6,270 | 0 | 36,402 | $1,880 | -$376 | **$1,504** |

---

### 6.2 MediaLive

**What it is:** Encoding layer. Takes raw signal from MediaConnect and encodes into multiple ABR renditions. Bills per channel-hour for input and output independently. Cost is driven by channel configuration and run time — NOT by viewer count.

**Channel inventory — 29 total, 14 active production:**

| Group | Channels | Status |
|---|---|---|
| CH01–CH09 FTA | 9 channels | Active production |
| CH01–CH09 SPL | 9 channels | Active production |
| CH11_SPL | 1 channel | Active — 4K matches only |
| CH10_SPL | 1 channel | Reserved — excluded from cost model |
| CH12–CH15, CH20 | 5 channels | Testing — excluded from cost model |

**All channels have exactly 4 audio outputs:**
Arabic (ara), English (eng), Original (ols), Arabic 2 (fra) — all AAC 128kbps at $0.2640/hr each = **$1.056/hr per channel in audio**.

**FTA channel (H.264 SDR) — per hour breakdown:**

| Component | AWS billing type | Rate/hr |
|---|---|---|
| Input | HD AVC 20-50mbps Standard | $0.5340 |
| Output 432p | SD AVC <10mbps 30-60fps | $0.6660 |
| Output 720p | SD AVC <10mbps 30-60fps | $0.6660 |
| Output 1080p | HD AVC <10mbps 30-60fps | $1.3320 |
| Audio × 4 | Standard Pipeline Audio Only | $1.0560 |
| **FTA total** | | **$4.2540/hr** |

**SPL/Premium channel (H.265 HDR10) — per hour breakdown:**

| Component | AWS billing type | Rate/hr |
|---|---|---|
| Input | HD HEVC 20-50mbps Standard | $1.0620 |
| Output 720p | HD HEVC 30-60fps | $5.3280 |
| Output 1080p | HD HEVC 30-60fps | $5.3280 |
| Audio × 4 | Standard Pipeline Audio Only | $1.0560 |
| **SPL total** | | **$12.7740/hr** |

**Standard match rate:** $4.2540 + $12.7740 = **$17.028/hr**

**CH11_SPL — 4K channel (H.264 UHD) — per hour breakdown:**

| Component | AWS billing type | Rate/hr |
|---|---|---|
| Input | UHD AVC 20-50mbps Standard | $3.1920 |
| Output 540p | SD AVC 30-60fps | $0.6660 |
| Output 720p | SD AVC 30-60fps | $0.6660 |
| Output 1080p | HD AVC 30-60fps | $1.3320 |
| Output 1440p | HD AVC 30-60fps | $1.3320 |
| Audio × 4 | Standard Pipeline Audio Only | $1.0560 |
| **CH11 total** | | **$8.2440/hr** |

**4K match rate:** $17.028 + $8.244 = **$25.272/hr**
**4K add-on only (CH11):** **$8.244/hr**

**Per-match cost table (on-demand theoretical — for reference):**

| Match type | Rate/hr | 4hrs | 6hrs | 8hrs |
|---|---|---|---|---|
| Standard (FTA+SPL) | $17.028 | $68.11 | **$102.17** | $136.22 |
| 4K add-on (CH11 only) | $8.244 | $32.98 | **$49.46** | $65.95 |
| 4K total (FTA+SPL+CH11) | $25.272 | $101.09 | **$151.63** | $202.18 |

**Current historical per-match cost (from bills — includes always-on overhead):**

| Month | Net bill | Matches | Per match |
|---|---|---|---|
| April | $72,585 | 95 | $764 |
| May | $58,402 | 83 | $703 |
| **Weighted avg** | | **178** | **$736/match** |

**The $736/match is the calculator default.** It includes always-on channel overhead amortized across matches. As transition to on-demand completes, this will converge toward the theoretical $102.17 for a standard 6-hour match.

**Idle channel fee:** $0.01/hr per input + $0.01/hr per output when channel exists but not encoding. For 15 channels: ~$216/month.

**Reference data:**

| Month | Input | Output | Inactive | Gross | EDP | Net | Matches |
|---|---|---|---|---|---|---|---|
| April | $9,911 | $80,656 | $163 | $90,731 | -$18,146 | **$72,585** | 95 |
| May | $9,402 | $63,272 | $328 | $73,003 | -$14,600 | **$58,402** | 83 |
| June | $1,888 | $15,202 | $323 | $17,413 | -$3,482 | **$13,931** | 0 |

---

### 6.3 MediaPackage

**What it is:** Packaging and origin layer. Receives encoded streams from MediaLive (ingest), packages into HLS/DASH, and serves segments to CDN on request (origination). Basic packaging only — no DVR, SSAI, or DRM at this layer.

**Billing components:**

| Component | Direction | Gross rate | Net rate (post-20% EDP) |
|---|---|---|---|
| Ingest | MediaLive → MediaPackage | $0.045/GB | **$0.036/GB** |
| Origination | MediaPackage → CDN | $0.060/GB | **$0.048/GB** |

**What drives each component:**
- **Ingest:** driven by how many channels run × duration — fixed per match regardless of viewers
- **Origination:** driven by CDN edge cache warming — partially fixed (CloudFront always warms a minimum set of caches) with a large increase for high-viewership matches

**Key finding from match data:**
Origination does NOT scale linearly with viewer count. Small matches still generate significant origination because CloudFront warms edge caches regardless of audience size. Only at very high viewer counts (>150K) does origination increase substantially.

**Confirmed origination data points:**

| Match | Viewers | Origination GB | Origination cost |
|---|---|---|---|
| Apr 10 (1 match, 2,650v) | 2,650 | 2,997 GB | $143.86 |
| Apr 7 per match (1,980v total) | ~990 | 955 GB | $45.84 |
| Apr 24 per match (16,310v total) | ~8,155 | 917 GB | $44.02 |
| May 12 Al-Hilal (378,210v) | 378,210 | 9,080 GB | $435.84 |
| Monthly avg April (95 matches) | ~4,500 avg | 426 GB | $20.45 |
| Monthly avg May (83 matches) | ~5,500 avg | 495 GB | $23.75 |

**Three-tier cost model — derived from confirmed data:**

| Tier | Viewer threshold | Cost/match | Source | Accuracy |
|---|---|---|---|---|
| Small | < 10,000 viewers | **$150/match** | Apr 10 ($167), Apr 7/24 avg ($138–$167) | ±15% |
| Medium | 10,000–150,000 | **$45/match** | Monthly bill weighted average | ±35% |
| Large | > 150,000 | **$459/match** | May 12 Al-Hilal confirmed | ±5% |

**Calculator rule:** Auto-select tier based on viewer count. No formula — flat values from confirmed data.

**Historical match mode (when daily Cost Explorer cost is known):**
```
Cost = daily MediaPackage cost × viewer share %
Apply % directly to daily total — do NOT divide by match count first

Example: May 12 daily $483.18, Al-Hilal = 95% of viewers
Cost = $483.18 × 0.95 = $459.02
```

**Reference data:**

| Month | Ingest GB | Ingest cost | Orig GB | Orig cost | Gross | EDP | Net |
|---|---|---|---|---|---|---|---|
| April | 78,119 | $3,515 | 40,515 | $2,430 | $5,946 | -$1,189 | **$4,757** |
| May | 36,500 | $1,642 | 41,068 | $2,464 | $4,106 | -$821 | **$3,285** |
| June | 6,840 | $307 | 2,237 | $134 | $442 | -$88 | **$354** |

---

## 7. CDN Layer — Multiple Providers

The CDN layer is kept separate from Media Elements in the calculator and all reporting. There is one Media Elements provider (AWS) and multiple CDN providers.

### 7.1 AWS CloudFront

**Contracted flat rates (private pricing — not public tiered rates):**

| Region | Net rate/GB | Primary use |
|---|---|---|
| ME (Bahrain) | **$0.007/GB** | Primary viewer delivery to MEA |
| EU (Ireland) | **$0.004/GB** | Origin egress + emergency backup |

EU is 43% cheaper per GB than ME. Routing more traffic via EU reduces cost but increases latency for MEA viewers and creates single-point-of-failure risk.

**Excluded line items (negligible — under $50/month combined):**
- HTTP/HTTPS request charges
- Data transfer out to origin

**Cost formula:**
```
CloudFront cost = GB × $0.007 (ME delivery)
               or GB × $0.004 (EU delivery)
               or (GB × ME%) × $0.007 + (GB × EU%) × $0.004 (mixed)

GB = viewers × bitrate_Mbps × (watch_time_mins × 60) ÷ 8,192

Bitrate auto-selects by viewer count:
  < 10,000 viewers  → 4.0 Mbps
  10K–150K viewers  → 3.9 Mbps
  > 150,000 viewers → 2.8 Mbps
```

**Viewing hour cost anchors (from confirmed match data):**

| Delivery | Rate/GB | Cost per viewing hour (at 2.8 Mbps) |
|---|---|---|
| ME only | $0.007 | $0.00917/hr |
| EU only | $0.004 | $0.00524/hr |
| Blended 55%EU/45%ME | $0.00469 | $0.00614/hr |

**Reference data:**

| Month | ME GB | ME cost | EU GB | EU cost | Total |
|---|---|---|---|---|---|
| April | 3,079,836 | $21,558 | 727,230 | $2,909 | **$24,467** |
| May | 2,054,784 | $14,383 | 2,513,647 | $10,055 | **$24,438** |

**May 2026 EU spike note:** AWS ME region incident forced ~55% of MEA traffic through EU CloudFront as fallback. Under normal operations ME carries majority of delivery traffic.

### 7.2 Alibaba CDN

Billing records pending. Billed per GB (not Gbps). Will be added to model when available.

### 7.3 Other CDN providers

To be added when provider details and billing records are available.

---

## 8. Channel Configuration and Viewer Quality Tiers

### 8.1 Quality tiers and channels

| Viewer quality | Channel | Codec | Renditions | Avg actual Mbps |
|---|---|---|---|---|
| SDR (FTA) | FTA CH01–CH09 | H.264 | 432p, 720p, 1080p | ~4.0 / 3.9 / 2.8 Mbps (by size) |
| HDR Premium | SPL CH01–CH09 | H.265 | 720p, 1080p | ~5.6 Mbps (estimated) |
| 4K | CH11_SPL | H.264 | 540p, 720p, 1080p, 1440p | ~4.5 Mbps (estimated) |

**4K match channel configuration:**
When match type = 4K, the following channels are active:
1. FTA Channel (always present)
2. Premium/SPL Channel (always present)
3. **CH11_SPL only** (4K) — CH10 is NOT included

### 8.2 Viewer quality split inputs

The calculator accepts:
- % watching FTA (SDR) — default 100%
- % watching HDR (Premium)
- % watching 4K

These must sum to 100%. When split is provided, GB is calculated separately per tier and summed:

```
Total GB = (FTA_viewers × FTA_Mbps × watch_secs ÷ 8,192)
         + (HDR_viewers × 5.6 Mbps × watch_secs ÷ 8,192)
         + (4K_viewers × 4.5 Mbps × watch_secs ÷ 8,192)
```

HDR and 4K bitrates are estimated — no confirmed Conviva quality-tier breakdown available yet.

### 8.3 HDR and 4K bitrate estimates

| Tier | Max bitrate | Estimated actual | Derivation |
|---|---|---|---|
| FTA | 8 Mbps | 2.8–4.0 Mbps (by audience size) | Confirmed from 6 real matches |
| HDR | 15 Mbps | ~5.6 Mbps | Estimated: FTA_max_ratio × FTA_actual |
| 4K | 12 Mbps (CH11 max) | ~4.5 Mbps | Estimated: 4K_max_ratio × FTA_actual |

---

## 9. Per-Match Cost Formula and Calculator Logic

### 9.1 Calculator inputs and structure

#### Multi-match design
The calculator supports **multiple matches in one calculation**. There is no "concurrent matches" count input — instead the user adds each match individually using an **"Add another match" button**. Each match has its own inputs. The total is the sum of all matches.

This correctly models days like April 29 (3 matches with very different viewer counts) where each match must be calculated separately — you cannot multiply a single viewer count by match count because each match has a different audience size.

#### Per-match inputs (repeated for each match added)

**Required inputs:**

| Input | Default | Notes |
|---|---|---|
| Total viewers | — | From Conviva or estimate |
| Stream duration (hrs) | 6 hrs | Adjustable |
| Match type | Standard | Standard / 4K |
| Delivery region | ME | ME / EU / Mixed |

**Quality split inputs — visible and adjustable for every match:**

| Input | Default | Notes |
|---|---|---|
| % FTA viewers (SDR) | 100% | FTA channel — H.264, 3 renditions |
| % HDR viewers (Premium) | 0% | SPL channel — H.265, HDR10 |
| % 4K viewers | 0% | CH11 only — only shown when match type = 4K |

**Rules for quality split:**
- The three percentages must always sum to 100%
- If match type = Standard: % 4K is hidden and forced to 0%
- If match type = 4K: % 4K is shown and editable
- Default is 100% FTA because we have no Conviva quality-tier data yet
- These fields must be clearly visible — not hidden in an advanced section

**Optional inputs:**

| Input | Default | Notes |
|---|---|---|
| Avg watch time (mins) | Stream duration × 50% | If unknown, assume viewers watch 50% of stream |
| Override bitrate (Mbps) | Auto from viewer count | Auto-selects: <10K=4.0, 10K-150K=3.9, >150K=2.8 |

**Note:** Watch time and bitrate override are clearly marked as optional. The calculator works without them using defaults.

#### GB calculation per quality tier

When quality split is provided, GB is calculated separately per tier and summed:

```
FTA GB  = (viewers × FTA%)  × 4.0/3.9/2.8 Mbps × (watch_secs) ÷ 8,192
HDR GB  = (viewers × HDR%)  × 5.6 Mbps          × (watch_secs) ÷ 8,192
4K GB   = (viewers × 4K%)   × 4.5 Mbps          × (watch_secs) ÷ 8,192

Total GB = FTA GB + HDR GB + 4K GB
```

Bitrate auto-selects based on total viewer count (not per-tier viewer count):
- < 10,000 viewers → 4.0 Mbps (FTA default)
- 10,000–150,000 → 3.9 Mbps (FTA default)
- > 150,000 → 2.8 Mbps (FTA default)
HDR and 4K use fixed estimated bitrates (5.6 and 4.5 Mbps) regardless of viewer count.

### 9.2 Per-match cost formula

**For each match entered separately:**

```
STEP 1 — CALCULATE GB:
  FTA GB  = (viewers × FTA%)  × FTA_Mbps  × (watch_mins × 60) ÷ 8,192
  HDR GB  = (viewers × HDR%)  × 5.6 Mbps  × (watch_mins × 60) ÷ 8,192
  4K GB   = (viewers × 4K%)   × 4.5 Mbps  × (watch_mins × 60) ÷ 8,192
  Total GB = FTA GB + HDR GB + 4K GB

STEP 2 — CLOUDFRONT:
  Cost = Total GB × $0.007 (ME) or $0.004 (EU) or blended rate
  Accuracy: ±5%

STEP 3 — MEDIALIVE:
  Cost = $736/match
  Accuracy: ±50%
  Note: same cost regardless of viewer count — channel-driven not viewer-driven

STEP 4 — MEDIACONNECT:
  Cost = $19.26/match
  Accuracy: ±25%

STEP 5 — MEDIAPACKAGE (auto-selects by total viewer count):
  < 10,000 viewers:    $150/match  (±15%)
  10K–150K viewers:   $45/match   (±35%)
  > 150,000 viewers:  $459/match  (±5%)

TOTAL MATCH COST = CloudFront + MediaLive + MediaConnect + MediaPackage
```

**For multiple matches:** repeat above for each match, then sum all totals.

```
TOTAL DAY COST = Match 1 total + Match 2 total + Match 3 total...
```

**Example — April 29 (3 matches):**
- Al-Nassr (374,900v): calculate → large tier
- Al-Taawoun (88,250v): calculate → medium tier
- Al-Riyadh (9,970v): calculate → small tier
- Sum = total day cost

### 9.3 4K match additions

```
MediaLive 4K add-on (CH11 only):
  $8.244/hr × stream_hours
  At 6hrs: $49.46

MediaConnect LargeInstance (CH11, 4 flows):
  4 × $1.368/hr × stream_hours
  At 6hrs: $32.83

Total 4K add-on at 6hrs: $82.29
```

### 9.4 Full per-match cost summary (standard, 6hrs, ME delivery)

| Component | Cost | Type |
|---|---|---|
| CloudFront | GB × $0.007 | Variable — viewer driven |
| MediaLive | $736 | Historical average |
| MediaConnect | $19.26 | Historical average |
| MediaPackage | $45–$459 | Tier-based on viewers |
| **Total excl. CloudFront** | **$800–$1,215** | |

### 9.5 Unified calculator — no historical vs forecast split

**The calculator must have one single input form and one single result.** There is no separate historical section and forecast section.

**Why:** The formula is identical whether calculating a past match or a future match. If you enter a past match's viewer count and watch time, you get the same calculation as projected numbers for a future match. Having two modes implies different logic — which is incorrect and confuses the user.

**How demos work:** Past matches are demo entries — pre-filled inputs the user clicks to load. Once loaded they appear in the same single calculator form. No separate historical calculator mode.

**The calculator must be general purpose** — not built around any specific match. References to Al Hilal vs Al Nassr or any specific match must only appear in the demo section, not in the main calculator UI or default values.

### 9.6 Formatting rule
All cost outputs ≥ $1,000 must display with comma separator: $1,234.56 — not $1234.56.

### 9.7 Historical save
After any calculation, show a pop-up: "Save this calculation to history?" User can save or dismiss. Saved calculations appear in a history log for reference. The history log is read-only — it is not a separate calculator mode.

### 9.8 App bugs to fix

The following bugs must be corrected in the calculator application:

**Bug 1 — Date picker:**
When navigating to a future month, the calendar must not automatically highlight any date. It must show the month clean with no pre-selected date. Currently it highlights today's day number in whatever month the user navigates to (e.g. navigating to July highlights July 28 if today is June 28). This is wrong — the user should click to select a date.

**Bug 2 — 4K expected viewers field not editable:**
The 4K expected viewers input field is currently disabled or read-only. It must be editable so the user can enter the expected number of 4K viewers.

**Bug 3 — Print vs PDF:**
Remove the print button. Replace with a "Save as PDF" button that generates a properly formatted PDF of the calculation result.

**Bug 4 — Historical vs Forecast sections:**
Remove the split between historical and forecast sections. Replace with a single unified calculator as described in section 9.5.

### 9.9 Design notes

The calculator must not look AI-generated. Design requirements:
- Use proper typography, spacing, and intentional color choices
- Avoid default component library styling without customization
- Sections must be clearly labeled: Media Elements (AWS) and CDN Layer
- Static costs section must be visually distinct from variable inputs — clearly read-only
- 4K upscaling section must be visually collapsible — not always expanded
- Subscriber section must be clearly marked as informational only

---

## 10. Subscriber vs Non-Subscriber Cost

### 10.1 Purpose
This section is **informational only** — shown separately below the main result. It answers:
- What does it cost us per viewer?
- How much does each subscriber cost vs generate in revenue?
- How much does each non-subscriber cost with no revenue offset?
- Is this match profitable after subscription revenue?
- What is the break-even subscriber percentage?

**Critical design rule:** This section has NO inputs for viewers or match cost. It reads total_viewers, total_match_cost, and per-service costs directly from the main calculator output. Adding separate viewer inputs here causes the formula to break and produce zero results.

### 10.2 Subscription tiers

| Tier | Price | USD equivalent | Monthly USD |
|---|---|---|---|
| Monthly | 70 SAR/month | ~$18.67/month | $18.67 |
| Yearly | 700 SAR/year | ~$186.67/year | $15.56/month |

*SAR to USD at 3.75 rate*

### 10.3 Inputs unique to this section

| Input | Notes |
|---|---|
| Subscriber % | What % of total viewers are paying subscribers |
| Matches per month | How many matches this month — derives revenue per match |

Total viewers and all service costs are read automatically from main calculator. Not re-entered here.

### 10.4 Core formula

```
STEP 1 — Cost per viewer (identical for all viewers):
  cost_per_viewer = total_match_cost ÷ total_viewers

  Why identical: same infrastructure delivered to every viewer
  regardless of whether they pay. Cost does not change by viewer type.

STEP 2 — Per service cost per viewer:
  CF_per_viewer = cloudfront_cost  ÷ total_viewers
  ML_per_viewer = medialive_cost   ÷ total_viewers
  MP_per_viewer = mediapackage_cost ÷ total_viewers
  MC_per_viewer = mediaconnect_cost ÷ total_viewers

STEP 3 — Subscriber and non-subscriber counts:
  subscriber_count     = total_viewers × subscriber_%
  non_subscriber_count = total_viewers × (1 − subscriber_%)

STEP 4 — Revenue per subscriber per match:
  monthly_revenue_per_match = $18.67 ÷ matches_per_month
  yearly_revenue_per_match  = $15.56 ÷ matches_per_month

STEP 5 — Subscriber net (cost minus revenue):
  net_monthly = cost_per_viewer − monthly_revenue_per_match
  net_yearly  = cost_per_viewer − yearly_revenue_per_match
  Negative = profit (revenue exceeds cost)
  Positive = still a net cost

STEP 6 — Total subscriber revenue offset:
  total_sub_revenue = subscriber_count × revenue_per_match

STEP 7 — Non-subscriber total cost:
  total_non_sub_cost = non_subscriber_count × cost_per_viewer
  (No revenue — pure cost)

STEP 8 — Net match cost after subscription revenue:
  net_match_cost = total_match_cost − total_sub_revenue
  Negative = match is profitable overall
  Positive = match still costs money after revenue

STEP 9 — Break-even subscriber %:
  break_even_% = cost_per_viewer ÷ revenue_per_match × 100
  This is the minimum % of subscribers needed to break even on this match
```

### 10.5 Full output display

**Per viewer breakdown:**

| Output | Formula | Shows |
|---|---|---|
| Cost per viewer | total_cost ÷ viewers | Same for all viewers |
| CloudFront per viewer | cf_cost ÷ viewers | Largest component |
| MediaLive per viewer | ml_cost ÷ viewers | |
| MediaPackage per viewer | mp_cost ÷ viewers | |
| MediaConnect per viewer | mc_cost ÷ viewers | |

**Subscriber section:**

| Output | Formula | Shows |
|---|---|---|
| Subscriber count | viewers × sub_% | How many are subscribers |
| Cost per subscriber | cost_per_viewer | Same as all viewers |
| Revenue per subscriber/match (monthly) | $18.67 ÷ matches | What each sub generates |
| Revenue per subscriber/match (yearly) | $15.56 ÷ matches | |
| **Net per subscriber (monthly)** | cost − revenue | Negative = profit |
| **Net per subscriber (yearly)** | cost − revenue | Negative = profit |
| Total subscriber revenue | sub_count × revenue | Total offset from subs |

**Non-subscriber section:**

| Output | Formula | Shows |
|---|---|---|
| Non-subscriber count | viewers × (1−sub_%) | How many are free viewers |
| **Cost per non-subscriber** | cost_per_viewer | Pure cost, no offset |
| **Total non-subscriber cost** | non_sub_count × cost_per_viewer | Total burden from free viewers |

**Match profitability:**

| Output | Formula | Shows |
|---|---|---|
| Total match cost | from main calculator | Full pipeline cost |
| Total subscription revenue | sub_count × revenue/match | Revenue this match generates |
| **Net match cost** | total_cost − sub_revenue | Negative = profitable |
| **Break-even subscriber %** | cost_per_viewer ÷ revenue × 100 | Min % subs to break even |

### 10.6 Example — May 12 Al-Hilal vs Al-Nassr (20% subscribers, 89 matches/month)

**Match data (from main calculator):**

| Service | Total cost | Per viewer |
|---|---|---|
| CloudFront | $4,363.76 | $0.01154 |
| MediaLive | $1,990.71 | $0.00526 |
| MediaPackage | $459.02 | $0.00121 |
| MediaConnect | $120.06 | $0.00032 |
| **Total** | **$6,933.55** | **$0.01833** |

**Subscriber analysis (20% = 75,642 subscribers):**

| Metric | Monthly sub | Yearly sub |
|---|---|---|
| Cost per subscriber | $0.01833 | $0.01833 |
| Revenue per subscriber/match | $18.67 ÷ 89 = **$0.2098** | $15.56 ÷ 89 = **$0.1749** |
| **Net per subscriber** | **−$0.1915 profit** | **−$0.1566 profit** |
| Total subscriber revenue | 75,642 × $0.2098 = **$15,872** | 75,642 × $0.1749 = **$13,230** |

**Non-subscriber analysis (80% = 302,568 viewers):**

| Metric | Value |
|---|---|
| Cost per non-subscriber | **$0.01833** |
| Total non-subscriber cost | 302,568 × $0.01833 = **$5,548** |
| Revenue from non-subscribers | **$0.00** |

**Match profitability:**

| Metric | Monthly sub | Yearly sub |
|---|---|---|
| Total match cost | $6,933.55 | $6,933.55 |
| Total subscription revenue | $15,872 | $13,230 |
| **Net match cost** | **−$8,938 (profitable)** | **−$6,296 (profitable)** |
| **Break-even subscriber %** | **8.7%** | **10.5%** |

**Reading the break-even:** At 20% subscribers, this match is profitable. You only need 8.7% of viewers to be monthly subscribers (or 10.5% yearly) to fully cover the match cost. Every subscriber above that threshold is pure profit.

---

## 11. 4K Upscaling Analysis

**Purpose:** Answers "If we upgrade X matches to 4K, what is the total additional cost across all four services?"

Displayed as a **collapsible/expandable section** — additional information below the main result, not replacing it.

### 11.1 How it works

Upgrading a match to 4K means adding CH11_SPL on top of the standard FTA + SPL configuration. All three viewer types exist simultaneously:
- FTA viewers → SDR on FTA channel (always present in every match)
- HDR viewers → Premium HDR on SPL channel (always present in every match)
- 4K viewers → 4K on CH11 (the add-on cost)

The additional cost covers all four services: MediaLive (CH11 encoding), MediaConnect (LargeInstance flows for CH11 signal), MediaPackage (additional CH11 ingest), and CloudFront (4K viewers consume more GB at 4.5 Mbps).

### 11.2 Inputs

| Input | Type | Default | Notes |
|---|---|---|---|
| Number of matches to upgrade | Number field | 1 | Enter season total or any count |
| Stream duration | Dropdown | **6 hours** | Options: 4hrs, 5hrs, 6hrs, 7hrs, 8hrs |
| Expected 4K viewers per match | Dropdown | **40,000 viewers** | Adjustable — default is 40,000 |
| Avg watch time per viewer | Dropdown | **45 minutes** | Adjustable — default is 45 mins |
| Delivery region | Fixed split | **70% ME / 30% EU** | Not adjustable in this section |

**Default values rationale:**
- 40,000 4K viewers: reasonable estimate for a mid-sized match with 4K adoption
- 45 minutes watch time: conservative average — 4K viewers tend to be more engaged
- 70% ME / 30% EU: reflects normal operational routing without incidents

### 11.3 Formula — per match add-on

```
MEDIALIVE ADD-ON (CH11 only):
  $8.244/hr × stream_hrs
  At 6hrs: $49.46

MEDIACONNECT ADD-ON (4 LargeInstance flows for CH11):
  4 × $1.368/hr × stream_hrs
  At 6hrs: $32.83

MEDIAPACKAGE ADD-ON (CH11 additional ingest):
  CH11 output bitrate: 0.666 + 0.666 + 1.332 + 1.332 = 3.996 Mbps
  Additional GB: 3.996 × (stream_hrs × 3,600) ÷ 8,192
  At 6hrs: ~10.5 GB × $0.036 = ~$5.00

CLOUDFRONT ADD-ON (4K viewers only):
  4K_GB = 4K_viewers × 4.5 Mbps × (watch_mins × 60) ÷ 8,192
  ME cost = 4K_GB × 0.70 × $0.007
  EU cost = 4K_GB × 0.30 × $0.004
  Total CF = ME cost + EU cost

  At 40,000 viewers, 45 mins:
  4K_GB = 40,000 × 4.5 × 2,700 ÷ 8,192 = 59,082 GB
  ME: 59,082 × 0.70 × $0.007 = $289.50
  EU: 59,082 × 0.30 × $0.004 = $70.90
  CF total: $360.40

TOTAL ADD-ON PER MATCH:
  $49.46 + $32.83 + $5.00 + $360.40 = $447.69
  (at defaults: 6hrs, 40K viewers, 45 mins, 70/30 routing)
```

### 11.4 Total for N matches

```
Total additional cost = per_match_add_on × number_of_matches

At defaults (40K viewers, 6hrs, 45 mins):
  Per match: $447.69
  10 matches: $4,477
  50 matches: $22,385
  95 matches (full season): $42,531
```

### 11.5 Output breakdown

| Output | Formula |
|---|---|
| MediaLive add-on total | $8.244 × stream_hrs × matches |
| MediaConnect add-on total | 4 × $1.368 × stream_hrs × matches |
| MediaPackage add-on total | ~$5 × matches |
| CloudFront add-on total | 4K_GB × (0.70×$0.007 + 0.30×$0.004) × matches |
| **Total additional cost** | Sum of all above |
| **Per match add-on** | Total ÷ matches |
| **Per 4K viewer** | Total ÷ (4K_viewers × matches) |

### 11.6 Cost per 4K viewer

```
At defaults (40K viewers, $447.69/match):
  Per 4K viewer = $447.69 ÷ 40,000 = $0.01119/viewer
```

---

## 12. Demo Matches

These are confirmed real matches to be used as demos in the calculator. All data is from Conviva and AWS Cost Explorer.

### Demo 1 — Apr 10: FDL Match (Low viewership)

| Parameter | Value |
|---|---|
| Date | April 10, 2026 |
| Match type | Standard (FDL) |
| Matches on day | 1 |
| Viewers | 2,650 |
| Avg watch time | 35.93 mins |
| Total GB delivered | 2,844.14 GB |
| GB per viewer | 1.073 GB |
| Actual bitrate | 4.08 Mbps |

**Confirmed daily costs:**

| Service | Daily cost (= match cost, 1 match) |
|---|---|
| MediaLive | $4,625.32 |
| MediaConnect | $322.71 |
| MediaPackage | $167.02 |
| CloudFront | 2,844 GB × $0.007 = **$19.91** |
| **Total** | **$5,134.96** |

**Note:** MediaLive cost is high for this match because it was early April with all channels always-on. This will reduce significantly under on-demand model.

---

### Demo 2 — Apr 29: Al-Nassr vs Al-Ahli (High viewership)

| Parameter | Value |
|---|---|
| Date | April 29, 2026 |
| Match type | Standard |
| Matches on day | 3 |
| Al-Nassr viewers | 374,900 |
| Al-Nassr avg watch time | 97 mins |
| Al-Nassr viewer share of day | ~79% (374,900 / 473,120) |
| Total day GB | 869,166.61 GB |
| Al-Nassr GB (79% share) | ~686,641 GB |
| GB per viewer | 1.837 GB |
| Actual bitrate | 2.59 Mbps |

**Confirmed daily costs and Al-Nassr allocation (79%):**

| Service | Daily total | Al-Nassr (79%) |
|---|---|---|
| MediaLive | $1,598.12 | $1,262.51 |
| MediaConnect | $144.49 | $114.15 |
| MediaPackage | $227.87 | $180.02 |
| CloudFront | 869,167 GB × $0.007 = $6,084.17 | $4,806.49 |
| **Total** | **$8,054.65** | **$6,363.17** |

---

## 13. Monthly Reference Data

### 13.1 April 2026 — 95 matches

| Service | Net cost |
|---|---|
| CloudFront | $24,467 |
| MediaConnect | $5,862 |
| MediaLive | $72,585 |
| MediaPackage | $4,757 |
| **Total** | **$107,671** |

### 13.2 May 2026 — 83 matches

| Service | Net cost |
|---|---|
| CloudFront | $24,438 |
| MediaConnect | $4,186 |
| MediaLive | $58,402 |
| MediaPackage | $3,285 |
| **Total** | **$90,311** |

### 13.3 June 2026 — 0 official matches (testing period)

| Service | Net cost |
|---|---|
| CloudFront | ~$200 |
| MediaConnect | $1,504 |
| MediaLive | $13,931 |
| MediaPackage | $354 |
| **Total** | **~$15,989** |

**Note:** June costs reflect testing activity — not a clean baseline. Used for directional reference only.

### 13.4 Cost share by service

| Service | April | May |
|---|---|---|
| MediaLive | 67.4% | 64.7% |
| CloudFront | 22.7% | 27.1% |
| MediaPackage | 4.4% | 3.6% |
| MediaConnect | 5.4% | 4.6% |

### 13.5 Confirmed per-match data points from April

| Match | Date | Viewers | ML daily | MC daily | MP daily | CF GB | CF cost |
|---|---|---|---|---|---|---|---|
| FDL (1 match) | Apr 10 | 2,650 | $4,625 | $322 | $167 | 2,844 | $19.91 |
| 2 matches | Apr 7 | 1,980 total | $3,933 | $287 | $138 | 1,888 | $13.21 |
| 2 matches | Apr 24 | 16,310 total | $1,106 | $84 | $134 | 25,839 | $180.87 |
| 3 matches | Apr 29 | 473,120 total | $1,598 | $144 | $228 | 869,167 | $6,084 |
| 2 matches (May 12) | May 12 | 473,130 total | $2,095 | $126 | $483 | — | $4,593 |

---

## 14. Calculator Inputs Reference and Rate Card

### 14.1 Complete rate card

| Service | Component | Net rate | Source |
|---|---|---|---|
| CloudFront | ME (Bahrain) | $0.007/GB | AWS account manager — contracted |
| CloudFront | EU (Ireland) | $0.004/GB | AWS account manager — contracted |
| MediaConnect | Standard flow | $0.1934/hr | Bill × 80% EDP |
| MediaConnect | LargeInstance (4K/CH11) | $1.368/hr | Bill × 80% EDP |
| MediaConnect | Regional transfer | $0.008/GB | Bill × 80% EDP |
| MediaConnect | Inbound Ireland→Frankfurt | Free | Bill confirmed |
| MediaLive | FTA channel | $4.254/hr | Bill rates × channel config |
| MediaLive | SPL channel | $12.774/hr | Bill rates × channel config |
| MediaLive | Standard match (FTA+SPL) | $17.028/hr | Sum |
| MediaLive | CH11_SPL (4K) | $8.244/hr | Bill rates × channel config |
| MediaLive | 4K match total | $25.272/hr | Sum |
| MediaLive | Idle channel fee | $0.01/hr/channel | Bill line items |
| MediaLive | Historical per match | $736/match | Weighted avg Apr+May |
| MediaPackage | Ingest | $0.036/GB | Bill × 80% EDP |
| MediaPackage | Origination | $0.048/GB | Bill × 80% EDP |
| MediaPackage | Small match (<10K) | $150/match | Apr 10 confirmed |
| MediaPackage | Medium match (10K–150K) | $45/match | Monthly bill avg |
| MediaPackage | Large match (>150K) | $459/match | May 12 confirmed |

### 14.2 Bitrate auto-selection

| Viewers | Default Mbps | Data points |
|---|---|---|
| < 10,000 | **4.0 Mbps** | Apr 7 (3.93), Apr 10 (4.08) |
| 10,000–150,000 | **3.9 Mbps** | Apr 24 (3.60), Apr 29 Al-Taawoun (4.11) |
| > 150,000 | **2.8 Mbps** | Apr 29 Al-Nassr (2.59), May 12 (2.93) |

User can override at any time.

---

## 15. Items Pending Validation

| Item | Current value | Action needed | Priority |
|---|---|---|---|
| MediaLive on-demand per match | $102.17 theoretical | Validate against first on-demand season bill | High |
| MediaLive per match accuracy | ±50% | Improve with on-demand season data | High |
| HDR actual bitrate | ~5.6 Mbps estimated | Conviva quality-tier breakdown | Medium |
| 4K actual bitrate | ~4.5 Mbps estimated | Conviva quality-tier breakdown | Medium |
| Alibaba CDN rates | Pending | Obtain billing records | Medium |
| Other CDN providers | Not documented | Provide provider and billing details | Medium |
| MediaPackage medium tier | ±35% accuracy | More match-specific data points | Medium |
| Subscriber % breakdown | Not provided | Provide subscriber count per match | Low |
| CH10_SPL status | Reserved/excluded | Confirm if permanently excluded | Low |
| Model recalibration | Apr–May 2026 only | Recalibrate at new season start | Seasonal |

---

*Document prepared by Thmanyah Streaming Infrastructure*
*Last updated: June 28, 2026*
*Version 3.0 — complete rewrite from v2*
*All costs are post-EDP-discount net figures as billed by AWS*
*Overall model accuracy: ±30% — see detailed accuracy statement at top*
*Based on April–May 2026 billing data (178 matches) and 6 confirmed match data points*
