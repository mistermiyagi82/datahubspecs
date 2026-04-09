# The Philosophy of Data — Harm Nico Plomp's Framework

Based on the DataPraat meeting of April 3rd, 2026.

---

## Core thesis

> "In norm zit afwijking en in afwijking zit betekenis."
> (In norms sit deviations, and in deviations sits meaning.)

Data on its own says nothing. Data only becomes information when measured against a norm. Without a norm, there is no deviation. Without deviation, there is no meaning. Without meaning, there is no action.

The fundamental question is not "what does the data say?" — it is **"who determines the norm?"**

---

## The Gartner Ladder applied to data

Harm Nico references the Gartner analytics maturity model, but grounds it in practice:

| Level | Question | What it takes |
|-------|----------|---------------|
| **Describing** | What happened? | Charts, tables, aggregations — relatively easy |
| **Diagnosing** | Why did it happen? | Norms, comparisons, anomaly detection — the real challenge |
| **Prescribing** | What should we do? | Combining data with context, policy, and human judgement |

Most data tools stop at describing. The value — and the difficulty — starts at diagnosing.

Describing is what AI already does well: show actuals, pivot data, generate charts. But description without diagnosis is decoration. A chart that shows youth care costs rising is useless without knowing: is that normal? Is it more than comparable municipalities? Is it a data error or a real trend?

---

## The 5-step validation ladder

Harm Nico developed this framework while working with hospital data (daily deliveries from all Dutch hospitals — admissions, diagnoses, medical procedures). The same logic applies to any data domain.

Each step adds a layer of external truth to validate against:

### Step 1 — Internal consistency (within a single delivery)

Check the data against itself.

- Are the fields filled?
- Do the values make sense?
- Are the ratios logical? (e.g., number of clients vs. number of procedures)
- Are there impossible values? (negative costs, dates in the future)

This catches: typos, missing fields, obvious errors.

This misses: systematic errors that are internally consistent but wrong.

### Step 2 — Previous delivery from the same source

Compare this delivery with the last one from the same organization.

- Is the order of magnitude the same?
- 1,000 patients this month vs. 10,000 last month? Something is wrong.
- Sudden jumps or drops in specific categories?

This catches: delivery errors, truncated files (Excel cutting off rows), format changes.

This misses: errors that existed in previous deliveries too (inherited bias).

### Step 3 — Other organizations at the same moment

Compare with peer organizations delivering similar data.

- How does this municipality compare to others of similar size?
- Is the cost per inhabitant wildly different?
- Are the ratios between care types consistent across the group?

This catches: organizational anomalies, outliers that signal data issues OR real differences.

This misses: if all organizations have the same systematic error.

### Step 4 — External data from the same source

Use other data from the same organization that comes through a different channel.

- Annual financial reports (jaarrekening)
- Council meeting minutes (raadsvergaderingen)
- Personnel data, capacity data
- The annual report says €5.2M in youth care — your data says €13.5M. Problem.

This catches: cross-channel inconsistencies, data that's internally consistent but factually wrong.

This misses: if all channels are equally wrong (contaminated source).

### Step 5 — Fully independent external data

Use data the organization has no influence over.

- CBS (national statistics): population counts, demographics
- VNG research data
- National benchmarks

This is the "ultimate external source." If your data says a small municipality has hospital-level costs per capita, something is fundamentally off — or you've found something real.

This catches: the things no internal validation can catch.

This misses: almost nothing, but CBS data itself can occasionally be wrong.

### The key insight

> Each step has value (it adds a reference point) **and** a potential contamination (the reference itself might be biased). You build outward from internal to external, each layer adding confidence.

Large deviations = probably a data issue.
Small deviations = probably real information.

The same analysis that validates data quality **also** produces the norms you need for diagnosis. They are the same process.

---

## The financial data principle

> "Data waar veel consequenties vanaf hangen, financiële data op basis waarvan betalingen worden gedaan, die zijn over het algemeen beter dan data die gaat over kwaliteit."

Data quality follows financial incentives. Where money is at stake, data is maintained. Where it's not, data degrades.

