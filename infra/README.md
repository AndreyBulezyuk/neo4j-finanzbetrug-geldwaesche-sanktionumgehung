# Infrastruktur

Zwei Ebenen, damit jedes Konzept lokal begreifbar **und** in AWS produktionsreif gezeigt wird.

## `local/` — Docker-basierte Lernumgebung
Geplanter `docker-compose.yml`:
- **Neo4j** (Graph Store, Running Case)
- **Redpanda/Kafka** (Event-Backbone)
- **Qdrant** oder **Postgres+pgvector** (Vector Store)
- **Redis** (Cache)
- **LocalStack** (S3, SQS/SNS, EventBridge, Lambda lokal emuliert)

## `aws/` — Infrastructure as Code
Geplant mit **AWS CDK (Python)** — Ziel-Services:
Bedrock (Knowledge Bases, Guardrails), Neptune Analytics, OpenSearch Serverless,
EventBridge/Kinesis, SQS/SNS, Lambda/Fargate, Step Functions, API Gateway,
CloudWatch/X-Ray, IAM/Secrets Manager.
