# Architecting AI-Ready Applications — 3-Tage-Schulung

> **Status:** Planungsphase · dieses Repository ist Vorlage **und** Zentrum der Schulung.
> Alle Übungen, Referenz-Implementierungen und Slides-Beispiele leben hier.

Softwarearchitektur für KI-Systeme — praxisnah, mit **Python 3**, **uv** und **AWS**.
Als durchgängiges Fallbeispiel ("Running Case") dient die bereits im Repo vorhandene
**Financial-Crime-Domäne** (Sanktionsumgehung, Geldwäsche, Finanzbetrug) auf Basis von
**Neo4j** — siehe [`docs/domain-demo-neo4j-sanctions.md`](docs/domain-demo-neo4j-sanctions.md)
und [`notebooks/neo4j_sanctions_notebook.ipynb`](notebooks/neo4j_sanctions_notebook.ipynb).

Abweichung von der Agenda (bewusst): Statt klassischem **RAG** bauen wir direkt
**GraphRAG** — die Kombination aus Vektorsuche und Graph-Traversierung liefert bei
vernetzten Domänen (genau unser Sanktions-/UBO-Netzwerk) messbar bessere
Antwortqualität und liefert nachvollziehbare, belegbare Kontextpfade
("Explainability" statt Black-Box-Retrieval).

---

## 1. Ziel der Schulung

Die Teilnehmenden lernen, **KI-fähige Unternehmensanwendungen** zu **architektieren** —
also nicht Modelle zu trainieren (Data Science) oder Pipelines zu betreiben (MLOps),
sondern die **Software-Architektur** drumherum zu entwerfen: event-getrieben,
datenzentriert, für Modell-Integration optimiert und produktionsreif observierbar.

Nach der Schulung können die Teilnehmenden:

- Event-Driven-Architekturen (EDA) für KI-Workloads entwerfen
- **GraphRAG**-Muster für Unternehmensanwendungen integrieren
- datenzentrierte Microservices rund um KI-Modelle designen
- Observability für KI-Komponenten definieren
- Domänenmodelle mit KI-Komponenten verbinden

---

## 2. Agenda → Repository-Mapping

