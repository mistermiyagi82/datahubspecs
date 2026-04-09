# Harm Nico's Vision → DataPraat Features

What Harm Nico wants, why he wants it, and what we need to build.

---

## The Core Philosophy

> "De kern is de norm. Wie bepaalt de norm? In norm zit afwijking en in afwijking zit betekenis."

Data means nothing without a norm. A norm reveals deviations. Deviations carry meaning. Meaning drives action. The tool must make this chain visible at every step.

---

## Vision → Functionality Map

### 1. Move Beyond Description to Diagnosis

**What Harm Nico says:**
> "Wat jullie nu hartstikke goed doen is het beschrijven. Maar de uitdaging zit erin dat je hem laat diagnostiseren."

The tool should answer questions like: *"Geeft gemeente X te veel uit aan jeugdzorg?"* — not just *"Hoeveel geeft gemeente X uit?"*

**Features needed:**

| Feature | Description |
|---|---|
| **Norm engine** | Compute norms per product category, per municipality size class, per period |
| **Deviation scoring** | Every data point gets a deviation score relative to the norm |
| **"Why" explanations** | When a number deviates, surface possible causes (peer comparison, trend break, category shift) |
| **Alderman view** | Simple dashboard answering: "Why do I spend more than my neighbor?" — not raw data, but interpreted data |

---

### 2. The 5-Step Validation Ladder (built into the product)

**What Harm Nico says:**
> "Volgens mij moeten jullie nog een, twee stappen terug, dat je op zoek gaat naar normen, te veel, te weinig, te hoog, te laag, en kwaliteit en volledigheid."

Every data delivery must pass through all 5 steps before it's trusted.

**Features needed:**

| Step | Validation | Feature |
|---|---|---|
| 1. Internal consistency | Does the data make sense on its own? | **Auto-checks on upload**: null detection, impossible values, ratio checks (clients vs. costs), type validation |
| 2. Historical comparison | Does it match previous deliveries from this gemeente? | **Time-series diff**: flag order-of-magnitude jumps, sudden category shifts, truncated files (Excel row limits) |
| 3. Peer comparison | How does it compare to similar gemeenten right now? | **Peer benchmarking**: group by size class, compute percentiles, flag outliers |
| 4. Cross-source validation | Does it match other data from the same gemeente? | **Jaarrekening cross-check**: compare uploaded data totals against annual financial reports |
| 5. External benchmark | Does it match independent data (CBS)? | **CBS integration**: population-normalized comparisons, national averages, demographic adjustments |

**Key insight from Harm Nico:** Steps 1-5 are not just quality checks — they simultaneously build the norms. Validation and norm-building are the same process.

> "Te grote afwijkingen, dat is data issue. Te kleine afwijkingen is informatie. Als je hem zo aanvliegt, dan kan je het in een slag doen."

---

### 3. Proactive Red Flags (don't wait for the user to ask)

**What Harm Nico says:**
> "Dat vind ik nou bij uitstek iets wat AI-achtige technieken voor je zouden moeten oplossen."

The tool must surface problems on its own, not passively wait for queries.

**Features needed:**

| Feature | Description |
|---|---|
| **Upload report card** | After every data upload, generate a validation report with green/yellow/red flags per step |
| **Anomaly alerts** | Proactively flag: "Gemeente X costs per inhabitant are 3x the peer average" |
| **Trend breaks** | Detect sudden changes between deliveries: "Category Y dropped 60% vs. last quarter" |
| **Cross-check alerts** | "Your data says €13.5M — the jaarrekening says €5.2M" |
| **Confidence indicator** | Every number shown with a confidence level based on how many validation steps it passed |

---

### 4. Financial Data as the Anchor

**What Harm Nico says:**
> "Data waar veel consequenties vanaf hangen, financiële data op basis waarvan betalingen worden gedaan, die zijn over het algemeen beter."

Data quality follows financial incentives. Billing data is the most reliable because providers need it to get paid.

**Features needed:**

| Feature | Description |
|---|---|
| **Facturatiedata first** | Default data source is billing/invoicing data — always show this as primary |
| **Data source quality tiers** | Visual indicator: financial data = high confidence, process data = medium, self-reported = low |
| **Known gaps warning** | Explicitly flag known gaps: e.g., "End dates of care trajectories are unreliable (no financial incentive to register)" |

---

### 5. Acknowledge the Bias in Historical Data

**What Harm Nico says:**
> "Je kijkt naar afgetopte data in plaats van werkelijke data."

A child's care trajectory reflects what was available, not what was needed. Waitlists, budget limits, and capacity constraints shape the data — but the data looks clean.

> "Als je dat er niet uithaalt en voortgaat op van we laten gewoon zien wat het geweest is, alsof dat de enige en de juiste waarheid is, dan bouw je het op drijfzand."

**Features needed:**

