# PuPrime Market Data Pipeline

## Business Problem

Forex brokers rely on accurate and up-to-date market data to power trading platforms, analytics dashboards, and risk management systems. Manual data collection is unreliable, and pipelines that silently fail or duplicate records can lead to incorrect business decisions and inaccurate reporting.

## Solution

An automated ETL pipeline that extracts live forex market data from the Twelve Data API, validates every response, computes trading indicators, and loads the results into PostgreSQL using idempotent upserts. The pipeline is designed to be rerun safely without creating duplicate records.

## Outcome

Processes three currency pairs (EUR/USD, GBP/USD, USD/JPY) per run. Network failures are retried automatically, malformed responses are rejected before loading, and duplicate records are prevented through PostgreSQL upserts.

---

## Architecture

```text
Twelve Data API
        │
        ▼
   Extraction
        │
        ▼
   Validation
        │
        ▼
 Transformation
        │
        ▼
 PostgreSQL (Neon)
        │
        ▼
Analytics & Reporting
```

---

## Data Source

The pipeline retrieves historical forex market data from the **Twelve Data API**.

Supported symbols:

- EUR/USD
- GBP/USD
- USD/JPY

---

## ETL Process

### 1. Extraction (`extractor.py`)

- Downloads the latest market data
- Retries failed requests automatically
- Stops immediately on invalid API credentials

### 2. Validation

The pipeline validates:

- Missing API responses
- Empty datasets
- Missing timestamps
- Invalid prices
- Negative values
- High prices lower than low prices

### 3. Transformation (`transformer.py`)

Calculates additional trading metrics.

| Metric | Description |
|---------|-------------|
| Daily Return | Percentage change in closing price |
| Price Range | High − Low |
| SMA 7 | Seven-day moving average |
| SMA 14 | Fourteen-day moving average |
| Volatility | Rolling standard deviation |

### 4. Loading (`loader.py`)

Loads data into PostgreSQL using idempotent upserts.

```sql
ON CONFLICT(symbol, datetime)
DO NOTHING;
```

This guarantees duplicate records are never inserted.

---

## Database Schema

```sql
CREATE TABLE forex_prices (
    id SERIAL PRIMARY KEY,
    symbol VARCHAR(10) NOT NULL,
    datetime TIMESTAMP NOT NULL,
    open NUMERIC(10,5),
    high NUMERIC(10,5),
    low NUMERIC(10,5),
    close NUMERIC(10,5),
    daily_return NUMERIC(10,6),
    price_range NUMERIC(10,6),
    sma_7 NUMERIC(10,5),
    sma_14 NUMERIC(10,5),
    volatility NUMERIC(10,6),
    created_at TIMESTAMP DEFAULT NOW(),
    UNIQUE(symbol, datetime)
);
```

---

## Reliability Features

| Feature | Implementation |
|---------|----------------|
| Retry Logic | Three automatic retry attempts |
| Validation | Rejects malformed API responses |
| Data Quality | Rejects invalid prices |
| Duplicate Prevention | PostgreSQL upsert |
| Structured Logging | Logs every pipeline stage |
| Idempotency | Safe repeated execution |

---

## Project Structure

```text
puprime-market-pipeline/
├── src/
│   ├── models/
│   └── pipeline/
├── tests/
├── .github/
├── README.md
├── requirements.txt
└── .env.example
```

---

## Setup

Install dependencies:

```bash
pip install -r requirements.txt
```

Create a `.env` file:

```env
DATABASE_URL=your_database_url
TWELVE_DATA_API_KEY=your_api_key
```

Run the pipeline:

```bash
python src/pipeline/pipeline.py
```

---

## Sample Output

```text
INFO - Extracting EUR/USD...
INFO - Validation passed.
INFO - Transformation complete.
INFO - Inserted 30 new rows.
INFO - Skipped 0 duplicate rows.
INFO - Pipeline completed successfully.
```

---

## Tech Stack

- Python 3.12
- PostgreSQL (Neon)
- Pandas
- Requests
- Psycopg2
- Python-dotenv
- Pytest
- Git
- GitHub

---

## Known Improvements

Future enhancements include:

- Airflow orchestration
- Automated scheduling
- Slack/email alerts
- Docker deployment
- Monitoring dashboard

---

## Business Value

This pipeline delivers clean, validated market data that can be safely consumed by downstream trading analytics and risk monitoring systems. By preventing duplicate records and validating every response before loading, it provides a dependable foundation for financial reporting and decision-making.

---

## Author

**Matthew James**

**Data Engineer**

GitHub: https://github.com/matthewoke94