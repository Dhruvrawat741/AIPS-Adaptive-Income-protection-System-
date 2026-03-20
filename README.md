# AIPS — Adaptive Income Protection System

> Built for Zepto and Blinkit delivery partners | Guidewire DEVTrails 2026

---

## What we are building and why

So here is the problem we kept coming back to. Delivery workers in India lose a chunk of their monthly income every time it rains hard, or a heatwave hits, or the government shuts down the internet in their district. None of that is their fault. And none of it is covered by any insurance product that exists right now.

The platform insurance from Zomato or Blinkit? It pays out after accidents. After hospitalizations. It is genuinely useful for those situations but it completely ignores the most common income loss event these workers face — losing ₹700 because their zone flooded for five hours during a storm. There is no hospital visit for that. No police report. The money is just gone and there is nothing they can do about it.

That is what AIPS is trying to solve.

Parametric insurance means the payout is based on an external measurable index — rainfall crossing a certain level, AQI crossing a certain level — rather than on the worker filing a loss claim. The worker does not report anything. The system watches the data feeds, detects when something bad happened, checks that the worker was actually there and working, and sends money to their UPI. That whole process takes under 2 hours. No forms. No call center. No waiting.

---

## The two ideas that actually make this different

We want to be upfront about this because most weather insurance for gig workers is honestly not that differentiated. The two things that we think are genuinely new here are the incentive delta and the score recovery supplement. Both came from reading the actual research on how Q-commerce workers earn. Neither exists in any product we could find.

**The incentive cliff.** A Zepto worker needs to hit 20 orders in a day to unlock a ₹150 bonus. If a storm forces them to stop at 19, they do not lose one order's worth of pay. They lose the entire ₹150. One disruption, half an hour before they would have finished, wipes the whole day's incentive. Standard hourly income replacement completely misses this. So we built the Incentive Delta — any payout for a trigger event that falls in the 7pm to 11pm window gets an extra ₹150 added to account for the milestone they could not reach.

**The algorithmic aftereffect.** This one took us a while to notice. When a disruption forces a worker offline, their acceptance rate falls. Their completion rate looks worse. The platform algorithm notices and starts assigning them fewer orders for the next 3 to 5 days — even after the weather clears. So the income loss from one storm actually stretches across most of a week. AIPS tracks acceptance rate and assignment frequency in the 5 days after a confirmed trigger event. If there is a measurable drop, the payout includes a Score Recovery Supplement. We have not seen any other insurance product model this.

---

## Why Zepto and Blinkit specifically

Honestly, everyone is going to build this for Zomato and Swiggy. We did not want to do that.

Q-commerce workers are in a worse position than food delivery workers for three reasons that are easy to miss. First, they work within a 2km radius of a dark store. They cannot reroute around a flooded street. When the zone goes down, they go down. Second, Zepto and Blinkit do not offer rain surge payments. Zomato does. So Q-commerce workers absorb the full cost of weather disruptions with no compensation at all. Third, the incentive cliff structure we described above is more severe in Q-commerce than in food delivery because of the higher order frequency and tighter time windows.

---

## The worker we are designing for

We are calling him Rajan. 28, Zepto partner, based out of the HSR Layout dark store in Bengaluru.

10 hours a day, 26 days a month. Gross earnings around ₹27,000 per month. After fuel and bike maintenance — we estimated this at 30 percent of gross, which is actually slightly conservative compared to the 32 percent figure from the IDInsight two-wheeler delivery study from 2024 — he takes home about ₹21,000. Roughly ₹810 a day, ₹100 an hour.

His current insurance covers accidents. That is it.

When a cloudburst shuts his zone for 5 hours including the evening peak, Rajan loses ₹500 in hourly pay and ₹200 in voided milestone bonuses. Seven hundred rupees gone. AIPS would pay him ₹636 within 2 hours. Not perfect compensation but close, and he gets it without doing anything.

---

## The demo we are building toward

We want to be specific about what the actual demo looks like because vague demos are a waste of everyone's time.

