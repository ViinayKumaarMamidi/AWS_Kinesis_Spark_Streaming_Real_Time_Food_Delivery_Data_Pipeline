# AWS_Kinesis_Spark_Streaming_Real_Time_Food_Delivery_Data_Pipeline
This repo contains details about how food delivery data pipeline is implemented in real time using Multiple AWS services and Spark Streaming, Thanks


Project End to End Documentation: 

https://deepwiki.com/ViinayKumaarMamidi/AWS_Kinesis_Spark_Streaming_Real_Time_Food_Delivery_Data_Pipeline


https://deepwiki.com/badge-maker?url=https%3A%2F%2Fdeepwiki.com%2FViinayKumaarMamidi%2FAWS_Kinesis_Spark_Streaming_Real_Time_Food_Delivery_Data_Pipeline


# Real-time Food Delivery Data Pipeline — Step-by-step Guide

This document explains, step by step, how to design, deploy, run, and monitor a real-time food delivery pipeline using AWS Kinesis and Apache Spark Streaming (PySpark). It assumes the repo contains sample producer and consumer code; the steps below show how to recreate and run the end-to-end pipeline, plus recommended configuration, testing, and troubleshooting.

Contents
- Overview
- Architecture (logical)
- Prerequisites
- AWS resource checklist
- Detailed step-by-step setup
  - 1) Create IAM roles & policies
  - 2) Create Kinesis Data Stream
  - 3) Create S3 buckets and optionally DynamoDB
  - 4) Producer (simulate delivery events)
  - 5) Spark Streaming consumer (Structured Streaming or DStreams)
  - 6) Launch Spark environment (EMR, EKS, or self-managed)
  - 7) Submit Spark job
  - 8) Checkpointing, offsets & exactly-once considerations
  - 9) Monitoring & logging
  - 10) Scaling & cost controls
- Testing & verification
- Security & best practices
- Troubleshooting tips
- Appendix
  - Sample event schema
  - Sample Python producer (boto3)
  - Sample PySpark Structured Streaming consumer
  - Helpful AWS CLI commands
---

Overview
This pipeline streams real-time food delivery events (e.g., new orders, driver pickup, delivery completed) into a Kinesis Data Stream. Spark Streaming consumes and processes events, writes aggregates / enriched data to persistent sinks (S3, DynamoDB, or a data warehouse), and produces real-time dashboards or downstream ML inferences.

Architecture (logical)
- Producer(s): mobile app simulator, backend, or Lambda producing JSON events into Kinesis Data Stream.
- Kinesis Data Stream: durable, sharded stream that buffers and delivers records.
- Spark Streaming (EMR / Spark cluster): consumes from Kinesis, processes events (parse, validate, enrich, aggregate), writes results/snapshots to:
  - S3 (raw and transformed)
  - DynamoDB or RDS (for low-latency lookups)
  - Optional: push aggregated metrics to CloudWatch or a visualization system.
- Monitoring: CloudWatch, Spark UI, EMR console, and alarms.

Prerequisites
- AWS account with permissions to create Kinesis, EMR/EC2, S3, IAM, CloudWatch, and optionally DynamoDB and Lambda.
- AWS CLI installed and configured (aws configure).
- Python 3.8+ and boto3 for producer scripts.
- PySpark / Spark 2.4+ (or 3.x) for consumer job. EMR recommended for managed Spark.
- (Optional) Terraform or CloudFormation for infra-as-code.

AWS resource checklist
- IAM role for Spark/EMR with Kinesis, S3, CloudWatch permissions.
- IAM user / role for running producer scripts (PutRecord/PutRecords).
- Kinesis Data Stream (name, shard count).
- S3 bucket(s) for raw events, checkpointing, and outputs.
- (Optional) DynamoDB table for fast lookups / state.
- EMR cluster, or Kubernetes/EKS cluster with Spark, or self-managed Spark cluster.

Detailed step-by-step setup

1) Create IAM roles & policies
- Create an IAM policy for Spark workers / EMR role, with at minimum:
  - kinesis:DescribeStream / GetShardIterator / GetRecords / SubscribeToShard / ListShards
  - s3:GetObject / PutObject / ListBucket (for outputs & checkpoints)
  - logs:CreateLogGroup / CreateLogStream / PutLogEvents (CloudWatch)
  - (Optional) dynamodb:PutItem / GetItem if writing state
