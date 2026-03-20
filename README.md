# Adversarial Defense & Anti-Spoofing Strategy

> **README — Idea Document Update · DEVTrails 2026 · Parametric Insurance Platform**

---

## Context

A coordinated syndicate of **500+ delivery workers** has been identified using free GPS-spoofing
applications to fake their location inside active weather alert zones. While sitting safely at home,
they trick the platform into believing they are stranded in a severe flood zone — triggering mass
false payouts and draining the liquidity pool before any anomaly is detected.

This section details our complete architectural response across three dimensions:
1. How we **differentiate** a real stranded worker from a bad actor
2. What **data signals** we use beyond GPS coordinates
3. How we handle **flagged claims** without penalising honest workers

---

## Problem Analysis — Why This Attack Works

Before the defense, it is important to understand exactly how the exploit works
and which architectural gaps make it possible.

### How the Attack Works (Step by Step)

1. A delivery worker installs a free GPS spoofing app — **MockGPS, Fake GPS GO** — freely available
   on Android, no root access required
2. They override their phone's reported location to a known red-alert weather zone at the OS level
3. They submit a payout claim — the platform's weather API confirms the zone is active
4. Platform sees: valid GPS coordinate inside a valid weather zone → **payout triggered**
5. 500 workers do this simultaneously, coordinated via a private **Telegram group**
6. The liquidity pool is drained before any anomaly is detected

> **The attack is elegant because it targets the single weakest assumption in the system:
> that the phone is telling the truth about its location.**

---

### The 5 Loopholes Being Exploited

| # | Loophole | Why it is exploitable |
|---|----------|-----------------------|
| 1 | **GPS is the only verification signal** | GPS is a software value on Android — any app can override it at OS level with no root needed |
| 2 | **No behavioral memory** | The system treats every claim in isolation with no record of where a worker normally operates, what time they work, or whether they have an active order |
| 3 | **No cross-device correlation** | 500 simultaneous claims from one zone looks like a disaster to the system, not fraud — there is no mechanism to detect social coordination |
| 4 | **No physical sensor validation** | The phone's accelerometer, WiFi network, cell tower, and microphone are never checked — only the GPS coordinate is trusted |
| 5 | **Instant payout with no cooling window** | Money leaves the pool before post-event satellite data, carrier records, or road closure data can retroactively confirm or deny the claim |

---

### What We Are Building to Fix This

We are not patching a single hole. We are replacing the single-signal trust model with a
**multi-layer, cross-signal verification architecture** where:

- ✅ A genuinely stranded worker passes naturally — their device, sensors, delivery history,
  and order data all independently point to the same location
- ❌ A fraudster fails — because faking GPS is trivial, but simultaneously faking GPS + WiFi
  location + cell tower data + sensor physics + behavioral history + avoiding syndicate detection
  is **economically irrational**

The three sections below explain exactly how each layer works.

---

## System Architecture — End-to-End Flow

The diagram below shows how a claim moves through all four defense layers, the composite risk
scoring, and the three UX outcome paths — in a single view.

![Anti-Spoofing System Flow](./system_flowchart.png)

*Fig 1 — Complete claim processing flow: submission → 4 defense layers → risk score → payout decision*

---

## 1. The Differentiation

**How our AI/ML architecture differentiates a genuinely stranded delivery partner from a bad actor
spoofing their location.**

### The Core Insight

GPS is a single software value on Android — overridable by any app, no root required. Our
architecture never trusts GPS alone. Instead, it asks a fundamentally different question:

> **"Does this entire device agree that this person is where they claim to be?"**

A real stranded worker has a phone whose GPS, WiFi, cell tower, accelerometer, and order history
all independently point to the same location. A fraudster sitting at home does not.

---

### Four-Layer Differentiation Model

| Layer | What it checks | Why it matters |
|-------|---------------|----------------|
| **Layer 1 — Device Trust** | Mock location flag (`isMockLocationEnabled` API), Google Play Integrity API, TEE hardware attestation | GPS can be faked by any app — device hardware and OS integrity cannot |
| **Layer 2 — Sensor Fusion** | GPS coordinate vs WiFi SSID registered location vs nearest cell tower ID vs accelerometer/gyroscope variance | All four must agree — one contradiction breaks the pattern |
| **Layer 3 — Behavioral ML** | XGBoost model on 90-day route history, active order coordinates, time-of-day pattern | A worker who has never delivered in Zone 7 and has no active order is statistically anomalous |
| **Layer 4 — Network Graph** | DBSCAN clustering on social connections, claim burst timing, exact coordinate duplication | 500 people submitting within 3 minutes is a coordination signature, not a weather signature |