We simulate a rainstorm in Bengaluru. The OpenWeather feed shows rainfall in the HSR Layout zone crossing 6mm per hour. The trigger engine picks this up within 15 minutes. It checks that Rajan was logged in for at least 2 hours that day and that his GPS puts him in the zone. Fraud checks run. The event overlaps the 7pm to 11pm window. Payout calculated: 60 percent of his daily earning plus the ₹150 Incentive Delta, coming to ₹636. Razorpay sandbox fires the transfer.

No manual steps anywhere in that chain. That is what we are demoing.

---

## How the trigger engine works

The trigger engine is the core of the product. Everything else — premiums, fraud checks, payouts — exists to support it or is downstream of it.

Every 15 minutes, it polls three external sources: OpenWeather for rainfall, temperature, and humidity; OpenAQ for AQI data; and the SFLC.in RSS feed for internet shutdown records. It evaluates whether any trigger condition has been met for any active zone. If yes, it pulls the GPS log for workers in that zone to check exposure. It scores the claim for fraud using the rule-based checklist. Depending on that score, it either dispatches a payout automatically, sends it to a human reviewer, or denies it with a written reason. All of this is logged with a full audit trail.

---

## Weekly pricing and how we calculate it

We price weekly because Zepto and Blinkit pay their partners weekly. Aligning the premium with the payout cycle means the money for the premium is already in the worker's account when the debit hits. Monthly premiums would require workers to budget ahead in a way that is genuinely hard when your income is variable and weekly.

Every worker's premium is anchored to their Baseline Weekly Earning, which we calculate as the median of their last 8 weeks of net earnings. We use the median rather than the average because one unusually good or bad week can pull the average significantly. Eight weeks rather than four because four weeks can be skewed by a single disruption event.

```
BWE = median of (last 8 weeks of platform payout x (1 minus cost ratio))

Petrol bike cost ratio: 0.30
EV or cycle cost ratio: 0.15
```

The three tiers:

| Tier | Weekly premium | Max payout per week | Who it suits |
|------|---------------|---------------------|--------------|
| Basic | ₹70 to ₹90 | 40% of BWE | Under 5 hours a day |
| Standard | ₹100 to ₹130 | 60% of BWE | 6 to 9 hours a day |
| Premium | ₹145 to ₹180 | 80% of BWE plus incentive bonus | 10 plus hours a day |

The premium adjusts each week based on zone flood and heat history, the current season, vehicle type, how long the worker has been on the platform, and whether they claimed recently. In the MVP this is a weighted formula. We are not calling it AI because it is not — it is arithmetic with good inputs. The XGBoost pricing model is planned for Phase 3 once we have real claims data to train on.

---

## The five triggers

### Heavy rain

Rainfall data from the OpenWeather One Call API, checked every 15 minutes, measured in millimeters per hour at the dark store GPS location.

Each city has its own threshold and this was a deliberate design decision. Delhi's drainage infrastructure fails at around 2.43mm per hour. Mumbai's handles around 8mm per hour before streets start flooding. Those numbers come from a 2025 ResearchGate study on urban precipitation failure points in Indian cities. A single national threshold would mean paying out in Mumbai when nothing is disrupted, or missing events in Delhi. Basis risk — the gap between what the index says and what the worker actually experienced — is the main way parametric products fail, so we built around it from the start.

| City | Trigger |
|------|---------|
| Delhi | 5mm per hour |
| Bengaluru | 6mm per hour |
| Hyderabad | 7mm per hour |
| Mumbai | 10mm per hour |

Partial event under 4 hours: 60 percent of daily earning. Full day zone closure: 100 percent. Event in 7pm to 11pm window: add the ₹150 Incentive Delta.

### Extreme heat

We compute the Heat Index rather than using raw temperature. The Heat Index combines temperature and relative humidity into a single value that reflects how the body actually experiences the conditions. A 42°C day with 60 percent humidity hits a human body very differently than 42°C with 20 percent humidity. Both data points come from the same OpenWeather call we are already making, so no additional API is needed.

| Heat Index | Payout |
|-----------|--------|
| Above 40°C for 2 or more hours | 25% of daily earning |
| Above 45°C for 2 or more hours | 60% of daily earning |
| Above 48°C | 100% of daily earning |

