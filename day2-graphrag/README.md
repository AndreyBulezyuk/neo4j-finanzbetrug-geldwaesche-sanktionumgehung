# Tag 2 — GenAI-Architektur & GraphRAG

**Agenda-Bezug:** Architektur von GenAI-Anwendungen · Schnittstellen & Grenzen ·
RAG-Patterns — hier umgesetzt als **GraphRAG**.

## Lernziele
- Entscheiden: reiner Modell-Call vs. RAG vs. GraphRAG
- Dokument-Pipeline bauen: Ingestion → Chunking → Entity-/Relation-Extraction → Knowledge Graph
- Embeddings & Indizes: Vektorindex + Graph, Hybrid-Retrieval
- Query-Flow und Prompt-Assembly für GraphRAG entwerfen

## Übung
**GraphRAG über das Sanktions-/UBO-Netzwerk** (Running Case):
- lokal: `neo4j-graphrag` + Neo4j + Bedrock-Embeddings/-Chat
- Cloud: Amazon Bedrock **Knowledge Bases** + **Neptune Analytics** (Graph + Vektor)
- Vergleich: reines Vektor-RAG vs. GraphRAG (Multi-Hop-Fragen)

## Stack (geplant)
- `neo4j-graphrag`, `ms-graphrag-neo4j`, `langchain-neo4j`, LlamaIndex `PropertyGraphIndex`
- Amazon Bedrock (FMs, Embeddings, Knowledge Bases), Neptune Analytics, OpenSearch Serverless

## Struktur (Platzhalter)
```
day2-graphrag/
├── ingestion/     # Dokument-Pipeline, Entity-Extraction
├── index/         # Vektor- + Graph-Index-Aufbau
└── query/         # GraphRAG-Retriever + Generation
```
