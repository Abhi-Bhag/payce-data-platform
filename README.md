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

<!-- Add your Lucidchart export here -->
![Data model](docs/data_model.png)

Each mart is a star schema. One fact table in the middle with the numbers, dimension tables around it with the labels, joined by surrogate keys.

---

## Dashboard

<!-- Add dashboard screenshots + link once published -->
![Dashboard](docs/dashboard.png)

Built in Databricks AI/BI, sitting directly on the Gold tables. It has three pages, one per mart, and each page has its own filter.

The thing that jumped out at me: fraud only appears in TRANSFER (0.77%) and CASH_OUT (0.18%) transactions. The other three types have none at all. Makes sense once you think about it, since the usual play is to move money out of an account you've taken over and then cash it out.

[Dashboard link](#) <!-- add once published -->

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

## Docs

- [Business Requirements Document](docs/Payce_BRD.md)
- Data model diagrams (Lucidchart)

---

<!-- add your contact -->
[LinkedIn](#) · [GitHub](#)
