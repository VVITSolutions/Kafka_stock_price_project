```markdown
# Real-Time Stock Market Data Pipeline with Apache Kafka (KRaft), Python, and AWS

![Project Architecture](architecture.png)
> A fully functional, modern real-time stock market data pipeline using **Apache Kafka 4.1.1 in KRaft mode** (no ZooKeeper), Python, boto3, and AWS cloud services.

## Features

- Real-time ingestion of stock market data using Python + `yfinance` / `alpha_vantage`
- High-performance **Apache Kafka 4.1.1** cluster running in **KRaft mode** (ZooKeeper-free, 2025 standard)
- Self-hosted Kafka broker on **Amazon EC2** with public endpoint
- Kafka Producer streaming live/simulated stock ticks
- Kafka Consumer writing data directly to **Amazon S3** as JSON files
- Data discoverable via **AWS Glue Data Catalog**
- Queryable instantly using **Amazon Athena** (SQL on S3)
- Fully serverless analytics layer — no EMR, no Redshift needed

## Tech Stack

| Layer              | Technology                              |
|--------------------|-----------------------------------------|
| Ingestion          | Python (`kafka-python`, `yfinance`)     |
| Streaming          | Apache Kafka 4.1.1 (KRaft mode)         |
| Compute (Broker)   | Amazon EC2 (t3.micro / t3.small)        |
| Storage            | Amazon S3                               |
| Catalog            | AWS Glue Crawler + Data Catalog         |
| Query Engine       | Amazon Athena (serverless SQL)          |
| File Handling      | `s3fs` + native JSON                    |

## Project Structure

```
.
├── producer.ipynb              ← Live stock data producer (real-time or simulation)
├── consumer_s3.ipynb           ← Kafka consumer → S3 (your working version!)
├── kafka_setup/                ← Scripts & config for EC2 Kafka setup
├── data/                       ← Sample output (JSON files in S3 structure)
├── architecture.png           ← Diagram (you're seeing it above)
└── README.md                   ← This file
```

## How It Works

1. **Producer** fetches stock data every few seconds and publishes to Kafka topic `stock_ticks`
2. **Kafka broker** (running on EC2) receives and buffers messages reliably
3. **Consumer** reads from Kafka and writes each message as a JSON file to:
   ```
   s3://kafka-stock-project-ravi/stock_market_0.json
   s3://kafka-stock-project-ravi/stock_market_1.json
   ...
   ```
4. **AWS Glue Crawler** runs (manually or scheduled) → infers schema → populates Glue Data Catalog
5. Query your live stock data instantly using **Athena SQL**:
   ```sql
   SELECT symbol, price, volume, timestamp
   FROM "stock_db"."kafka_stock_data"
   WHERE date = '2025-11-27'
   ORDER BY timestamp DESC
   LIMIT 10;
   ```

## Setup Instructions

### 1. Launch Kafka on EC2 (One-time)
See `kafka_setup/` folder for full guide:
- Uses latest **Apache Kafka 4.1.1** with **KRaft** (no ZooKeeper!)
- Public IP accessible broker
- Persistent logs (optional)
- Auto-restart script included

### 2. Run Producer & Consumer
Open the Jupyter notebooks:
```bash
jupyter lab Kafka_producer.ipynb
jupyter lab Kafka_consumer.ipynb
```

Make sure:
- Your AWS credentials are configured (`aws configure`)
- Default region is set to `us-east-1`
- S3 bucket `kafka-stock-project-ravi` exists

### 3. Query with Athena
1. Go to AWS Console → Athena
2. Run Glue Crawler on `s3://kafka-stock-project-ravi/`
3. Query your data in seconds!

## Future Improvements (Planned)

- [ ] Write **Parquet** files with partitioning (`year=`, `month=`, `day=`) for 10x faster queries
- [ ] Use **Kafka Connect + S3 Sink Connector** (fully managed, zero code)
- [ ] Add **Docker + ECS/Fargate** deployment (no EC2 management)
- [ ] Real-time dashboard with **Streamlit / Quicksight**

## Why This Project Rocks

- Uses **2025-best-practice Kafka** (KRaft, no legacy ZooKeeper)
- Shows real-world pattern: **Kafka → S3 → Glue → Athena**
- 100% open source + cloud-native
- Easily extensible to 1000s of stocks or any streaming use case

---

**Built with help from Grok (xAI)** – because even senior engineers Google things

Completed by  **Ravi** • November 2025

