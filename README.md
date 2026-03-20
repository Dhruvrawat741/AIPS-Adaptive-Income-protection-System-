# AIPS — Adaptive Income Protection System

> Built for Zepto and Blinkit delivery partners | Guidewire DEVTrails 2026

---

## What we are building and why

AIPS is a parametric insurance platform for Q-commerce delivery partners — specifically Zepto and Blinkit workers. The core idea is simple: when something outside a worker's control stops them from earning (a storm, a heatwave, a curfew), they should get paid automatically without filing any paperwork.

Right now, no product does this. Platform insurance from Zomato or Blinkit covers accidents and hospitalizations. That is useful, but it completely misses the most common income loss event — a worker losing ₹600 because their zone flooded for 5 hours. There is no claim form for that. No hospital visit. The money is just gone.

We want to fix that.

The "parametric" part means payouts are triggered by external measurable events — rainfall crossing a threshold, AQI crossing a level — rather than by the worker submitting a loss report. This matters because it removes the entire claims process. The system watches, detects, and pays. The worker does nothing.

---

## Why we picked Zepto and Blinkit

We deliberately avoided Zomato and Swiggy. Everyone will build for them.

Q-commerce workers actually have a more severe income problem, and here is why. A Zepto worker has to complete, say, 20 orders to earn a ₹150 daily bonus. If a sudden storm forces them to stop at order 19, they do not just lose one order's pay — they lose the entire ₹150 bonus. One disruption wipes the whole day's incentive. This cliff-edge earnings structure makes Q-commerce workers more vulnerable than food delivery workers, not less.

On top of that, Zepto and Blinkit do not offer rain surge payments. Zomato does. So Q-commerce workers already have less protection built in.

They also operate within a 2km radius of a dark store. They cannot reroute around a flooded street the way a food delivery worker sometimes can. When the zone is down, they are down.

---

## The worker we are designing for

We are calling him Rajan. 28 years old, Zepto partner, works out of the HSR Layout dark store in Bengaluru.

He works about 10 hours a day, 26 days a month. Gross monthly earnings come to around ₹27,000. After fuel and bike maintenance — which we estimate at 30% of gross, based on an IDInsight study of two-wheeler delivery workers from 2024 — he takes home around ₹21,000 net. That is roughly ₹810 per day, ₹100 per hour.

His platform insurance covers accidents only. Completely useless when his zone floods.

When Bengaluru gets a cloudburst and his zone shuts for 5 hours including the 7pm to 11pm peak window, Rajan loses ₹500 in hourly earnings and ₹200 in voided milestone bonuses. Total loss is ₹700. AIPS would pay him ₹636 automatically, within 2 hours of the event, directly to his UPI account. No form. No call center. No wait.

---

## How weekly pricing works

We price weekly because platforms pay weekly. Zepto and Blinkit settle with partners every week, so a weekly insurance premium fits naturally into when money is already moving. A monthly premium would require workers to budget ahead — that creates friction we do not need.

### Setting each worker's baseline

Every worker has what we call a Baseline Weekly Earning, or BWE. We calculate it as the median of their last 8 weeks of net earnings.

We use 8 weeks, not 4, because 4 weeks can be skewed by one unusually good or bad week. The median is more stable than the average for the same reason. Net earnings are calculated by taking platform payouts and subtracting the cost ratio.

```
BWE = median of the last 8 weeks of (platform payout x (1 minus cost ratio))

Cost ratio is 0.30 for petrol bikes
Cost ratio is 0.15 for electric vehicles or cycles
```

The EV discount reflects reality — electric riders spend significantly less on fuel and maintenance. The IDInsight study puts average costs at 32% of gross for petrol vehicles. We use 30% as a slightly conservative estimate.

### The three tiers

| Tier | Weekly premium | Max payout per week | Who it is for |
|------|---------------|---------------------|---------------|
| Basic | ₹70 to ₹90 | 40% of BWE | Part-time, under 5 hours a day |
| Standard | ₹100 to ₹130 | 60% of BWE | Full-time, 6 to 9 hours a day |
| Premium | ₹145 to ₹180 | 80% of BWE plus incentive bonus | Power riders, 10 plus hours a day |

### What makes the premium go up or down

The base tier premium is not fixed. It adjusts each week based on a few factors:

The worker's zone has a history of flooding or severe heat: premium increases 8 to 12 percent. It is monsoon season (June through September): premium increases 5 to 15 percent depending on forecast confidence. The worker rides an EV or cycle: flat ₹10 per week discount. The worker has had 12 clean consecutive weeks with no suspicious claims: 5 percent loyalty discount. The worker filed a claim in the last 4 weeks: 5 percent increase, reflecting higher short-term risk.

In the hackathon build, this is a weighted formula. We have been transparent about that. In Phase 3, once we have actual earnings and claims data to train on, this becomes an XGBoost regression model. We are not calling it AI in the MVP because it is not — it is arithmetic with good inputs.

---

## The five triggers

These are the five events that can activate a payout. All of them use external, publicly available data. The worker reports nothing.

### Trigger 1: Heavy rain

We pull rainfall data from the OpenWeather One Call API every 15 minutes. The key metric is millimeters of rain per hour at the GPS location of the worker's dark store.

Each city gets its own threshold. This was an important design decision. A lot of parametric insurance products use a single national rainfall threshold and then wonder why payouts do not match actual losses. The reason is that Delhi's drainage system fails at 2.43mm per hour while Mumbai's tolerates around 8mm per hour before streets start flooding. That difference is documented in a 2025 ResearchGate study on urban precipitation in Indian cities. We use city-specific numbers because basis risk — the gap between what the index says and what the worker actually experienced — is the biggest technical failure mode of parametric products.

| City | Threshold |
|------|-----------|
| Delhi | 5mm per hour |
| Bengaluru | 6mm per hour |
| Hyderabad | 7mm per hour |
| Mumbai | 10mm per hour |

Payout is 60% of the daily earning for a partial shutdown event (under 4 hours). For a full-day zone closure it is 100%. If the rain hits between 7pm and 11pm — the peak earning window where milestone bonuses are at stake — there is an additional ₹150 supplement we call the Incentive Delta. More on that in a moment.

### Trigger 2: Extreme heat

Raw temperature is not enough. A 42°C day in Delhi with 60% humidity feels significantly worse on a human body than 42°C with 20% humidity. We compute the Heat Index, which combines temperature and relative humidity into a single "feels like" temperature. This is standard in occupational health research and matches how the body actually responds to heat stress.

Both data points — temperature and humidity — come from the same OpenWeather API call we are already making for rainfall. No extra API needed.

| Heat Index | Payout |
|-----------|--------|
| Above 40°C for 2 or more hours | 25% of daily earning |
| Above 45°C for 2 or more hours | 60% of daily earning |
| Above 48°C | 100% of daily earning |

These thresholds line up with India's official heatwave definitions from the IMD. A 2025 study from IDeasForIndia tracked Delhi gig workers during heatwave weeks and found they reduced working hours by 1.2 hours per day and skipped 0.8 days per week compared to normal weeks. That data informs our payout percentages.

### Trigger 3: Severe air pollution

We use the OpenAQ API, which is open source and aggregates data from India's official CPCB monitoring network. The trigger index is the 24-hour average AQI at the monitoring station closest to the worker's zone.

When Delhi's AQI crosses 400, the government activates GRAP Stage III — a pollution emergency. Schools close. Construction stops. But delivery workers keep riding because they cannot afford not to. A TGPWU survey from 2024 found that 52% of outdoor gig workers reported physical symptoms directly reducing their output during severe pollution days. Their productivity drops 15 to 20% per hour, which translates directly into fewer orders completed, which means less money earned.

This is income loss from impaired earning capacity. We frame it that way because the alternative framing — a health benefit — falls outside our coverage scope. The mechanism is the same. The framing matters for compliance.

| AQI level | Daily payment |
|-----------|--------------|
| 300 to 400 (Very Poor) | ₹100 |
| Above 400, GRAP Stage III | ₹200 |
| Above 450, GRAP Stage IV | ₹300 |

The worker must have logged in for at least 2 hours that day. We need proof of exposure, not just proof that the air was bad.

### Trigger 4: Curfew or zone closure

This one is harder to automate cleanly because government notifications are not machine-readable in any standardized way. Our plan for the MVP is a combination of scraping CAQM and state disaster management websites, plus a simulated platform zone status flag. We are honest that in the real product this would require either a direct API partnership with Zepto and Blinkit or a more robust notification scraping pipeline.

If a zone is confirmed closed during the 7pm to 11pm peak window for 2 or more hours, the payout is 50% of the daily baseline for each affected peak day.

### Trigger 5: Internet shutdown

India leads the world in government-ordered internet shutdowns. For a delivery worker, the internet is their workplace. No connectivity means zero orders, zero income.