Quelle: *Architecting AI-Ready Applications – Softwarearchitektur für KI-Systeme*
([expertes.de](https://www.expertes.de/seminare/architecting-ai-ready-applications-softwarearchitektur-fuer-ki-systeme.html) /
[it-schulungen.com](https://www.it-schulungen.com/seminare/kuenstliche-intelligenz/ki-coding-ki-softwareentwicklung/architecting-ai-ready-applications-softwarearchitektur-fuer-ki-systeme.html)).
Die vollständige Agenda liegt in [`docs/agenda.md`](docs/agenda.md).

| Tag | Themenblock (Agenda) | Repo-Modul |
|-----|----------------------|-----------|
| **1** | Rolle der SW-Architektur in KI-Systemen (vs. Data Science / MLOps) · Event-Driven Architecture für KI-Workloads · datenzentrierte Architekturen (Datenflüsse, Datenqualität, Read/Write-Trennung, Caching) · Persistenzkonzepte (operational / analytics / vector stores) · Übung: event-getriebene User-Profil-Komponente für KI-Personalisierung | [`day1-foundations-eda/`](day1-foundations-eda/) |
| **2** | Architektur von GenAI-Anwendungen (wann RAG? wann reiner Modell-Call?) · Schnittstellen & Grenzen · **GraphRAG**-Patterns: Dokument-Pipeline, Embeddings, Indizes, Query-Flow | [`day2-graphrag/`](day2-graphrag/) |
| **3** | Integration von KI-Services in Microservice-Landschaften (sync/async, Error-Handling, Fallbacks) · Orchestrierung von KI-Funktionen und Agenten (Service Mesh vs. Orchestrierungs-Frameworks) · Observability & Evaluation | [`day3-integration-observability/`](day3-integration-observability/) |

---

## 3. Tech-Stack (Recherche-Ergebnis)

> Auswahlprinzip: **AWS-native zuerst** (Zielplattform der Schulung), mit
> **Open-Source-Äquivalent für lokal** (schneller, kostenfreier Loop via Docker/LocalStack).
> Jedes Konzept wird also zweimal gezeigt: lokal begreifbar → in der Cloud produktionsreif.

### 3.1 Sprache, Build & Basis
| Zweck | Tool |
|-------|------|
| Sprache | **Python 3.12+** |
| Paket-/Env-Management | **uv** (Astral) — schneller Resolver, `uv sync`, `uv run`, Lockfile |
| Datenmodelle / Validierung | **Pydantic v2**, `pydantic-settings` |
| API / Microservices | **FastAPI** + **Uvicorn** |
| Async-Tasks / Workflows | **AWS Step Functions** (Cloud), lokal **Prefect**/`asyncio` |
| Tests / Qualität | **pytest**, **ruff**, **mypy** |
| Container / lokal | **Docker** + **docker compose**, **LocalStack** (AWS-Emulation) |
| IaC | **AWS CDK** (Python) — Alternative: Terraform |

### 3.2 GenAI / GraphRAG (Python-Libraries)
| Zweck | Library |
|-------|---------|
| GraphRAG-Kern (Neo4j) | **`neo4j-graphrag`** (offizielles Neo4j-Paket: Retriever, KG-Builder, Pipelines) |
| MS-GraphRAG-Methodik auf Neo4j | **`ms-graphrag-neo4j`** (Entity-Extraction, Community Detection/Leiden, hierarchische Summaries) |
| Orchestrierung / Agenten | **LangGraph** + **LangChain** (`langchain-aws`, `langchain-neo4j`) |
| Alternative Index-/Query-Engine | **LlamaIndex** (`PropertyGraphIndex`) |
| Foundation Models / Embeddings | **Amazon Bedrock** via **`boto3`** / **`langchain-aws`** |
| Graph-Algorithmen | **Neo4j GDS** (Graph Data Science), **`networkx`** (lokal/klein) |
| Evaluation | **Ragas**, **DeepEval** — Faithfulness, Context-Precision/-Recall |

### 3.3 Stores, DBs & Streaming (OS-Tools ↔ AWS-native)
| Rolle | Open-Source / lokal | AWS-native |
|-------|---------------------|-----------|
| **Graph Store** (Kern der Domäne) | **Neo4j** (Community/Docker) | **Amazon Neptune** / **Neptune Analytics** (Graph + Vektor, openCypher) |
| **Vector Store** | **Qdrant** / **pgvector** / OpenSearch | **Amazon OpenSearch Serverless**, **Aurora PostgreSQL + pgvector**, **S3 Vectors**, Neptune Analytics |
| **Operational Store** (Read/Write, State) | **PostgreSQL** | **Amazon DynamoDB** / **Aurora** |
| **Analytics Store** | **DuckDB** | **Amazon Redshift** / **Athena + S3** |
| **Cache / Feature-Serving** | **Redis** | **Amazon ElastiCache** / **MemoryDB** |
| **Event-/Stream-Backbone** | **Apache Kafka** / **Redpanda** | **Amazon Kinesis** / **Amazon MSK**, **EventBridge**, **SQS/SNS** |
| **Object Store** (Dokumente, Rohdaten) | **MinIO** | **Amazon S3** |

### 3.4 AWS-native Services (Zielarchitektur)
| Kategorie | Service | Rolle in der Schulung |
|-----------|---------|-----------------------|
| GenAI-Plattform | **Amazon Bedrock** | Foundation Models, **Knowledge Bases** (managed RAG/GraphRAG), **Agents**, **Guardrails** |
| Graph + Vektor | **Amazon Neptune Analytics** | GraphRAG-Backend: Vektor-Ähnlichkeit + openCypher-Traversierung |
| Vector Store | **Amazon OpenSearch Serverless** | Default-Vektorindex für Produktions-RAG |
| Relational + Vektor | **Aurora PostgreSQL (pgvector)** | operational + vector im gleichen Store |
| Events | **Amazon EventBridge** | Event-Bus, Schema-Registry, Routing |
| Streaming | **Amazon Kinesis / MSK** | Streams, Topics für KI-Workloads |
| Async-Messaging | **Amazon SQS / SNS** | entkoppelte, fehlertolerante KI-Calls |
| Compute | **AWS Lambda**, **ECS/Fargate**, **EKS** | Serverless bis Container für Inferenz-Gateways |
| Orchestrierung | **AWS Step Functions** | Orchestrierung von KI-Funktionen & Agenten |
| API | **Amazon API Gateway** | sync/async Schnittstellen, Throttling |
| MLOps-Nachbarschaft | **Amazon SageMaker** | Abgrenzung Architektur ↔ MLOps (nur Einordnung) |
| Observability | **Amazon CloudWatch**, **AWS X-Ray** | Traces, Metriken, Logs für KI-Komponenten |
| Security/Config | **IAM**, **Secrets Manager**, **Parameter Store** | Zugriff, Credentials, Konfiguration |

### 3.5 Observability & Evaluation (KI-spezifisch)
| Zweck | Tool |
|-------|------|
| LLM-Tracing / Prompt-Management | **Langfuse** (self-hosted, Open Source) |
| Standard-Instrumentierung | **OpenTelemetry** + **OpenLLMetry** (GenAI-Semantik-Konventionen) |
| RAG-Debugging | **Arize Phoenix** |
| Offline-Evaluation | **Ragas**, **DeepEval** |
| Cloud-Telemetrie | **CloudWatch** / **X-Ray** (Export via OTel) |

---

## 4. Warum GraphRAG statt RAG? (Kurzbegründung)

- **Multi-Hop-Fragen**: Unsere Domäne ("Ist Firma A über Umwege mit einer sanktionierten
  Person verbunden?") ist per Definition ein Graph-Traversal-Problem — reines
  Vektor-Retrieval verliert die Beziehungsstruktur.
- **Bessere Generation-Qualität**: Graph liefert präzisen, verbundenen Kontext statt
  loser Chunks → weniger Halluzination, vollständigere Antworten bei "globalen" Fragen.
- **Explainability & Audit**: Der Retrieval-Pfad ist ein konkreter Graph-Pfad —
  in Compliance/Finanzkriminalität (unser Case) ist Nachvollziehbarkeit Pflicht.
- **Hybrid**: In Produktion routen wir einfache Faktfragen → Vektor-RAG,
  komplexe/analytische Fragen → GraphRAG (siehe Tag 2 & 3).

---

## 5. Vorgeschlagene Repository-Struktur

```
.
├── README.md                         ← dieses Planungsdokument
├── pyproject.toml / uv.lock          ← Dependencies (uv)
├── .python-version                   ← Python-Pin
│
├── docs/
│   ├── agenda.md                     ← vollständige Seminaragenda
│   ├── domain-demo-neo4j-sanctions.md← Fallbeispiel / Domänen-Write-up (Blog)
│   ├── architecture/                 ← C4-Diagramme, Referenzarchitektur
│   └── adr/                          ← Architecture Decision Records
│
├── infra/
│   ├── local/                        ← docker-compose (Neo4j, Kafka, Qdrant, LocalStack …)
│   └── aws/                          ← IaC (CDK/Terraform) für die Cloud-Ziele
│
├── src/aiready/                      ← geteilte Bibliothek (Config, Clients, Modelle)
│
├── day1-foundations-eda/             ← EDA, datenzentriert, Persistenz, User-Profil-Übung
├── day2-graphrag/                    ← Dokument-Pipeline, Embeddings, Indizes, GraphRAG-Query
├── day3-integration-observability/   ← Microservice-Integration, Orchestrierung, Observability
│
├── data/                            ← Beispieldaten (Offshore-Leaks, Sanktionslisten, synthetisch)
└── notebooks/                        ← Neo4j-Sanctions-Notebook & Explorationen
```

---

## 6. 3-Tage-Ablauf (Entwurf)

### Tag 1 — Foundations & Event-Driven / Data-Centric
- Einordnung: Architektur vs. Data Science vs. MLOps
- EDA-Grundlagen: Events, Topics, Streams (Kafka lokal → Kinesis/EventBridge in AWS)
- Datenzentrierte Architektur: Datenflüsse, Datenqualität, CQRS/Read-Write-Trennung, Caching
- Persistenzkonzepte: operational vs. analytics vs. vector store — wann was?
- **Übung:** event-getriebene *User-Profil-Komponente* für KI-Personalisierung

### Tag 2 — GenAI-Architektur & GraphRAG
- Wann RAG, wann GraphRAG, wann reiner Modell-Call? Schnittstellen & Grenzen
- Dokument-Pipeline: Ingestion → Chunking → Entity-Extraction → Knowledge Graph
- Embeddings & Indizes: Vektorindex + Graph, Hybrid-Retrieval
- **Übung:** GraphRAG über das Sanktions-/UBO-Netzwerk (Neo4j lokal ↔ Bedrock KB + Neptune Analytics)

### Tag 3 — Integration, Orchestrierung & Observability
- KI-Services in Microservice-Landschaften: sync/async, Timeouts, Retries, Fallbacks
- Orchestrierung von Agenten: Step Functions / LangGraph vs. Service Mesh
- Guardrails & Sicherheit
- **Observability:** Tracing (OTel/Langfuse), Evaluation (Ragas/DeepEval), Kostenkontrolle

---

## 7. Setup (Kurz)

```bash
# Repo klonen & Umgebung
uv sync                       # installiert Dependencies aus uv.lock
uv run jupyter lab            # Notebook-Umgebung

# Lokale Infrastruktur (geplant, siehe infra/local/)
docker compose -f infra/local/docker-compose.yml up -d
```

> AWS-Zugang (Bedrock, Neptune, OpenSearch …) wird zu Schulungsbeginn bereitgestellt;
> Region/Modelle und Guardrails werden in `infra/aws/` bzw. `.env` konfiguriert.

---

## 8. Voraussetzungen der Teilnehmenden

- Solide Python-Kenntnisse, Grundverständnis von Microservices/REST
- Grundlagen Cloud (idealerweise AWS)
- Kein ML-Training-Wissen nötig — Fokus ist **Architektur**, nicht Modellierung

---

## 9. Nächste Schritte (offene Punkte)

- [ ] `pyproject.toml` um GenAI/GraphRAG/AWS-Dependencies erweitern (Tag 2/3)
- [ ] `infra/local/docker-compose.yml` (Neo4j, Kafka/Redpanda, Qdrant, LocalStack) ausarbeiten
- [ ] Referenzarchitektur-Diagramm (C4) in `docs/architecture/`
- [ ] Beispieldaten-Loader in `data/` (Offshore-Leaks / OpenSanctions)
- [ ] Übungs-Skelette je Tag mit Lösungsvorschlägen
- [ ] Entscheidung Vektor-Backend: OpenSearch Serverless vs. Neptune Analytics vs. pgvector (ADR)

---

*Domäne, Datenmodell und Cypher-Beispiele: siehe [`docs/domain-demo-neo4j-sanctions.md`](docs/domain-demo-neo4j-sanctions.md)
und das [GDS/APOC-Cheatsheet](gds_apoc_cheatsheet.md).*