- Create or use an EMR role with the above policy attached.
- Create an IAM user or role for producers with PutRecord/PutRecords allowed for the stream.

Example minimal policy (replace resource ARNs):
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "kinesis:DescribeStream",
        "kinesis:GetShardIterator",
        "kinesis:GetRecords",
        "kinesis:PutRecord",
        "kinesis:PutRecords",
        "kinesis:ListShards",
        "kinesis:SubscribeToShard"
      ],
      "Resource": "arn:aws:kinesis:<region>:<account-id>:stream/<stream-name>"
    },
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:PutObject",
        "s3:ListBucket"
      ],
      "Resource": [
        "arn:aws:s3:::<bucket-name>",
        "arn:aws:s3:::<bucket-name>/*"
      ]
    },
    {
      "Effect": "Allow",
      "Action": [
        "logs:CreateLogGroup",
        "logs:CreateLogStream",
        "logs:PutLogEvents"
      ],
      "Resource": "*"
    }
  ]
}
```

2) Create Kinesis Data Stream
- Decide shard count: shards determine throughput. Start with 1-4 shards for development.
- Create stream via Console or CLI:
```bash
aws kinesis create-stream --stream-name food-delivery-stream --shard-count 2 --region us-east-1
aws kinesis wait stream-exists --stream-name food-delivery-stream --region us-east-1
```

3) Create S3 buckets and optionally DynamoDB
- S3 for:
  - raw-events-bucket (raw JSON backup)
  - processed-output-bucket (aggregates/parquet)
  - checkpoint-bucket (for Spark Structured Streaming)
- Example:
```bash
aws s3 mb s3://food-delivery-raw-<your-id>
aws s3 mb s3://food-delivery-processed-<your-id>
```
- (Optional) Create DynamoDB table for fast state (e.g., delivery_status table).

4) Producer — simulate delivery events
- Implement a Python producer that uses boto3 to put JSON events into Kinesis with a partition key (e.g., order_id or driver_id).
- Keep data schema consistent (see Appendix).
- Example producer (full code in Appendix) uses PutRecord or PutRecords for batching.

Producer best practices:
- Batch writes with PutRecords for throughput/cost efficiency.
- Use a partition key that evenly distributes across shards (e.g., random suffix or hashed driver id).
- Retry on ProvisionedThroughputExceededException.

5) Spark Streaming consumer — design choices
You can consume using:
- Spark Structured Streaming (recommended if using modern Spark versions): readStream.format("kinesis") with a connector.
- Spark Streaming DStreams with spark-streaming-kinesis-asl (older API).

I’ll provide a sample Structured Streaming approach and an alternate DStream snippet in the appendix.

Key processing steps:
- Parse incoming JSON.
- Validate fields and schema.
- Enrich events (e.g., geo-lookups, driver status join).
- Aggregate metrics in sliding windows (e.g., avg delivery time last 5 minutes).
- Write outputs to S3 (parquet) or DynamoDB.

Important options:
- Use checkpointing (S3 path) for fault tolerance.
- Set trigger intervals appropriately based on latency requirements.
- Handle late data using watermarking (if aggregating).

6) Launch Spark environment (EMR example)
- Launch an EMR cluster with Spark enabled and attach the IAM role created earlier.
- For EMR 6.x (Spark 3.x):
  - Master: r5.xlarge (for dev use smaller/larger as needed)
  - Core: 2–4 nodes
  - Add bootstrap actions if you need extra Python libraries.
- Example AWS CLI to create EMR (simplified):
```bash
aws emr create-cluster \
  --name "FoodDeliverySparkCluster" \
  --release-label emr-6.6.0 \
  --applications Name=Spark \
  --ec2-attributes KeyName=<your-key> \
  --instance-type m5.xlarge \
  --instance-count 3 \
  --use-default-roles \
  --region us-east-1
