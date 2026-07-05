# Tag 1 — Foundations, Event-Driven & Data-Centric

**Agenda-Bezug:** Rolle der SW-Architektur in KI-Systemen · EDA für KI-Workloads ·
datenzentrierte Architekturen · Persistenzkonzepte · Übung User-Profil-Komponente.

## Lernziele
- Architektur vs. Data Science vs. MLOps sauber abgrenzen
- Events/Topics/Streams für KI-Workloads modellieren
- Read/Write-Trennung (CQRS), Caching und Datenqualität begründen
- operational / analytics / vector store gezielt einsetzen

## Übung
Event-getriebene **User-Profil-Komponente** für KI-Personalisierung:
Events (Kafka/Redpanda lokal → EventBridge/Kinesis in AWS) → Profil-Aggregat
→ operational store (Postgres/DynamoDB) + Feature-/Vector-View.

## Stack (geplant)
- Kafka/Redpanda ↔ Amazon Kinesis/MSK, EventBridge
- Postgres/DynamoDB, Redis/ElastiCache
- FastAPI-Consumer/Producer, Pydantic-Events

## Struktur (Platzhalter)
```
day1-foundations-eda/
├── producers/     # Event-Erzeuger
├── consumers/     # Profil-Aggregator
└── stores/        # operational / vector view
```
