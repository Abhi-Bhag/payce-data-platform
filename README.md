# Payce — BNPL Data Platform

Payce is a made-up Buy Now Pay Later company. I built a full data pipeline for it on Databricks and AWS, going from raw files all the way to a set of data marts and a dashboard. The datasets are real financial data; the BNPL company is just the story I wrapped around them.

It processes about 96 million rows in total.

---

## Architecture

<!-- Add your architecture diagram here -->
![Architecture](docs/architecture.png)

```
Sources → Bronze (raw) → Silver (cleaned) → Gold (marts) → Dashboard
```

Raw files sit in S3. Bronze pulls them in untouched. Silver cleans them up. Gold turns them into star schemas that a dashboard can sit on top of. Standard Medallion setup.

---

## The idea

A BNPL business really comes down to a few moments in a customer's journey. Someone applies to split a purchase into instalments. They pay at checkout. Somewhere in there you have to catch fraud. And eventually the loan gets paid back, or it doesn't.

I found a dataset that fits each of those moments and built the pipeline around them. By the end you can answer three questions from the marts:

- Where is fraud happening?
- Who is likely to default?
- Who are the customers?

---

## Stack

- AWS S3 for storage
- Databricks (Serverless) for compute
- PySpark for the transformations
- Auto Loader for ingestion
- Delta Lake for the table format
- Unity Catalog for governance and the S3 connection
- Databricks Secrets + an IAM role for credentials
- Databricks AI/BI for the dashboard
- GitHub for version control

---

## Data sources

| Dataset | What I used it for | Rows |
|---|---|---|
| PaySim | Checkout payments and fraud flags | 6.3M |
| IEEE-CIS | Fraud signals | 590K |
| Home Credit | Credit applications and affordability | 307K |
| LendingClub | Loan repayment outcomes | 2.26M |

---

## The pipeline

### Bronze

Everything comes in exactly as it is and gets stored in Delta. Nothing gets changed here, so there's always a full copy of the original (~96M rows). I used Auto Loader for PaySim, IEEE-CIS and Home Credit. LendingClub had invalid characters in its column names so that one went in as a plain batch read instead. The S3 connection runs through Unity Catalog, which means no access keys are sitting inside the notebooks.

### Silver

This is where the data actually gets cleaned. Types get fixed, duplicates get dropped, and I cut each dataset down to the columns that are worth keeping.

- PaySim was already fairly clean. Cast the types, added a `balance_change_orig` column.
- IEEE-CIS came with 473 columns and I kept about 14. It also had junk rows where the fraud label was null or 0.5, which turned out to be test and submission files that got mixed in during ingestion. Filtered those out.
- Home Credit's Bronze table was a mess because 10 different files had been merged together. Instead of untangling that, I just went back to raw and re-read the one file I needed, `application_train.csv`. Dropped it from 122 columns to 13 and added `age_years`.
- LendingClub was the same story, so I re-read the accepted loans file from raw. Its `loan_status` column had 11 different values, one of which was somehow a date, so I collapsed them into a simple `loan_outcome` flag. Some of the numeric fields had bad values in them so I used `try_cast` instead of a normal cast.

### Gold

Three star schemas, 11 tables between them.

| Mart | Fact table | Dimensions | What it answers |
|---|---|---|---|
| Fraud | `fct_transactions` (6.3M) | type, date | Where is fraud? |
| Credit Risk | `fct_loans` (2.26M) | grade, purpose, term | Who defaults? |
| Customer | `fct_customer_profile` (307K) | segment, income band, education | Who are the customers? |

---

## Data model

<img width="2290" height="922" alt="image" src="https://github.com/user-attachments/assets/9bca87e1-e059-403b-8912-48f46df5eeba" />


Each mart is a star schema. One fact table in the middle with the numbers, dimension tables around it with the labels, joined by surrogate keys.

---

## Dashboard

Built in Databricks AI/BI, sitting directly on the Gold tables. Three pages, one per mart, and each page has its own filters. Every page follows the same layout: the health numbers up top (KPIs), then where the risk is concentrated (charts), then a table you can actually act on.

The screenshots below show all three pages. The dashboard was published in Databricks, but the live link sits behind a Databricks workspace login, so it won't open for external viewers — the screenshots are the best way to see it. (Live link, for reference: [Databricks dashboard](https://dbc-6766d642-c684.cloud.databricks.com/dashboardsv3/01f1717e29681a5f9b7279a64b75da5f/published?o=7474649544931900) — requires workspace access.)

### Fraud

<img width="2870" height="1290" alt="image" src="https://github.com/user-attachments/assets/077c79c4-0b8a-4742-97d5-c339f568bad3" />

The thing that jumped out at me: fraud only appears in TRANSFER (0.77%) and CASH_OUT (0.18%) transactions. The other three types have none at all. And when you break it down by hour, fraud spikes hard overnight (around 3–5am, hitting ~22%) and is near zero during the day. Makes sense once you think about it, since the usual play is to move money out of an account you've taken over and cash it out when no one's watching.

### Credit Risk

<img width="2866" height="1276" alt="image" src="https://github.com/user-attachments/assets/8322e111-1aea-4bfd-a3c4-c5dc56476d10" />

Default rate climbs cleanly with the loan grade: about 7% for low-risk loans, 17% for medium, and 31% for high-risk. That's the grade doing its job as a real risk signal. The exposure column is shown as an average per loan rather than a total, so a segment doesn't look scary just because it has more loans in it.

### Customer

<img width="2872" height="1284" alt="image" src="https://github.com/user-attachments/assets/92dd5905-f482-4eb8-ae31-73599264d5fd" />

This one has an honest finding: income barely predicts default. Splitting customers into income thirds gives default rates of roughly 8% / 9% / 7% — basically flat. So in this data, income on its own isn't telling you much about who defaults; the signal is somewhere else (credit history, existing debt). That's a real result, not a bug.

---

## A note on the money values

The Home Credit data records income and credit amounts in an anonymised foreign currency, so the raw numbers are large next to what you'd expect in pounds (average recorded income is around 150,000+). The pipeline preserves source values as they are, it doesn't quietly rescale them, because that's not the pipeline's job. For the dashboard I put a £ label on the amounts for readability, and I segment customers by relative income percentile rather than treating the absolute figures as real GBP. So the risk relationships hold up; the exact pound figures are illustrative. Same goes for the PaySim fraud amounts, which are large because the synthetic fraud transactions tend to drain whole balances.

---

## Things I ran into

- The Bronze tables ended up being external tables because I wrote the Delta files to S3 first and registered them in the catalog afterwards. It works fine, but if I did it again I'd set the catalog up first and write straight into managed tables.
- Home Credit and LendingClub both had messy Bronze tables from merged files. Re-reading the source file from raw was way less painful than trying to fix the merge downstream.
- IEEE-CIS's 473 columns were mostly anonymised machine learning features full of nulls. I only kept the ones that meant something for fraud.
- I kept Silver broad and left the real trimming for Gold, where each mart only holds what it needs.
- PaySim and IEEE-CIS don't share a key, so I didn't try to join them. They went into separate places instead of forcing something that wasn't there.

---

## Repo layout

```
payce-data-platform/
├── notebooks/
│   ├── 01_ingestion/     # Bronze
│   ├── 02_processing/    # Silver
│   ├── 03_gold/          # Gold marts
│   └── 04_orchestration/
├── src/
├── tests/
├── configs/
├── docs/                 # BRD, diagrams, dashboard images
└── README.md
```

---
[LinkedIn](#) · [GitHub](#)