```
- Alternatively: run Spark on EKS or self-managed cluster. Ensure connectors (kinesis-sql connector or spark-streaming-kinesis) are available.

7) Submit Spark job
- Package the PySpark job and submit via spark-submit or EMR step.
- Example spark-submit command (Structured Streaming connector example):
```bash
spark-submit \
  --packages org.apache.spark:spark-sql-kinesis_2.12:3.2.1,com.amazonaws:aws-java-sdk-kinesis:1.12.0 \
  --py-files dependencies.zip \
  process_kinesis_stream.py \
  --stream-name food-delivery-stream \
  --region us-east-1 \
  --checkpoint s3://food-delivery-processed-<your-id>/checkpoints/
```
- Configure memory/executors according to volume. On EMR, you can use Step API to run this as a job step.

8) Checkpointing, offsets & exactly-once
- Use Structured Streaming with S3 checkpoints for consistent recovery of offsets.
- For sinks that support idempotency (S3 Parquet append), structured streaming offers end-to-end semantics when configured properly.
- For external sinks like DynamoDB, implement idempotent writes (use conditional writes or dedupe by event id).
- Always set a stable checkpoint location in S3 and be cautious when deleting checkpoint directories (it will reset offsets).

9) Monitoring & logging
- CloudWatch: configure EMR/log groups to send logs (YARN, Spark driver/executor logs).
- Spark UI: monitor jobs, stages, and tasks.
- Kinesis Metrics (CloudWatch): ReadProvisionedThroughput, IncomingBytes, GetRecords.IteratorAgeMilliseconds (latency).
- Set CloudWatch alarms for:
  - Read/Write throttling
  - IteratorAge > threshold (lag)
  - High error rates or failed Spark steps

10) Scaling & cost controls
- Scale Kinesis shards horizontally for throughput; each shard supports 1 MB/s write and 2 MB/s read (numbers vary by AWS region — always confirm latest docs).
- EMR: use auto-scaling policies for Core/Task nodes.
- For dev/test use minimal resources; for production estimate required throughput and processing latency and provision accordingly.

Testing & verification

A) Smoke test
- Run the producer to send a known batch of events.
- Verify via CloudWatch or Kinesis GetRecords that records arrive.
- Check Spark application logs for successful consumption and parse results.
- Validate sink contents in S3 or DynamoDB.

B) End-to-end functional test
- Send events representing order lifecycle and confirm processed events produce correct aggregates (e.g., delivery time computations, status counts).
- Use a dataset of varying size and distribution to validate shard/partitioning strategy.

C) Performance test
- Ramp up producer throughput with PutRecords in batches; monitor GetRecords.IteratorAgeMilliseconds and provisioning alarms.
- Adjust shard count and Spark resources as needed.

Security & best practices
- Use least-privilege IAM policies.
- Encrypt data at rest (S3 SSE, Kinesis server-side encryption) and in transit (HTTPS endpoints).
- Use VPC endpoints (VPC interface endpoints for Kinesis and S3) to keep traffic internal to AWS.
- Rotate credentials and use IAM roles for EC2/EMR rather than long-lived keys.
- Protect S3 buckets with policies and enable logging & lifecycle rules (archive older raw events to Glacier if desired).

Troubleshooting tips
- If Kinesis throttling occurs:
  - Increase shard count, or reduce producer rate and batch size.
- If Spark job fails with serialization errors:
  - Ensure all custom libraries are packaged and available on executors; use --py-files or bootstrap steps.
- If offsets seem to rewind/duplicate:
  - Check checkpoint path permissions and existence. Deleting or modifying checkpoint can cause reprocessing.
- If records not visible in S3:
  - Check sink write paths and IAM permissions for Spark role.

Appendix

Sample event schema (JSON)
- Keep schema stable and versioned.
```json
{
  "event_type": "order_created",   // order_created, pickup, delivered, cancelled
  "order_id": "ORD123456",
  "driver_id": "DRV987",
  "restaurant_id": "RST12",
  "timestamp": "2026-01-04T12:34:56Z",
  "location": {"lat": 40.7128, "lon": -74.0060},
  "items": [{"item_id": "I1", "qty": 2, "price": 9.99}],
  "total_price": 19.98,
  "metadata": {}
}
```

Sample Python producer (boto3)
```python
# producer.py
import json
import time
import boto3
import uuid
import random

kinesis = boto3.client('kinesis', region_name='us-east-1')

STREAM_NAME = 'food-delivery-stream'