We use the SFLC.in shutdown tracker, which publishes a real-time RSS feed of shutdowns by district and time. It is run by the Software Freedom Law Centre and is the most complete public record of Indian internet shutdowns we could find. We cross-reference their data with our own app heartbeat monitoring — if more than 70% of workers in a pin code stop responding while the neighboring pin code is fine, we flag a shutdown event.

Payout is 80% of the daily earning for a full-day shutdown. For partial shutdowns we pro-rate by the number of affected hours. The worker must have been active at least 3 days in the prior week to be eligible. This prevents someone from signing up right before an announced shutdown.

---

## The incentive delta: a feature specific to Q-commerce

This deserves its own section because it is one of the most important ideas in the product and it is unique to how Q-commerce workers earn.

Delivery workers on Zepto and Blinkit earn a base amount per order plus milestone bonuses for hitting daily or weekly order counts. The bonus is binary — you either hit the milestone and get ₹150, or you miss it and get nothing. There is no partial credit for 19 out of 20 orders.

This creates what we call the incentive cliff. If a storm hits at 9pm and forces a worker to stop at 19 orders instead of 20, they lose not just the income from the last order but the entire ₹150 milestone. A 30 to 60 minute disruption during peak hours can reduce daily income by 15% or more.

Standard hourly income replacement would not capture this. Our Incentive Delta supplement adds ₹150 to any payout for a trigger event that falls within the 7pm to 11pm window. That window is when milestone bonuses are being earned. If the disruption hits there, the payout accounts for the cliff, not just the missed hours.

No other insurance product we have found models this. It is specific to the economics of Q-commerce work and it came directly from reading research on how gig workers actually earn.

---

## How we handle fraud

Parametric insurance is actually easier to defraud than traditional insurance in some ways. Because payouts are automatic and based on external triggers, a bad actor does not need to fabricate a loss — they just need to appear to be in the right place at the right time when a trigger fires.

Here are the specific fraud vectors we are defending against and how.

### GPS spoofing

Workers sometimes use fake location apps to make their phone appear in a zone it is not in. This is common enough that the Times of India ran a story about it in 2025.

Our defense is to cross-reference GPS coordinates with the phone's accelerometer and gyroscope — together called the IMU (Inertial Measurement Unit). If the GPS shows the worker moving at 30 km/h on a bike but the accelerometer reads zero physical motion, that is a flag. Real bike movement has a signature: speed variation, minor vibrations, turns. A fake GPS path does not.

In the MVP this is a threshold check. In Phase 3 we plan to train an LSTM — a type of neural network that is good at detecting patterns in time-series data — on real delivery movement logs to distinguish genuine riding patterns from simulated ones.

### Pre-event inactivity

Someone who knows a heatwave is forecast could sign up for coverage, avoid working for a few days to maximize their apparent "loss," and then claim. We prevent this with two rules. First, a worker must have at least 3 active days of 4 or more hours each in the 7 days before any claim week. Second, the baseline earning figure is locked when the policy starts. It cannot be updated after a trigger event is announced.

### Working on another platform during the event

A worker insured under their Zepto account could shift to Swiggy during a rain event, earning there while claiming a loss on the Zepto side. Our defense is a government ID-linked Unique Partner ID. One policy per person, regardless of how many platform accounts they hold. If GPS shows consistent movement patterns during the event window, we investigate further.

### Coordinated zone fraud

If 50 or more workers in the same zone all show a synchronized earnings drop with no corresponding external trigger event, that is a Zone Fraud Alert. The probability of that happening naturally is very low. It goes to a human reviewer immediately.

### The three-tier decision

Every potential payout passes through this logic:

Fraud score below 0.4: payout executes automatically. No human involved.
Fraud score between 0.4 and 0.75: the claim is held and a human reviewer has 4 hours to approve or deny with the full evidence in front of them.
Fraud score above 0.75: automatic denial. The worker receives a reason and has the right to appeal.

In the MVP, fraud scores come from a rule-based checklist. It is not ML yet. But the structure — the three tiers, the human review queue, the appeal rights — is designed to work the same way regardless of whether the score comes from rules or a model.

---

## Why we are using blockchain

We want to be upfront about why blockchain is in this product, because a lot of projects add it as decoration.

The real problem is this: insurance companies have strong financial incentives to deny or delay claims. For a worker earning ₹800 a day, a disputed ₹500 claim is not worth fighting legally. The insurer knows this. Traditionally, the worker has no recourse.

