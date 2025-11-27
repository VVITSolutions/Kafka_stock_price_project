```markdown
# Real-Time Stock Market Data Pipeline – Setup Guide

This guide walks you through deploying a **modern Apache Kafka 4.1.1 (KRaft mode)** single-node cluster on AWS EC2 and connecting it to a Python-based producer/consumer that streams stock market data into **Amazon S3 → AWS Glue → Amazon Athena**.

> **No ZooKeeper** • **KRaft native** • **Publicly accessible broker** • **Fully working end-to-end pipeline**

---

### Prerequisites

- AWS account with permissions for EC2, S3, Glue, and Athena
- Basic familiarity with SSH and the AWS Console
- Python 3.9+ environment (local or Jupyter) with `kafka-python`, `s3fs`, `pandas`, `yfinance`

---

### Step 1: Launch an EC2 Instance

1. Launch an Amazon Linux 2 or Amazon Linux 2023 instance  
   → Recommended: `t3.small` (or `t3.micro` for testing)
2. Security Group rules (inbound):
   | Type         | Protocol | Port Range | Source       |
   |--------------|----------|------------|--------------|
   | SSH          | TCP      | 22         | Your IP      |
   | Custom TCP   | TCP      | 9092       | 0.0.0.0/0 (or your IP) |

3. Note down the **Public IPv4 address** (e.g., `3.88.12.26`)

---

### Step 2: Install Java & Download Kafka

```bash
# Connect to your instance
ssh -i your-key.pem ec2-user@<public-ip>

# Install Amazon Corretto 17 (OpenJDK)
sudo rpm --import https://yum.corretto.aws/corretto.key
sudo curl -L -o /etc/yum.repos.d/corretto.repo https://yum.corretto.aws/corretto.repo
sudo yum install -y java-17-amazon-corretto-devel

# Download and extract latest Apache Kafka 4.1.1 (Scala 2.13)
wget https://downloads.apache.org/kafka/4.1.1/kafka_2.13-4.1.1.tgz
tar -xzf kafka_2.13-4.1.1.tgz
cd kafka_2.13-4.1.1
```

---

### Step 3: Configure Kafka (KRaft Mode – No ZooKeeper)

```bash
# Generate a cluster UUID (run once)
CLUSTER_ID=$(bin/kafka-storage.sh random-uuid)
echo $CLUSTER_ID

# One-time format storage (uses standalone mode)
bin/kafka-storage.sh format --standalone -t $CLUSTER_ID -c config/server.properties
```

#### Update `config/server.properties` with your public IP

```bash
PUBLIC_IP="3.88.12.26"  # ← REPLACE WITH YOUR EC2 PUBLIC IP

cat << EOF >> config/server.properties

# === KRaft Single-Node + Public Access ===
node.id=1
process.roles=broker,controller

controller.quorum.voters=1@$PUBLIC_IP:9093
controller.quorum.bootstrap.servers=$PUBLIC_IP:9093

listeners=PLAINTEXT://:9092,CONTROLLER://:9093
advertised.listeners=PLAINTEXT://$PUBLIC_IP:9092
listener.security.protocol.map=CONTROLLER:PLAINTEXT,PLAINTEXT:PLAINTEXT
inter.broker.listener.name=PLAINTEXT

# Persistent logs (survives reboot)
log.dirs=/home/ec2-user/kafka-logs
EOF
```

```bash
# Create persistent log directory
mkdir -p ~/kafka-logs
```

---

### Step 4: Start Kafka (Survives SSH Disconnect)

```bash
# Start Kafka in background with small heap (perfect for t3.small/micro)
nohup bash -c 'export KAFKA_HEAP_OPTS="-Xms256m -Xmx512m"; \
    bin/kafka-server-start.sh config/server.properties' \
    > ~/kafka.log 2>&1 &

# Check it's running
tail -f ~/kafka.log | grep -i "Kafka Server started"
```

You should see:
```
[KafkaRaftServer nodeId=1] Kafka Server started
```

---

### Step 5: Test Kafka Locally & Remotely

#### From EC2:
```bash
bin/kafka-topics.sh --create --topic stock_ticks \
  --bootstrap-server $PUBLIC_IP:9092 --partitions 3 --replication-factor 1
```

#### From your laptop (optional – using Kafka CLI):
```bash
kafka-topics --create --topic stock_ticks \
  --bootstrap-server 3.88.12.26:9092
```

---

### Step 6: Run Producer & Consumer (Jupyter Notebooks)

1. Open `Kafka_Producer.ipynb`
   - Streams real-time or simulated stock data into `stock_ticks` topic
2. Open `Kafka_Consumer_S3.ipynb`
   - Consumes messages and writes each as a JSON file to S3:
     ```
     s3://kafka-stock-project-ravi/stock_market_0.json
     s3://kafka-stock-project-ravi/stock_market_1.json
     ...
     ```

> Ensure your local AWS credentials are configured:
> ```bash
> aws configure
> # Region: us-east-1 (or your bucket region)
> ```

---

### Step 7: Query Data with Amazon Athena

1. **Create S3 bucket** (if not exists):
   ```bash
   aws s3 mb s3://<bucket name> --region us-east-1
   ```
2. In AWS Console → **AWS Glue** → **Crawlers** → Create crawler
   - Data source: `s3://<bucket name>/`
   - Database: `stock_db` (create if needed)
   - Run crawler
3. Go to **Amazon Athena**
   - Select database `stock_db`
   - Query your live data:
     ```sql
     SELECT * FROM "stock_db"."kafka_stock_project_ravi" 
     ORDER BY timestamp DESC 
     LIMIT 10;
     ```


---

### Project Files in This Repo

- `Kafka_Producer.ipynb` – Live stock data → Kafka
- `Kafka_Consumer_S3.ipynb` – Kafka → S3 (JSON)
- `kafka_setup/` – Full EC2 + Kafka deployment scripts
- `architecture.png` – Visual diagram

---

**Built in November 2025**  
**Apache Kafka 4.1.1 • KRaft • AWS • Python**

Enjoy your production-grade streaming pipeline!
```