| Feature | Description |
|---|---|
| **"Data ≠ reality" disclaimer** | Contextual warnings on care trajectory data: "This reflects delivered care, not necessarily needed care" |
| **Waitlist correlation** | Where available, cross-reference care data with waiting list data to flag potential truncation |
| **Forecast caveats** | Any forecast built on historical data must carry a caveat about inherited bias |

---

### 6. Continuous Validation (not a one-time step)

**What Harm Nico says:**
> "Iedere maand ging er bij twee, drie, vier, vijf ziekenhuizen iets gigantisch mis. Dit is gewoon iets waar je continu bewust van moet zijn."

Errors are permanent. Even "perfect" pipelines break regularly.

**Features needed:**

| Feature | Description |
|---|---|
| **Validation on every upload** | No data enters the system without passing through the ladder |
| **Validation history** | Track validation results over time per gemeente — see if quality improves or degrades |
| **Data quality dashboard** | Overview across all 13 gemeenten: who's delivering clean data, who isn't |
| **Pipeline error detection** | Check for encoding issues (comma vs. point), truncated files, schema changes between deliveries |

---

### 7. Transparency and Proof at Every Step

**What Harm Nico says (implied throughout):**
The tool must build trust through transparency — every number traceable, every norm explainable.

**Features needed:**

| Feature | Description |
|---|---|
| **Drill-down to source** | Click any number → see the source rows, the calculation, the norm it was compared against |
| **Downloadable evidence** | Export the underlying data + validation results as CSV/Excel for independent verification |
| **Norm explanation** | "This norm is based on 8 gemeenten of similar size, using Q3 2025 data" — always visible |
| **Audit log** | Who uploaded what, when, what validation results it got, what changed |

---

### 8. The Trust Problem (design for resistance)

**What Harm Nico says:**
> "Dat vertrouwt dat wat die data jou zegt, echt veel beter is dan wat jij al jaren gelooft en wat je collega's al jaren tegen elkaar aan het roepen zijn. Dat is vertrouwen."

People reject data that contradicts their worldview. The biggest adoption challenge is human, not technical.

**Features needed:**

| Feature | Description |
|---|---|
| **Gentle framing** | Don't say "you're wrong." Say "this is what the data shows, here's how it compares, here's the evidence" |
| **Multiple perspectives** | Show the same data from different angles — let users arrive at conclusions themselves |
| **Evidence stacking** | When data contradicts expectations, stack validation evidence: "This passed 5/5 validation steps" |
| **Chat for exploration** | Let users ask questions in natural language — less threatening than a dashboard with red flags |

---

### 9. Acknowledge What Data Cannot Answer

**What Harm Nico says:**
> "Er zit ruimte tussen wat je met data kan meten en waar je feitelijk op aan het sturen bent."

Data can tell you what is efficient. It cannot tell you what is just. This gap will always exist.

**Features needed:**

| Feature | Description |
|---|---|
| **"Data stops here" boundary** | Explicitly mark where data-driven insight ends and policy judgment begins |
| **No auto-prescribe** | The tool never says "you should do X" — it says "the data suggests X, here's the context" |
| **Context fields** | Allow users to annotate data with qualitative context ("We had a staffing crisis in Q3") |

---

## Summary: The Full Feature Set to Make Harm Nico Happy

### Must-haves (core to the philosophy)

1. **5-step validation ladder** on every data upload
2. **Norm engine** — compute and display norms from peer data
3. **Deviation scoring** — every number measured against the norm
4. **Proactive red flags** — surface anomalies without being asked
5. **Financial data anchor** — billing data as primary, everything else layered on top
6. **CBS integration** — Step 5 validation against independent external data
7. **Drill-down transparency** — every number traceable to source
8. **Confidence indicators** — show how many validation steps each data point passed
9. **Upload validation report** — automatic report card per data delivery

### Should-haves (important for adoption and trust)

10. **Peer benchmarking dashboard** — compare gemeenten side by side
11. **"Why" explanations** — diagnostic answers, not just descriptions
12. **Data quality dashboard** — overview of all gemeenten delivery quality over time
13. **Chat interface** — natural language exploration of validated data
14. **Downloadable evidence** — export everything for independent verification
15. **Audit log** — full trail of uploads, validations, changes

### Nice-to-haves (mature product features)

16. **Bias/truncation warnings** — flag where historical data may not reflect reality
17. **Context annotations** — users add qualitative notes to data
18. **Multiple perspective views** — same data, different frames for different roles
19. **Waitlist correlation** — cross-reference care trajectories with capacity data
20. **Trend break detection** — automatic alerts on significant period-over-period changes

---

## What Harm Nico Would Say If He Saw This Built

> "Eindelijk snapt iemand dat je eerst moet valideren voordat je gaat presenteren."

The tool doesn't just show data — it proves the data is trustworthy, surfaces what deviates, explains why it matters, and lets the human decide what to do about it.