Blockchain addresses one specific part of this: it makes the policy terms and the trigger records tamper-proof. When a worker activates coverage, the exact thresholds — "Bengaluru rainfall trigger: 6mm per hour" — are written to a smart contract on the Polygon blockchain. They cannot be changed after that. By anyone. Including us.

When a trigger fires, the data from the external API is brought on-chain by a Chainlink oracle — a service specifically designed to bring real-world data onto blockchains in a verifiable way. This creates a permanent, public record: "At 21:43 on June 14, rainfall in HSR Layout zone exceeded 6mm per hour." The insurer cannot later claim the threshold was not crossed. The record exists and is public.

The payout smart contract reads this record and either executes the payout automatically or routes it to human review based on the fraud score. The human reviewer's decision is also written on-chain. Every approval and every denial is permanently recorded, with the reason.

### Why Polygon and not Ethereum mainnet

Ethereum mainnet is too slow and too expensive for this use case. Transactions take 12 seconds and cost significant gas fees. We are processing 15-minute trigger polls and paying out amounts as small as ₹200. The economics do not work.

Polygon is an Ethereum Layer 2 network. It processes transactions in about 2 seconds and costs under ₹1 per transaction. Smart contracts are written in standard Solidity and work identically to Ethereum. Several Indian fintech projects have already used it in the RBI sandbox environment.

We also considered Hyperledger Fabric, which is a private permissioned blockchain used by enterprises. We rejected it because a chain controlled by the insurer defeats the entire transparency purpose. Workers need to be able to verify the chain independently, not trust that the insurer is running it honestly.

### What humans can and cannot do in this system

Humans review fraud cases. They approve or deny held claims. They handle appeals. Their decisions are recorded on-chain.

Humans cannot alter the trigger record after the fact. They cannot change policy terms once activated. They cannot block a payout the smart contract already executed. They cannot deny a claim without the reason being written permanently to the chain.

The design is intentional. Human judgment is valuable for ambiguous fraud cases. It is a liability for routine approved payouts.

---

## All the factors we use

We wanted to list every input the system uses, organized by what decision each one feeds into.

### For calculating the weekly premium
The worker's zone flood, heat, and AQI history over the past 2 years. Vehicle type — petrol bike versus EV or cycle. The rolling 8-week median of net earnings. Current season and IMD seasonal forecast. How long the worker has been on the platform. Prior claims in the last 4 weeks.

### For checking if a worker is eligible for a payout
Hours logged in on the event day — minimum 2 hours. GPS confirmation the worker was in the affected zone. Activity in the 7 days before the event — minimum 3 active days. Whether the event overlapped the 7pm to 11pm incentive window. The actual measured index value: rainfall in mm, Heat Index in degrees Celsius, AQI number. How long the event lasted — partial versus full day rate.

### For scoring fraud risk
GPS movement matched against accelerometer readings. Whether location changes are physically plausible given time elapsed. Number of claims filed this season. Whether earnings dropped without a corresponding trigger event firing. Whether the worker was active on another platform during the event window. Whether they went inactive in the days before a publicly forecast bad weather event. Identity verification result from the event-time check. Whether the zone as a whole shows suspicious synchronized drops. Whether the earnings baseline was recently changed before the event.

### For calculating the final payout amount
The Average Daily Earning — BWE divided by 7, adjusted for vehicle cost ratio. Coverage tier selected by the worker. Severity tier of the trigger event — Tier 1 at 25%, Tier 2 at 60%, Tier 3 at 100% of daily earning. Whether the event hit the 7pm to 11pm incentive window. The fraud score outcome. Weekly payout cap based on tier percentage of BWE.

---

## One more factor we are adding: platform score recovery

This came out of reading the research carefully. When a disruption forces a worker offline, their platform rating often drops in the following 3 to 5 days because their order acceptance rate falls and their completion rate looks worse. The platform's algorithm then assigns them fewer orders during that window — so the income loss continues even after the weather clears.

No insurance product we found addresses this. AIPS will track changes in acceptance rate and order assignment frequency in the 5 days after a confirmed trigger event. If a measurable drop occurs, the payout includes a Score Recovery Supplement — a small additional amount to compensate for the algorithmic penalty that follows the disruption.

We think this is genuinely novel. The research on gig worker income volatility makes clear that the damage from a disruption outlasts the disruption itself, but no one has tried to insure the aftereffect.