**Layer 1 detail:** Android exposes an `isMockLocationEnabled` API. If mock location is active,
the claim is immediately downgraded. A genuinely stranded delivery worker has zero reason to
have this enabled. We additionally run Google Play Integrity API to verify the device has not been
tampered with at the OS level.

**Layer 2 detail:** If GPS claims "Dharavi flood zone" but the device is connected to
`Sharma_Home_5G` — a WiFi network registered 12km away — that contradiction is an immediate
fraud signal. Cell tower triangulation (OTDOA — Observed Time Difference of Arrival) confirms
the device's physical location independently of GPS. A person sitting indoors also has
near-zero accelerometer variance — contradicting an active storm claim.

**Layer 3 detail:** An XGBoost model trained on each individual worker's own 90-day GPS history
scores how anomalous the claimed location is. We also check order correlation: was this worker on
an active delivery when the weather hit? A worker with an active pickup assignment in the claimed
zone gets their claim score boosted toward legitimate.

**Layer 4 detail:** Individual fraud is caught by Layers 1–3. Organised rings are caught here.
We build a graph where nodes are workers and edges are connections (same referral chain, same
onboarding IP subnet, same device model batch). DBSCAN clustering identifies dense connected
groups submitting simultaneously. Real stranded workers trickle in over 30–60 minutes as they
discover they are stranded. A 500-claim burst in 3 minutes is statistically impossible in nature.

---

> **The key differentiator:** A sophisticated attacker can fake GPS. They cannot simultaneously
> fake GPS + WiFi network location + cell tower triangulation + accelerometer physics +
> a 90-day delivery history + exact coordinate variance + avoid being clustered with 499 others
> doing the same thing at the same moment.
>
> Each additional layer raises the fraud cost until it exceeds the payout value.

---

## 2. The Data

**The specific data points — beyond basic GPS coordinates — our system analyzes to detect a
coordinated fraud ring.**

| Signal | What it tells us | Fraud indicator |
|--------|-----------------|-----------------|
| **Mock location flag** | Android API reveals whether GPS override is active at OS level | Flag = ON → immediate score downgrade |
| **WiFi SSID registered location** | Every WiFi network has a registered physical address. If the device is on a home network 15km from the claimed zone, GPS is lying | SSID location ≠ GPS location |
| **Cell tower ID & OTDOA** | Carrier-side triangulation places the device independently of GPS — cannot be overridden by any app | Tower zone ≠ claimed zone |
| **Accelerometer & gyroscope** | A person cycling or walking in a storm has active movement readings. A person at home has near-zero variance | Flat sensors during claimed active zone |
| **Ambient audio classifier** | On-device YAMNet model (~3MB) classifies background noise passively. A flood zone has rain, wind, traffic sounds. A silent room is anomalous | Silent audio in claimed storm zone |
| **Active order correlation** | Was this worker on a delivery when weather hit? Pickup and dropoff should place them near the claimed zone. Stationary for 2hrs with no order = strong fraud signal | No active order in claimed zone |
| **Claim burst timing** | Real claims trickle in over 30–60 minutes. 200 claims in a 3-minute window is a Telegram-coordination signature, not a weather signature | > 50 claims in < 5 minutes |
| **Social graph clustering** | Same referral chain, same onboarding IP subnet, same device model batch — DBSCAN finds dense connected clusters submitting together | Cluster size > configurable threshold |
| **Exact coordinate duplication** | Real GPS has natural variance of ±5–10m. Fraudsters copying a Telegram coordinate submit pixel-perfect identical coordinates — physically impossible in nature | Identical coordinates across N workers |
| **Multi-source weather consensus** | Cross-reference IMD/NDMA with Sentinel-2 SAR satellite imagery, Google Maps road closures, and IoT rain gauge readings | Zone is not activated until ≥ 3 independent sources agree |

---

### Dynamic Canary Watermarking — A Unique Signal for Ring Detection

When a weather event fires, each device receives the live zone polygon with a **tiny unique
coordinate offset (~30 metres — completely invisible on a phone map)**. If 200 workers submit
claims using the same watermarked boundary, they all copied from one shared source.

This is a **zero-false-positive syndicate detection mechanism** — a genuinely stranded worker
never compares GPS polygon vertices with 199 other people on Telegram.

> **Note on zone dynamics:** Weather zones are computed in real-time from live APIs — they
> cannot and should not be pre-defined. The watermark is applied dynamically at query time,
> so it works regardless of how frequently zones change.