These thresholds match the IMD's official heatwave definitions. A 2025 IDeasForIndia study found gig workers reduced their hours by 1.2 per day and missed 0.8 days per week during heatwave periods compared to normal weeks.

### Severe air pollution

OpenAQ API, which is open source and pulls from India's official CPCB monitoring stations. When Delhi's AQI crosses 400 the government activates GRAP Stage III — schools close, construction stops, but delivery workers keep riding because they cannot afford not to.

A TGPWU survey from 2024 found that 52 percent of outdoor gig workers reported physical symptoms directly reducing their output on severe pollution days. That is a 15 to 20 percent productivity drop per hour, which means fewer orders completed, which means less money. We frame this as income loss from impaired earning capacity rather than as a health benefit because the coverage scope is income only. The mechanism is identical either way.

| AQI | Daily payment |
|-----|--------------|
| 300 to 400 | ₹100 |
| Above 400, GRAP Stage III | ₹200 |
| Above 450, GRAP Stage IV | ₹300 |

Worker must have been logged in for at least 2 hours. We need evidence of exposure, not just that the air was bad.

### Curfew or zone closure

This is the hardest trigger to automate well and we want to be honest about that. Government notifications are not published in any machine-readable standard format. Our MVP uses a combination of scraping CAQM and state disaster management websites plus a simulated platform zone status flag. A production version of this would need either a direct data partnership with Zepto and Blinkit or a more robust scraping setup. We are not pretending otherwise.

If a zone is confirmed closed during the 7pm to 11pm window for 2 or more hours, payout is 50 percent of the daily baseline for each affected peak day.

### Internet shutdown

India runs more government-ordered internet shutdowns than almost any other country. For a delivery worker the internet is their workplace. No connectivity, no orders, no income.

We use the SFLC.in shutdown tracker, which publishes a real-time RSS feed of shutdowns by district. We cross-reference it with our app heartbeat data — if over 70 percent of workers in a pin code go silent while the neighboring pin code is fine, we call it. Full day shutdown pays 80 percent of daily earning. Partial shutdowns are pro-rated. Worker must have been active at least 3 days in the prior week, which prevents someone from signing up right before an announced shutdown.

---

## Basis risk — the known limitation we are not hiding

Every parametric insurance product has basis risk. The payout is based on an index, not on the worker's actual loss. A trigger can fire when a worker still managed to earn. A trigger can miss when a worker genuinely lost income. We cannot eliminate this.

What we do to reduce it: city-specific thresholds instead of national ones bring the index closer to the actual local disruption. Requiring minimum activity before the event filters out workers who were not going to earn anyway. Weekly payout caps control over-compensation when multiple triggers fire together. The 8-week rolling baseline means the payout reference reflects the worker's actual recent earnings, not a fixed national average.

The honest tradeoff is that workers get paid within 2 hours instead of waiting weeks for a claims process, at the cost of some precision. For someone earning ₹800 a day that speed matters a lot.

---

## Fraud detection

### Basic motion validation

Some workers use fake location apps to appear in a zone they are not in. We cross-reference GPS data with the phone's accelerometer. If GPS shows movement at 30 km/h but the accelerometer reads no physical motion, that is a flag. We are calling this basic motion validation rather than IMU-based fraud detection because that is honestly what the MVP version is. The full motion signature model using an LSTM trained on real delivery patterns is Phase 3 work.

### Passive login fraud

Someone could log in, do nothing, and wait for a trigger to fire. We require a minimum of 10 hours of actual activity during any trigger week for the Standard tier payout.

### Pre-event inactivity gaming

If a worker barely worked for 3 days before a forecast heatwave and then claims income loss, that is suspicious. We require at least 3 active days of 4 or more hours each in the 7 days before any claim week. The earnings baseline is locked when the policy starts and cannot be changed after a trigger event is announced.

### Multi-platform double claiming

A worker insured on their Zepto account could switch to Swiggy during a rain event and earn there while claiming loss here. We use a government ID-linked Unique Partner ID — one policy per person regardless of how many platform accounts they hold.

### Zone-level coordinated fraud

If 50 or more workers in the same zone show synchronized earnings drops with no external trigger confirmed, that is a Zone Fraud Alert and goes straight to a human reviewer.