---

## Our tech stack and why we made these choices

```
Frontend
React with Tailwind CSS, built as a Progressive Web App

We chose PWA over a native Android app for one practical reason:
many Zepto partners use entry-level phones with limited storage.
A PWA installs like an app but does not require Play Store space.
It also works partially offline, which matters during the exact
events we are insuring against.

Backend
Node.js with Express

We are all more comfortable in JavaScript than in Go or Java.
For a 6-week build, using what the team knows well is the right call.

ML layer
Python with FastAPI

Python is the obvious choice for anything involving data processing
and model serving. FastAPI is lightweight and easy to connect to
the Node backend via REST.

Blockchain
Polygon with Solidity smart contracts, Chainlink oracles

Reasons explained in the blockchain section above.

Database
PostgreSQL for worker profiles, policies, and payout records
Redis for caching the real-time trigger state and zone status
TimescaleDB for storing time-series data — earnings history,
trigger event logs, fraud score history

We added TimescaleDB specifically because standard PostgreSQL
gets slow on time-series queries once you have millions of rows.
The earnings history and trigger log tables will grow fast.

External APIs
OpenWeather One Call API — rainfall, temperature, humidity (free tier)
OpenAQ API — AQI data, open source and free
SFLC.in RSS feed — internet shutdown tracker, public
Razorpay Test Mode — mock UPI payouts for the demo
Simulated Zepto and Blinkit partner API — we do not have real
access to platform data, so we are building a mock that returns
plausible order logs and zone status flags
```

---

## What we are actually building in 6 weeks versus what comes later

We want to be honest about scope. Here is the line.

| Feature | In the hackathon build | Post-hackathon |
|---------|----------------------|----------------|
| Trigger engine | Rule-based, 5 triggers, polling every 15 min | ML-enhanced with adaptive thresholds |
| Premium calculation | Weighted formula | XGBoost trained on historical data |
| Fraud detection | Rule checklist plus GPS-accelerometer check | LSTM model plus Isolation Forest ensemble |
| Platform integration | Simulated mock API | Real Zepto and Blinkit partnership |
| AQI data | OpenAQ aggregated | Direct CPCB feed plus edge sensors |
| Payouts | Razorpay sandbox | Live UPI |
| Identity | Government ID simulation | Aadhaar OTP plus e-Shram ID |
| City coverage | Delhi, Mumbai, Bengaluru, Hyderabad | Pan-India |

The rule-based MVP is not a shortcut. For a product that affects someone's income, explainability matters more than model sophistication at launch. A worker should be able to understand why they did or did not get paid. A formula can be explained. A neural network output often cannot.

---

## 6-week build plan

Phase 1, weeks 1 and 2 (now): Research, persona definition, trigger design, premium model, system architecture, this README, basic onboarding wireframe.

Phase 2, weeks 3 and 4: Worker registration with simulated ID verification. Policy engine with live BWE calculation. Trigger engine connected to OpenWeather and OpenAQ. Claims engine with Razorpay sandbox payout. Worker dashboard showing coverage status and payout history.

Phase 3, weeks 5 and 6: ML premium pricing model. Fraud detection with GPS and accelerometer cross-check. Insurer admin dashboard with zone risk map. Blockchain smart contract integration on Polygon testnet. End-to-end demo where we simulate a rainstorm and show the automatic payout happening.

---

## What we do not cover

These are hard limits, not gray areas.

Health or medical expenses. Life insurance or accidental death. Vehicle repair costs. Personal illness or voluntary time off. Platform deactivation or algorithm changes. Long-term disability.

AIPS covers only income lost because of an external event the worker could not predict or control — weather, pollution, curfews, internet shutdowns.

---

## Data sources

| What we need | Where we get it | Cost |
|--------------|----------------|------|
| Hourly rainfall | OpenWeather One Call API | Free tier |
| Temperature and humidity | OpenWeather One Call API | Free tier |
| Air quality index | OpenAQ API | Free, open source |
| GRAP stage alerts | CPCB and CAQM scraper | Simulated in MVP |
| Internet shutdown records | SFLC.in RSS tracker | Free, public |
| Zone closure data | Platform API | Simulated mock |
| UPI payouts | Razorpay Test Mode | Sandbox |

---

## Submission links

GitHub repository: to be added
2-minute demo video: to be added
Hackathon: Guidewire DEVTrails 2026

---

AIPS: because the worker who delivered your groceries in the rain deserves a safety net.