def generate_event():
    order_id = str(uuid.uuid4())
    event = {
        "event_type": random.choice(["order_created", "pickup", "delivered"]),
        "order_id": order_id,
        "driver_id": f"driver-{random.randint(1,50)}",
        "restaurant_id": f"rest-{random.randint(1,20)}",
        "timestamp": time.strftime("%Y-%m-%dT%H:%M:%SZ", time.gmtime()),
        "location": {"lat": 40.0 + random.random(), "lon": -74.0 + random.random()},
        "items": [{"item_id": "item-1", "qty": 1, "price": 12.5}],
        "total_price": 12.5,
        "metadata": {}
    }
    return event

def send_event(event):
    payload = json.dumps(event)
    partition_key = event["driver_id"]
    kinesis.put_record(StreamName=STREAM_NAME, Data=payload.encode('utf-8'), PartitionKey=partition_key)

if __name__ == "__main__":
    for _ in range(100):
        ev = generate_event()
        send_event(ev)
        time.sleep(0.1)
```

Sample PySpark Structured Streaming consumer (example)
```python
# process_kinesis_stream.py
import argparse
import json
from pyspark.sql import SparkSession
from pyspark.sql.functions import from_json, col, udf, window
from pyspark.sql.types import StructType, StructField, StringType, DoubleType, ArrayType, TimestampType, MapType

def main(args):
    spark = SparkSession.builder.appName("FoodDeliveryKinesisProcessor").getOrCreate()
    # Kinesis connector options may vary by connector version
    df = spark.readStream \
        .format("kinesis") \
        .option("streamName", args.stream_name) \
        .option("region", args.region) \
        .option("initialPosition", "TRIM_HORIZON") \
        .load()

    # df schema: key, value (binary), partitionKey, sequenceNumber, approxArrivalTimestamp
    schema = StructType([
        StructField("event_type", StringType(), True),
        StructField("order_id", StringType(), True),
        StructField("driver_id", StringType(), True),
        StructField("restaurant_id", StringType(), True),
        StructField("timestamp", StringType(), True),
        StructField("location", MapType(StringType(), DoubleType()), True),
        StructField("items", ArrayType(MapType(StringType(), StringType())), True),
        StructField("total_price", DoubleType(), True)
    ])

    json_df = df.selectExpr("CAST(data AS STRING) as json_str") \
                .select(from_json(col("json_str"), schema).alias("data")) \
                .select("data.*")

    # Example: compute counts per event type in a sliding window
    json_df = json_df.withColumn("ts", col("timestamp").cast("timestamp"))
    agg = json_df.groupBy(
        window(col("ts"), "5 minutes", "1 minute"),
        col("event_type")
    ).count()

    query = agg.writeStream \
        .outputMode("update") \
        .format("parquet") \
        .option("path", args.output_path) \
        .option("checkpointLocation", args.checkpoint) \
        .trigger(processingTime="60 seconds") \
        .start()

    query.awaitTermination()

if __name__ == "__main__":
    parser = argparse.ArgumentParser()
    parser.add_argument("--stream-name", required=True)
    parser.add_argument("--region", required=True)
    parser.add_argument("--output-path", required=True)
    parser.add_argument("--checkpoint", required=True)
    args = parser.parse_args()
    main(args)
```

Notes about the connector:
- The `format("kinesis")` connector availability depends on Spark and connector packages installed. On EMR, you may need to add the appropriate Maven packages with `--packages` or include the library when packaging.
- If the connector isn't available, use the Kinesis Client Library (KCL) integration (spark-streaming-kinesis-asl) or AWS-provided connectors.

Helpful AWS CLI commands
- List streams:
```bash
aws kinesis list-streams --region us-east-1
```
- Describe stream:
```bash
aws kinesis describe-stream --stream-name food-delivery-stream --region us-east-1
```
- PutRecords from CLI (useful for testing):
```bash
aws kinesis put-record --stream-name food-delivery-stream --partition-key key1 --data "$(echo -n '{"event_type":"order_created","order_id":"1"}' | base64 -w 0)"
```

Final tips
- Start small: a single shard + small EMR cluster to validate functional correctness.
- Add observability early: send logs and metrics to CloudWatch, add dashboards.
- Automate infra with Terraform or CloudFormation for repeatable deployments.
- Version your event schema and handle schema evolution carefully in Spark code.


Which of the above would you like me to produce next?
