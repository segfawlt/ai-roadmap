# Managed ML Services Cheatsheet

Side-by-side reference for managed AI/ML platform services across the major clouds. Use this to compare what each provider offers for model catalogs, MLOps tooling, compute options, and unique differentiators when choosing where to run a workload.

> **Note**
>
> Note
>
> This page currently covers Azure and GCP only — there is no AWS section yet, despite the rest of this roadmap being AWS-centric. If you need AWS managed AI services (Rekognition, Textract, Transcribe, SageMaker), the **What Is Deep Learning & MLOps?** track references them directly in its OCR & Document AI and Speech AI modules; this cheatsheet’s AWS gap is a known omission, not evidence those services don’t exist on the platform.

## Azure

### Azure AI Platform Overview

- **Unified experience:** Azure AI Studio vs. Azure Machine Learning studio.

- **Key pillars:** foundational models, machine learning, responsible AI.

### Core AI Development & Model Services

#### Azure AI Studio

- Hub for generative AI and copilot development.

- **Prompt Flow** for orchestrating LLM workflows.

- Evaluation and monitoring tools.

#### Model Infrastructure

- **Azure OpenAI Service:** GPT-4o, GPT-4 Turbo, embeddings (`text-embedding-3-large`), DALL-E 3, moderation, fine-tuning.

- **Azure AI Model Catalog (via AI Studio):** curated catalog of open-source and frontier models (Llama, Mistral, Phi, etc.) with integrated endpoints.

- **Custom model training:** bring your own model (BYOM) and framework to Azure ML.

#### Applied AI Services (Azure AI Services)

- **Cognitive Services:** Vision, Speech, Language, Decision — prebuilt APIs.

- **Azure AI Search:** **hybrid (keyword + vector) search** with semantic ranking as the backbone for production RAG systems.

- **Azure AI Agents:** service for building multi-step, reasoning agents with built-in tools and evaluation.

- **Azure AI Speech:** text-to-speech avatars, real-time speech-to-speech translation.

### Machine Learning Lifecycle (Azure Machine Learning)

- **Data & compute infrastructure:** notebooks, compute clusters/instances, serverless compute for ML.

- **Model training & experimentation:** automated ML (AutoML), MLflow integration.

- **MLOps:** model registry, pipelines, endpoint deployment, monitoring.

### Core Cloud Infrastructure for AI/ML

#### Compute Options

- **Virtual Machines:** NCasT4_v3, NC A100 v4, ND H100 v5/H200 series for cutting-edge GPUs. **Spot instances** for cost-effective training.

- **Serverless inference:** **Managed Online Endpoints** (with auto-scaling) and **Serverless Endpoints** (pay-per-execution) in Azure ML.

- **Kubernetes:** Azure Kubernetes Service (AKS) with GPU nodes for scalable inference.

#### Data & Storage

- **Blob Storage & Data Lake Storage Gen2:** primary data store for unstructured data.

- **Vector databases:** Azure Cache for Redis (with vector search) or integrated vector search in Azure AI Search.

- **Databases:** Cosmos DB (for operational data), Azure SQL Database.

### Responsible AI & Governance

- Toolbox for fairness, interpretability, and transparency.

- Content safety filters.

- Audit trails and compliance (GDPR, etc.).

### Architecture Patterns & Best Practices

- **Retrieval-Augmented Generation (RAG)** with Azure AI Search.

- Fine-tuning vs. prompt engineering.

- **Cost optimization:** spot VMs, managed endpoints, utilization monitoring.

### Integration & Ecosystem

- **GitHub Copilot & Azure:** developer integration.

- **Power Platform:** build AI-powered apps with Copilot Studio.

- **Microsoft Fabric:** unified analytics platform with AI synergy (OneLake, Synapse Data Engineering, Direct Lake).

### Azure References

- [Azure AI Studio Documentation](https://learn.microsoft.com/en-us/azure/ai-studio/)

- [Azure OpenAI Service Documentation](https://learn.microsoft.com/en-us/azure/ai-services/openai/)

- [Azure AI Search Documentation](https://learn.microsoft.com/en-us/azure/search/)

- [Azure Machine Learning Documentation](https://learn.microsoft.com/en-us/azure/machine-learning/)

- [Azure AI Agents Service](https://learn.microsoft.com/en-us/azure/ai-services/agents/)

- [Microsoft Fabric Documentation](https://learn.microsoft.com/en-us/fabric/)

## GCP

### Google Cloud AI/ML Overview

Google Cloud Platform (GCP) offers a powerful suite of AI/ML services and is particularly strong in AI research and development.

### Core Compute Services

#### Compute Engine

- Virtual machines with GPU/TPU support

- A2, N1 instances for ML workloads

- Preemptible VMs for cost savings

#### Cloud Functions

- Serverless compute

- Event-driven architecture

- Auto-scaling

#### Cloud Run

- Containerized applications

- Serverless container platform

- Pay-per-request pricing

### AI/ML Services

#### Vertex AI

- Unified ML platform (replaces the legacy AI Platform)

- Training with custom code and AutoML

- Model deployment and monitoring

- Feature Store

- Pipelines (Kubeflow)

#### Generative AI Studio

- Access to Gemini and PaLM model families

- Prompt engineering tools

- Fine-tuning capabilities

#### Cloud AI APIs

- Vision AI, Natural Language AI

- Speech-to-Text, Text-to-Speech

- Translation API

- Pre-trained models ready to use

### Storage Services

#### Cloud Storage

- Object storage (equivalent to AWS S3)

- Integration with ML services

- Multiple storage classes

#### Persistent Disk

- Block storage for VMs

- SSD and standard options

### Database Services

#### Cloud SQL

- Managed PostgreSQL, MySQL

- High availability

#### Firestore, Bigtable

- NoSQL options

- Real-time and scalable

### Unique GCP Advantages

#### TPUs (Tensor Processing Units)

- Google’s custom ML accelerators

- Optimized for TensorFlow

- Fast training for large models

#### BigQuery ML

- ML directly on data warehouse data

- SQL-based ML models

- Integrated analytics

### GCP References

- [Vertex AI Documentation](https://cloud.google.com/vertex-ai/docs)

- [Generative AI on Vertex AI](https://cloud.google.com/vertex-ai/generative-ai/docs)

- [Cloud Storage Documentation](https://cloud.google.com/storage/docs)

- [TPU Documentation](https://cloud.google.com/tpu/docs)

- [BigQuery ML Documentation](https://cloud.google.com/bigquery/docs/bqml-introduction)
