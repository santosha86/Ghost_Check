<!--
Fictional example CV shown to forkers of GhostCheck.

This represents the canonical markdown shape that produces the sharpest audits
(H1 for name, H2 for sections, H3 for role blocks, bulleted achievements). You
don't need to reformat your own CV to match — MarkItDown parses PDF/DOCX into
markdown automatically when you run `/ghostcheck audit`. This file is a
reference for the best-case input and a starting template for anyone building
a CV from scratch.

The person "Rohan Mehta" does not exist. All companies, clients, and metrics
are fictional or genericized. Any resemblance to real organizations is
incidental.
-->

# Rohan Mehta

**AI Architect & Manager | Generative AI, LLMs & Enterprise AI Systems**

Riyadh, Saudi Arabia | +966 5XX XXX XXXX | rohan.mehta.ai@example.com | linkedin.com/in/rohanmehta-ai/

---

## Professional Summary

AI Architect, Manager and Researcher in Emerging Technologies with 16 years of experience leading the design and delivery of enterprise AI systems for a national oil & gas operator, a Gulf petrochemical major, a national telecom operator, and a national utilities operator across Oil & Gas, Petrochemical, Utilities, and Telecom sectors in KSA. Proven track record architecting end-to-end GenAI solutions, including Retrieval-Augmented Generation (RAG) pipelines, multi-agent orchestration frameworks, embedding and vectorization pipelines, and production LLM inference systems on secure Kubernetes-native platforms. Experienced in providing technical direction to AI engineering squads, defining architecture standards, and aligning AI capability roadmaps with enterprise business objectives. Pursuing a Doctorate in Emerging Technologies with Generative AI specialization at a US-based university.

---

## Core Competencies

- **GenAI Architecture & LLM Engineering:** Large Language Models (LLMs), Retrieval-Augmented Generation (RAG), RAFT, Multi-Agent Orchestration (LangGraph, LangChain, CrewAI), Embedding Pipelines, Vector Databases (Milvus, FAISS), Multimodal AI (STT/TTS)
- **AI Solution Architecture:** End-to-End AI System Design, Data Ingestion to Inference Pipeline Architecture, Model Orchestration, Enterprise System Integration, Modular AI Service Composition, AI System Modernization
- **MLOps & Model Lifecycle:** Model Versioning, Deployment, Monitoring & Retraining (MLflow, Kubeflow), CI/CD for ML, Canary Deployments, Production AI Environments, Cloud-Native ML Platforms
- **AI Management & Leadership:** Technical Direction & AI Squad Leadership, Architecture Governance, Stakeholder Management, AI Roadmap Development, Framework & Tool Selection, Compliance-Aligned AI Design
- **Python & ML Frameworks:** Python (FastAPI, PyTorch, Scikit-learn, Pandas), NL-to-SQL, NLP Pipelines, Feature Engineering, Real-Time Scoring APIs
- **Data Engineering:** Apache Spark, Kafka, ETL/ELT Pipelines, Real-Time Streaming, Data Warehousing
- **Security & Governance:** OAuth 2.0, JWT, PII Masking, SDAIA/NDMO Compliance, RBAC, Air-gapped Enterprise Deployments

---

## Executive AI Experience

### Principal AI Solution Architect | Gulf AI Advisory | 2024 – Present

*KSA. Client: a national utilities operator.*

- Defined the end-to-end AI architecture for the client's enterprise Dispatch Intelligence platform, selecting the multi-agent framework (LangGraph + LangChain), orchestration patterns, embedding pipeline design, and on-premises LLM deployment strategy using Ollama, ensuring the solution met enterprise security and offline operability requirements.
- Architected a modular, agent-based AI system with composable service design, where specialized agents (SQL, RAG, CSV Analytics, Math) operate independently and are orchestrated via a routing layer, an approach architecturally aligned with Model Context Protocol (MCP) principles for interoperable AI service composition.
- Led the architecture and delivery of an AI-powered Online Interview platform integrating multimodal capabilities: Speech-to-Text (STT) and Text-to-Speech (TTS) for live voice interaction, semantic CV-to-JD matching using LLM-based embedding comparison, and an automated multi-criteria scoring engine governed by configurable HR rubrics.
- Provided technical direction to the development team on AI architecture standards, framework selection rationale, API design patterns, and deployment strategies, ensuring all deliverables met production-readiness, security, and performance targets.