**The police example:** At a police force, the most accurate data was overtime hours — because every officer meticulously tracked their overtime (it affected their paycheck). The worst data was about actual working hours and case details — no financial consequence, no accuracy.

**The youth care example:** Invoicing/billing data (facturatiedata) is reliable because providers need it to get paid and municipalities need it to manage budgets. But start/stop dates of care trajectories? The start date is registered (needed to begin billing), but the end date often isn't (no financial consequence).

**Implication for DataPraat:** Always anchor analyses in financial data first. It's the most trustworthy foundation. Layer quality/process data on top with appropriate caveats.

---

## The gap between data and reality

Data can never fully capture reality. There is always space between what data says and what is actually true — especially in domains with societal and political dimensions.

### Three sources of this gap:

**1. Biased data (afgetopte data)**

Youth care trajectories in the data don't reflect what a child *needed* — they reflect what was *available*. If the right care wasn't available (waiting lists, budget constraints), the child received something else. The data then shows a care path that looks intentional but was actually compromised by circumstance.

> "Je kijkt naar afgetopte data in plaats van werkelijke data."
> (You're looking at truncated data instead of actual data.)

**2. Paradigm conflicts**

When data contradicts someone's worldview, the natural response is to reject the data, not adjust the worldview.

> "Je krijgt informatie van de data die niet aansluit bij wat jij denkt, weet, vindt van de wereld."
> (You receive information from the data that doesn't match what you think, know, believe about the world.)

A left-leaning municipality has a paradigm about how social services should work. A right-leaning one has a different paradigm. Data that challenges either paradigm faces resistance — not because the data is wrong, but because accepting it requires changing beliefs.

The question becomes: **Do I trust the data and adjust my worldview? Or do I trust my worldview and dismiss the data?**

**3. Value judgements that data cannot make**

A housing corporation can show that certain apartments are financially optimal. But "financially optimal" and "socially desirable" are different things. Data can tell you what is efficient. It cannot tell you what is just.

> "Er zit ruimte tussen wat je met data kan meten en waar je feitelijk op aan het sturen bent."
> (There is space between what you can measure with data and what you're actually steering towards.)

This gap will always exist. Data-driven policy is not data-determined policy.

---

## The colored balls metaphor

Imagine data points arriving like balls rolling in from the horizon:

- A stream of **grey balls** arrives. They match the norm. No one reacts. No interpretation needed.
- Then a **yellow ball** appears. It deviates from the norm. Now the entire mechanism of interpretation activates: Why is this one different? Is it an error? Is it real? What does it mean?

But here's the deeper question: **who decided that grey is the norm?**

The norm itself is a construct. It was built from historical data, which carries its own biases, paradigms, and gaps. The norm feels objective but it's a product of choices — which data to include, which comparisons to make, which external sources to trust.

---

## Implications for DataPraat

1. **Build the validation ladder into the product.** Don't just show data — show how confident we are in it. Each step of validation adds a confidence layer.

2. **Surface deviations, not just descriptions.** The tool should proactively flag: "This is 15% higher than comparable municipalities" or "This doesn't match the annual financial report."

3. **Make the norm explicit.** Always show what the comparison is. Never present a number without context. "€13.5M" means nothing. "€13.5M — which is 2.3x the average for municipalities of this size" means everything.

4. **Provide proof at every step.** Users must be able to download the underlying data, see the formula, and verify independently. Trust is built through transparency, not claims.

5. **Acknowledge the gap.** The tool should never pretend to have the full answer. It provides data-driven input for human decision-making. The policy document doesn't roll out of the tool — the tool supports the human who writes it.

6. **Financial data first.** Anchor everything in billing/invoicing data. Layer other data types with explicit quality indicators.

7. **Norms and validation are one process.** Don't build data validation and benchmarking as separate features. They're the same analysis running at different thresholds: large deviations = data quality issue, small deviations = actionable information.

---

*"De kern is de norm. Wie bepaalt de norm?"*
*(The core is the norm. Who determines the norm?)*

— Harm Nico Plomp, April 3rd 2026