---

## 3. The UX Balance

**How our workflow handles flagged claims without unfairly penalising honest gig workers who
may be experiencing a genuine network drop in bad weather.**

### Core Principle

> We never deny a claim outright based on automated scoring alone.
> We **hold, explain transparently, and re-verify passively.**
>
> The message to a flagged worker is:
> *"Your claim is being verified due to high activity in this area.
> You will hear back within 2 hours. Your claim is safe."*
>
> **NOT:** *"Your claim is suspicious."*

---

### Three-Path Claim Routing (Composite Risk Score 0–100)

| Score | Decision | What happens |
|-------|----------|--------------|
| **0 – 30** | ✅ **Auto-Approve** | Instant payout. No friction. No message needed. |
| **30 – 70** | ⏳ **Soft Hold (2 hr)** | Claim held. Worker receives a friendly, non-accusatory notification. Passive re-verification runs silently in background. If cleared, payout releases automatically — worker never needs to do anything. |
| **70 – 100** | ❌ **Deny + One-Tap Appeal** | Claim denied but worker immediately sees an appeal button. Human reviewer responds within 4 hours. ML model learns from human decisions — wrongly flagged workers added to permanent verified list. |

---

### What Happens During the Soft Hold Window

The worker does nothing. The system re-verifies passively and silently:

| Check | What it looks for |
|-------|-------------------|
| **Persistence check** | Does the worker's phone stay in the zone for another 20 minutes? A genuinely stranded worker stays. A fraudster at home may move. |
| **Corroborating witness** | Is there another low-risk, unconnected worker with an active claim in the same zone? Independent corroboration is strong legitimacy evidence. |
| **Cell tower confirmation** | Does a carrier-side cell tower in the zone show this device as actively connected? GPS-independent and cannot be spoofed by any app. |
| **Weather event maturity** | As satellite and IoT gauge data accumulates post-event, the system retroactively confirms or denies zone conditions with higher confidence. |

---

### Specific Protections for Genuine Network Drops in Bad Weather

Bad weather degrades GPS signal accuracy. Our system is explicitly designed to account for this:

- **NDMA/IMD red alert relaxation** — When a government authority declares a red alert for a
  zone, behavioral anomaly thresholds for workers in that zone are automatically relaxed. We give
  more trust, not less, during confirmed real disasters.

- **GPS degradation is an expected condition** — Poor GPS accuracy and network drops during
  storms are treated as normal, expected conditions — not fraud signals. A worker with degraded
  GPS during a storm is not penalised for it.

- **Reputation-based soft-trust** — Workers with 6+ months on the platform, a clean claim
  record, and high delivery count receive automatic soft-trust status. Their claims are
  auto-approved even at moderate risk scores. Reputation is earned and meaningfully rewarded.

- **Escrow payout for new workers** — New or low-history workers receive a 4-hour escrow
  payout. The claim is accepted immediately but the money releases after post-event verification
  against satellite imagery and carrier data. The worker is told upfront:
  *"Payout confirmed — releases within 4 hours once weather conditions are verified."*

---

## Architectural Principle

> The goal is not to build a system that catches every fraudster at the cost of mistreating
> honest workers.
>
> The goal is to **raise the cost of fraud so high** — across device trust, physics, behavioral
> history, social graph, and timing — that the fraud becomes economically irrational, while
> ensuring that every genuine claim from an honest worker is resolved quickly, fairly, and
> with dignity.

---

## Technology Stack Summary

| Layer | Problem solved | Technology |
|-------|---------------|------------|
| Device Trust | GPS OS-level spoofing | Android `isMockLocationEnabled`, Google Play Integrity API, StrongBox TEE |
| Sensor Fusion | Single-source GPS trust | Fused Location Provider, WiFi SSID mapping, OTDOA carrier API |
| Behavioral ML | Anomaly and teleportation detection | XGBoost, PostGIS spatial history, Redis Time-Series |
| Network Graph | Syndicate ring detection | Neo4j, Apache Kafka, DBSCAN (Python NetworkX) |
| Weather Oracle | Single API dependency risk | Sentinel-2 SAR, IMD/NDMA, HERE Traffic, IoT rain gauges |
| Canary Watermark | Coordinate sharing detection | Per-session polygon offset, backend diff engine |
| Payout Gate | Risk-proportionate response | Weighted ensemble score, human review queue, escrow engine |

---

*DEVTrails 2026 — Parametric Insurance Platform — Phase 1 Submission*