### The payout decision tiers

Below 0.4 fraud score: automatic payout.
Between 0.4 and 0.75: human reviewer has 4 hours with full evidence.
Above 0.75: automatic denial with reason and appeal rights.

---

## Blockchain — the honest version

We are using blockchain for a specific reason, not because it sounds impressive.

Insurance companies have financial incentives to deny or delay claims. For a worker earning ₹800 a day, a disputed ₹500 payout is not worth fighting in court. They have no leverage. Historically that asymmetry goes one way.

Blockchain fixes one specific part of this: it makes the policy terms and the trigger records impossible to alter after the fact. When a worker activates coverage, the exact thresholds are written to a smart contract on Polygon. When a trigger fires, the Chainlink oracle writes the event on-chain. That record is permanent and public. The insurer cannot later claim the threshold was not crossed or that the data said something different.

We chose Polygon over Ethereum mainnet because Polygon settles in about 2 seconds at under ₹1 per transaction. Ethereum at current gas prices makes ₹200 micropayments economically impossible. We looked at Hyperledger Fabric and rejected it because a private chain controlled by the insurer defeats the entire transparency argument.

**Scope note:** In the MVP, the blockchain layer is simulated for the demo. Full Chainlink oracle integration and live smart contract execution are Phase 3 work. We are not claiming to have built that in a 6-week hackathon.

---

## All factors the system uses

For premium calculation: zone flood, heat, and AQI history over 2 years; vehicle type; rolling 8-week median net earnings; current season and forecast; worker tenure; prior claims in last 4 weeks.

For trigger eligibility: hours logged in on event day (min 2); GPS confirms zone presence; at least 3 active days in prior 7 days; whether event hit 7pm to 11pm window; actual measured index value; event duration.

For fraud scoring: GPS versus accelerometer match; location plausibility; claim frequency this season; earnings drop without trigger firing; activity on another platform during event; pre-event inactivity relative to forecast; identity check result; zone-level synchronized drops; recent baseline changes.

For payout calculation: Average Daily Earning (BWE divided by 7, adjusted for cost ratio); coverage tier; trigger severity tier at 25, 60, or 100 percent; whether event hit incentive window; fraud score outcome; weekly payout cap.

---

## Tech stack

```
Frontend
React with Tailwind CSS as a Progressive Web App.
Chose PWA over native because many Zepto partners run
entry-level phones with limited storage. Also works
partially offline, which matters during connectivity shutdowns.

Backend
Node.js with Express.
We are most comfortable in JavaScript.
For a 6-week build, using what the team knows well beats
trying to learn a new language mid-project.

ML layer
Python with FastAPI.
Standard for data processing. FastAPI connects cleanly
to the Node backend via REST.

Blockchain
Polygon with Solidity. Simulated in MVP, full
Chainlink integration in Phase 3.

Database
PostgreSQL for profiles, policies, payout records.
Redis for real-time trigger state and zone status cache.
TimescaleDB for earnings history and trigger logs.
Added TimescaleDB because standard PostgreSQL degrades
on time-series queries at scale and our logs grow fast.

External data
OpenWeather One Call API — rainfall, temperature, humidity
OpenAQ — AQI, open source
SFLC.in RSS — shutdown tracker
Razorpay Test Mode — mock payouts
Simulated platform API — we do not have real Zepto or
Blinkit data access so we built a mock that returns
plausible order logs and zone status flags
```

---

## What we do not cover

Health or medical expenses. Life insurance or accidental death. Vehicle repair. Personal illness or voluntary time off. Platform deactivation. Long-term disability.

Income lost because of something the worker could not predict or control. That is it.

---

## Data sources

| Need | Source | Cost |
|------|--------|------|
| Hourly rainfall | OpenWeather One Call API | Free |
| Temperature and humidity | OpenWeather One Call API | Free |
| AQI | OpenAQ | Free, open source |
| GRAP alerts | CPCB and CAQM scraper | Simulated in MVP |
| Shutdown records | SFLC.in RSS | Free, public |
| Zone closure | Platform API | Simulated mock |
| Payouts | Razorpay Test Mode | Sandbox |

---