### Principal AI Architect | Mideast Tech Partners | 2021 – 2024

*KSA. Clients: a Gulf petrochemical major, a national oil & gas operator.*

- Owned the enterprise GenAI architecture for the petrochemical major's AI programme, defining the reference architecture for RAG-based document intelligence, embedding pipeline design (BERT/FAISS), vector storage strategy (Milvus), LLM inference layer (Llama3 on OpenShift AI), and integration with existing enterprise procurement and contract management workflows.
- Led the architecture decision to adopt RAFT (Retrieval Augmented Fine-Tuning) over standard RAG for domain-specific petrochemical documentation, directed the Kubeflow-based fine-tuning pipeline design, and governed model versioning and deployment standards, achieving 40% higher accuracy over baseline RAG.
- Directed the design of an NL-to-SQL Co-Pilot using LangChain, defining the schema mapping strategy, query validation governance, and data access control patterns that enabled non-technical procurement users to safely query live operational databases.
- Established and governed a standardized MLOps framework across all AI workloads for the petrochemical major, defining CI/CD for ML standards, MLflow experiment tracking policies, Kubeflow orchestration patterns, canary deployment strategies, and Prometheus/Grafana monitoring requirements.
- Architected an NLP-based supply chain risk intelligence platform, defining the real-time data ingestion architecture (Spark and Kafka), ML model deployment topology as Python microservices on Kubernetes, and observability design, improving risk identification accuracy by 40%.
- Designed the AI architecture for the oil & gas operator's Finance forecasting solution, defining data pipeline specifications, feature engineering standards, model lifecycle governance via MLflow, and automated retraining triggers based on data drift thresholds.
- Defined enterprise AI security architecture across all workloads, including OAuth 2.0 and JWT authentication patterns, PII masking strategies, RBAC policies, and ELK Stack audit logging design in air-gapped environments, ensuring full SDAIA/NDMO compliance.

### Senior Data Scientist & Analytics Manager | International Tech Services | 2015 – 2021

*Riyadh, KSA. Client: a national telecom operator.*

- Managed and provided technical direction to the telecom operator's analytics AI squad, governing model development standards, experiment tracking practices, and production deployment governance across 12 telecom sectors.
- Architected the customer churn prediction platform, defining the end-to-end pipeline from BSS/CRM data ingestion through feature engineering, XGBoost model training, and real-time scoring API deployment, retaining over 50% of at-risk subscribers.
- Designed a real-time fraud detection system processing 350+ streaming attributes via Kafka and Spark, defining the streaming architecture, ML model integration patterns, and BSS alert workflow connections, reducing fraud by 30%.
- Led delivery of centralized P&L dashboards unifying financial data from 12 sectors, cutting manual reporting by 60%, and standardized MLflow-based experiment tracking for reproducible, governed model delivery across all divisions.

### Data Analyst & Functional Consultant | International Tech Services | 2008 – 2015

*Bahrain & India. Clients: a national telecom operator (KSA), an Indian telecom operator.*

- Delivered data architecture and functional consulting for large-scale telecom billing systems, building ETL pipelines, BSS data warehousing frameworks, and integration layers across billing, CRM, and network data sources.
- Contributed to revenue assurance frameworks and operational reporting for telecom networks across GCC and India, establishing foundational expertise in enterprise BSS/CRM data systems.

---

## Key Architectural Projects & Use Cases

### Multi-Agent AI Enterprise Dispatch Assistant — a national utilities operator

