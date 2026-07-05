# Tag 3 — Integration, Orchestrierung & Observability

**Agenda-Bezug:** Integration von KI-Services in Microservice-Landschaften
(sync/async, Error-Handling, Fallbacks) · Orchestrierung von Funktionen & Agenten
(Service Mesh vs. Orchestrierungs-Frameworks).

## Lernziele
- KI-Services robust einbinden: Timeouts, Retries, Circuit Breaker, Fallbacks
- sync- vs. async-Schnittstellen (API Gateway, SQS/SNS) bewerten
- Agenten orchestrieren: Step Functions / LangGraph vs. Service Mesh
- Observability & Evaluation für KI-Komponenten definieren

## Übung
GraphRAG-Service aus Tag 2 als **Microservice** hinter API Gateway, orchestriert per
Step Functions/LangGraph, mit Guardrails, Tracing und Offline-Evaluation.

## Stack (geplant)
- FastAPI, API Gateway, SQS/SNS, AWS Step Functions, LangGraph
- Amazon Bedrock **Guardrails**
- **Observability:** OpenTelemetry + OpenLLMetry, Langfuse, CloudWatch/X-Ray
- **Evaluation:** Ragas, DeepEval

## Struktur (Platzhalter)
```
day3-integration-observability/
├── gateway/       # sync/async KI-Service-Schnittstelle
├── orchestration/ # Step Functions / LangGraph
└── observability/ # OTel, Langfuse, Eval-Suites
```
