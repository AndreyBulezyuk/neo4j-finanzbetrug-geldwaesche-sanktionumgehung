# Seminaragenda — Architecting AI-Ready Applications

**Softwarearchitektur für KI-Systeme** · 3 Tage
Quelle: [expertes.de](https://www.expertes.de/seminare/architecting-ai-ready-applications-softwarearchitektur-fuer-ki-systeme.html)
· [it-schulungen.com](https://www.it-schulungen.com/seminare/kuenstliche-intelligenz/ki-coding-ki-softwareentwicklung/architecting-ai-ready-applications-softwarearchitektur-fuer-ki-systeme.html)

## Zielsetzung
Architekturen für KI-fähige Unternehmensanwendungen entwerfen, die **event-getrieben**,
**datenzentriert** und für den **Einsatz von KI-Modellen optimiert** sind.

## Inhalte (laut Anbieter)

1. **Rolle der Softwarearchitektur in KI-Systemen** — Abgrenzung zu Data Science und MLOps
2. **Event-Driven Architecture (EDA) für KI-Workloads** — Events, Topics, Streams, typische Use Cases
3. **Datenzentrierte Architekturen** — Datenflüsse, Anforderungen an Datenqualität,
   Read/Write-Trennung, Caching
4. **Persistenzkonzepte für KI-fähige Anwendungen** — operational stores, analytics stores,
   vector storage
5. **Praxisbeispiel** — Design einer event-getriebenen User-Profil-Komponente für KI-Personalisierung
6. **Architektur von GenAI-Anwendungen** — wann RAG? wann reine Modell-Calls? Schnittstellen & Grenzen
7. **RAG-Patterns im Überblick** — Dokument-Pipeline, Embeddings, Indizes, Query-Flow
   *(→ in dieser Schulung als **GraphRAG** umgesetzt)*
8. **Integration von KI-Services in Microservice-Landschaften** — sync/async-Schnittstellen,
   Error-Handling, Fallbacks; Orchestrierung von KI-Funktionen und Agenten:
   Service Mesh vs. Orchestrierungs-Frameworks

## Lernergebnisse
Nach dem Seminar können die Teilnehmenden:
- event-getriebene Architekturen für KI-Workloads entwerfen
- RAG- (hier: GraphRAG-)Patterns für Unternehmensanwendungen integrieren
- datenzentrierte Microservices rund um KI-Modelle designen
- Observability für KI-Komponenten definieren
- Domänenmodelle mit KI-Komponenten verbinden

## Zielgruppe & Voraussetzungen
- Softwarearchitekt:innen, Senior-Entwickler:innen, Tech Leads
- Solide Programmier- und Microservice-Grundlagen; Cloud-Grundlagen (AWS von Vorteil)
- Kein ML-/Modelltraining-Vorwissen nötig — Fokus ist **Architektur**

## Bewusste Abweichung in diesem Repo
Statt klassischem RAG wird durchgängig **GraphRAG** unterrichtet — bessere Generation-Qualität
und Nachvollziehbarkeit bei vernetzten Domänen. Begründung siehe [README](../README.md#4-warum-graphrag-statt-rag-kurzbegründung).