- **Objective:** Define and deliver an enterprise-grade AI architecture enabling field operators to query operational databases, vehicle analytics, and regulatory documents in natural language, accessible via mobile in remote power plant environments.
- **Architecture:** LangGraph + LangChain multi-agent orchestration with Ollama LLM (on-premises). Modular agent composition: SQL Agent (NL-to-SQL on SQLite), PDF Agent (RAG over Grid Code documents), CSV Agent (dwell time analytics), Math Agent. Embedding pipeline for document vectorization and semantic retrieval. Python FastAPI backend with SSE streaming. Flutter mobile app (Android/iOS) with bilingual Arabic/English RTL, biometric authentication, offline caching, and interactive data visualizations.
- **Business Impact:** Query success rate greater than 90%, average response time under 5 seconds, app binary under 50 MB, delivered full AI-powered operational intelligence to field teams with offline capability.

### AI-Powered Online Interview and Multimodal CV Matching Platform

- **Objective:** Architect an end-to-end AI recruitment platform integrating LLM-based semantic CV-to-JD matching, dynamic interview question generation, live multimodal voice interaction, and automated candidate scoring.
- **Architecture:** Multi-agent orchestration (CrewAI / LangGraph) for semantic JD-CV comparison using embedding-based similarity scoring. Dynamic interview question generation from JD analysis. Multimodal integration: Whisper-based Speech-to-Text (STT) for candidate voice capture and Text-to-Speech (TTS) for AI interviewer responses, enabling fully voice-driven interview sessions. Automated scoring engine with configurable HR evaluation rubrics. JWT / OAuth 2.0 role-based access control.
- **Business Impact:** Reduced recruiter screening effort significantly, improved candidate-to-role alignment accuracy through semantic matching, and delivered consistent, bias-reduced evaluation across all sessions.

### Enterprise GenAI Procurement Intelligence — a Gulf petrochemical major

- **Objective:** Architect a production LLM-powered document intelligence system for vendor contract Q&A on a secure, air-gapped enterprise platform, establishing a reusable GenAI reference architecture for the client.
- **Architecture:** End-to-end RAG pipeline on OpenShift AI (Kubernetes-native): document ingestion, BERT/FAISS embedding pipeline, Milvus vector database, Llama3 inference layer. RAFT fine-tuning pipeline on Kubeflow for domain-specific accuracy. MLflow for model versioning and lifecycle governance. ELK Stack for audit trail, data lineage, and thought chain observability. Full air-gap compliance with enterprise security controls.
- **Business Impact:** Reduced query response times by 50%, improved answer accuracy by 35% over baseline, established a reusable GenAI reference architecture rolled out across multiple business units.

### Supply Chain Risk Intelligence Platform — a Gulf petrochemical major

- **Objective:** Define the AI architecture for real-time predictive risk detection across the petrochemical supply chain, enabling proactive procurement and operational decisions.
- **Architecture:** Real-time data ingestion architecture using Apache Spark and Kafka from multiple operational sources. NLP-based risk classification models deployed as Python microservices on Kubernetes. Prometheus and Grafana observability layer for continuous model health and inference monitoring.
- **Business Impact:** Improved supply chain risk identification accuracy by 40%, enabling data-driven strategic decisions and measurable cost avoidance across procurement operations.

---

## Technical Toolkit

- **AI / LLM:** LangChain, LangGraph, Ollama, Llama3, FAISS, Milvus, BERT, Whisper STT/TTS
- **Python & ML:** Python, FastAPI, PyTorch, Scikit-learn, XGBoost, Pandas
- **MLOps & Platforms:** Docker, Kubernetes, OpenShift AI, MLflow, Kubeflow, CI/CD for ML
- **Data Engineering:** Apache Spark, Kafka, SQLite, ETL/ELT Pipelines
- **Security & Observability:** OAuth 2.0, JWT, Prometheus, Grafana, ELK Stack
- **Mobile & Frontend:** Flutter, React, TypeScript

---

## Education & Certifications

- **Pursuing:** Doctorate in Emerging Technologies, Generative AI Specialization | a US-based university
- **Post Graduate Program (PGP) in Artificial Intelligence & Machine Learning** | a top-tier Indian engineering institute | 2019
- **Master of Computer Applications (MCA)** | 2008
- **Pursuing:** Certified Kubernetes Application Developer (CKAD)
- **Pursuing:** AWS Solutions Architect Associate
