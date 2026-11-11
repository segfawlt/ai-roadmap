# Glossary

A comprehensive reference of key terms used throughout the AI Vanguard courses. Terms are grouped by domain for easier navigation.

------------------------------------------------------------------------

## AI & Machine Learning

LLM (Large Language Model)  
A neural network trained on massive text corpora that can generate, summarize, and reason about natural language. Modern frontier LLMs such as Claude Opus 4.6, Claude Sonnet 4.6, GPT-5.2, and Gemini 3 use the Transformer architecture and contain hundreds of billions of parameters. They power chatbots, coding agents, and most generative-AI applications.

Transformer  
The deep-learning architecture introduced in the 2017 paper *Attention Is All You Need*. Transformers rely on self-attention to process input tokens in parallel, enabling efficient training on large datasets. They form the backbone of virtually all modern LLMs and many computer-vision models.

Token  
The smallest unit of text an LLM processes. A token can be a word, sub-word, or punctuation mark depending on the tokenizer. Understanding tokenization is important because model context limits, pricing, and latency are all measured in tokens.

Prompt Engineering  
The practice of crafting input text (prompts) to steer an LLM toward a desired output. Techniques include providing examples (few-shot), assigning roles, and using structured instructions. Good prompt engineering can dramatically improve output quality without changing the model itself.

Fine-tuning  
The process of continuing a pre-trained model’s training on a smaller, task-specific dataset. Fine-tuning adapts general-purpose LLMs to specialized domains such as legal text or medical records, improving accuracy and relevance for those use cases.

Hallucination  
When an LLM generates information that sounds plausible but is factually incorrect or entirely fabricated. Hallucinations are a key limitation of LLMs and one of the primary motivations for Retrieval-Augmented Generation (RAG), which grounds answers in retrieved evidence.

Knowledge Cutoff  
The date after which an LLM has no training data. Any events, publications, or facts that emerged after the cutoff are unknown to the model unless supplied through retrieval or tool use. This is why RAG and web-search integrations are valuable complements to LLMs.

Temperature  
A sampling parameter that controls how random or creative an LLM’s output is. A temperature of 0 makes the model nearly deterministic (always picking the most likely token), while higher values (e.g., 0.8-1.0) increase diversity and creativity at the cost of consistency.

Top-k / Top-p Sampling  
Two strategies for narrowing the set of candidate tokens during generation. Top-k keeps only the *k* most probable tokens; Top-p (nucleus sampling) keeps the smallest set of tokens whose cumulative probability exceeds *p*. Both help balance quality and diversity in generated text.

In-Context Learning  
The ability of an LLM to learn a task from examples provided directly in the prompt, without any gradient updates. The model uses the patterns in the prompt to generalize and produce correct outputs. This emergent capability is what makes few-shot and zero-shot prompting possible.

Few-shot Learning  
Providing a small number of input-output examples in the prompt so the LLM can infer the desired behavior. Few-shot prompting is a practical application of in-context learning and often yields significant quality improvements over zero-shot approaches for structured tasks.

Zero-shot Learning  
Asking an LLM to perform a task with only an instruction and no examples. Zero-shot prompting works well for straightforward tasks but may struggle with complex or ambiguous requirements where examples would clarify the expected output format.

RLHF (Reinforcement Learning from Human Feedback)  
A training technique where human evaluators rank model outputs and a reward model is trained on those rankings. The LLM is then fine-tuned using reinforcement learning to maximize the reward signal. RLHF is a key step in aligning LLMs with human preferences and safety goals.

Attention Mechanism  
The core component of Transformers that allows each token to attend to every other token in the input sequence. Attention computes weighted relevance scores, enabling the model to capture long-range dependencies and contextual relationships. Multi-head attention runs several attention functions in parallel for richer representations.

------------------------------------------------------------------------

## RAG & Retrieval

RAG (Retrieval-Augmented Generation)  
An architecture that combines information retrieval with LLM generation. When a user asks a question, relevant documents are first retrieved from a knowledge base and then passed to the LLM as context. RAG reduces hallucinations and keeps answers grounded in up-to-date, domain-specific data.

Embedding  
A dense vector representation of text (or other data) in a continuous high-dimensional space. Embeddings capture semantic meaning so that similar concepts have vectors close together. They are the foundation of semantic search and are produced by specialized encoder models.

Vector Database  
A database optimized for storing, indexing, and querying high-dimensional vectors. Examples include Pinecone, Weaviate, Qdrant, and Chroma. Vector databases enable fast approximate nearest-neighbor search, which is essential for retrieving relevant documents in RAG pipelines.

Chunking  
The process of splitting large documents into smaller, semantically meaningful pieces before embedding them. Chunk size and overlap significantly affect retrieval quality. Common strategies include fixed-size, sentence-based, and semantic chunking.

Semantic Search  
Searching by meaning rather than keyword matching. A query is embedded into the same vector space as the documents, and the nearest neighbors are returned as results. Semantic search understands synonyms and paraphrases, making it more robust than traditional keyword search.

Dense Retrieval  
A retrieval approach that uses dense vector embeddings for both queries and documents. Similarity is computed via cosine distance or dot product. Dense retrieval excels at understanding intent and meaning but may miss exact keyword matches.

Sparse Retrieval  
A retrieval approach based on traditional term-frequency methods such as BM25 or TF-IDF. Documents and queries are represented as sparse vectors where each dimension corresponds to a vocabulary term. Sparse retrieval is fast and effective for exact keyword matching.

Hybrid Search  
Combining dense (semantic) and sparse (keyword) retrieval to get the best of both worlds. Results from each method are typically merged using Reciprocal Rank Fusion (RRF) or weighted scoring. Hybrid search tends to outperform either approach alone.

BM25  
A probabilistic ranking function widely used in information retrieval. BM25 scores documents based on term frequency, inverse document frequency, and document length normalization. It is the most common baseline for sparse retrieval and remains competitive in many scenarios.

RRF (Reciprocal Rank Fusion)  
A simple and effective method for combining ranked lists from multiple retrieval systems. Each document’s score is the sum of 1/(k + rank) across all lists. RRF is popular in hybrid search because it is parameter-light and robust.

HyDE (Hypothetical Document Embeddings)  
A query transformation technique where the LLM first generates a hypothetical answer to the query, and that answer is then embedded and used for retrieval. HyDE can improve recall by bridging the vocabulary gap between short queries and long documents.

HNSW (Hierarchical Navigable Small World)  
A graph-based algorithm for approximate nearest-neighbor search. HNSW builds a multi-layer graph structure that enables fast traversal to find similar vectors. It is one of the most widely used indexing algorithms in vector databases due to its balance of speed and recall.

Bi-Encoder  
A model architecture where the query and document are encoded independently into separate embeddings, then compared via cosine similarity. Bi-encoders are fast because document embeddings can be pre-computed and cached, making them ideal for first-stage retrieval.

Cross-Encoder  
A model that takes a query-document pair as a single input and outputs a relevance score. Cross-encoders are more accurate than bi-encoders because they model token-level interactions, but they are too slow for searching large collections. They are typically used as re-rankers.

MMR (Maximal Marginal Relevance)  
A technique for diversifying search results by penalizing documents that are too similar to already-selected ones. MMR balances relevance to the query with diversity of the result set, reducing redundancy in retrieved context.

Re-ranking  
A second-stage retrieval step where an initial set of candidate documents is re-scored using a more powerful model (often a cross-encoder). Re-ranking improves precision by promoting the most relevant documents to the top of the list.

Context Window  
The maximum number of tokens an LLM can process in a single forward pass, including both the prompt and the generated output. Context window size determines how much retrieved information can be passed to the model. Modern frontier models offer 1M-token context windows (Claude Opus 4.6, Claude Sonnet 4.6), while faster tier models typically offer 200K tokens.

Grounding  
The practice of anchoring LLM outputs in verifiable source material. In RAG systems, grounding means ensuring the model’s answer is supported by the retrieved documents rather than its parametric knowledge. Grounding reduces hallucinations and increases trustworthiness.

------------------------------------------------------------------------

## LangChain & LangGraph

Chain  
A LangChain abstraction that connects multiple processing steps (LLM calls, retrievals, transformations) into a sequential pipeline. Chains enable composable workflows where the output of one step feeds into the next. LCEL (LangChain Expression Language) is still used to compose runnables, but since LangChain 1.x most agent-style workloads are built with `create_agent` rather than hand-rolled chains.

Agent  
An LLM-powered entity that can reason about which tools to use and in what order to accomplish a goal. Unlike static chains, agents make dynamic decisions at runtime: they observe tool outputs and decide the next action. In LangChain 1.x, the canonical factory is `create_agent` from `langchain.agents`, which builds a LangGraph-backed runtime with streaming, persistence, and human-in-the-loop support out of the box.

Tool  
A function or API that an agent can invoke to interact with external systems. Examples include web search, database queries, calculators, and code execution. Tools extend the LLM’s capabilities beyond text generation to real-world actions and data access.

Memory  
A mechanism for maintaining conversation history or state across multiple interactions. LangChain provides several memory types including buffer memory (full history), summary memory (compressed history), and entity memory (key facts). Memory enables coherent multi-turn conversations.

Prompt Template  
A reusable template that combines static instructions with dynamic variables to construct prompts. Templates enforce consistent formatting and make it easy to swap variables without rewriting the entire prompt. LangChain supports chat and string prompt templates.

Retriever  
A LangChain interface for fetching relevant documents given a query. Retrievers abstract over different backends (vector stores, BM25, web search) and return a list of Document objects. They are the primary integration point between retrieval systems and LLM chains.

Document Loader  
A component that ingests data from various sources (PDFs, web pages, databases, APIs) and converts it into LangChain Document objects. Document loaders handle format-specific parsing so downstream components receive clean, uniform text.

Text Splitter  
A utility that breaks documents into smaller chunks suitable for embedding and retrieval. LangChain offers several splitters including recursive character, token-based, and semantic splitters. Choosing the right splitter and chunk size is critical for retrieval quality.

LangGraph  
A framework for building stateful, multi-actor LLM applications as directed graphs. Each node is a function or LLM call, and edges define the control flow including conditionals and loops. LangGraph extends LangChain with explicit state management and cyclic graph support.

State Graph  
The core LangGraph data structure where nodes represent computation steps and edges represent transitions between them. The graph maintains a shared state object that nodes can read and write, enabling complex coordination patterns like multi-agent collaboration.

Human-in-the-Loop  
A design pattern where the system pauses execution to request human input or approval before proceeding. In LangGraph, this is implemented via interrupt nodes that halt the graph and resume after receiving human feedback. It is essential for high-stakes decisions.

Checkpointer  
A LangGraph component that persists graph state at each step, enabling pause/resume, time-travel debugging, and fault recovery. Checkpointers can store state in memory, SQLite, or PostgreSQL, making long-running agent workflows resilient to failures.

------------------------------------------------------------------------

## Evaluation & Observability

RAGAS  
An open-source framework for evaluating RAG pipelines. RAGAS provides automated metrics that assess both retrieval quality and generation quality without requiring ground-truth answers for every question. It has become a standard tool for RAG evaluation.

Faithfulness  
A RAGAS metric that measures whether the generated answer is supported by the retrieved context. High faithfulness means the LLM is not hallucinating or adding information beyond what the documents provide. It is computed by checking each claim in the answer against the context.

Answer Relevancy  
A RAGAS metric that evaluates how well the generated answer addresses the original question. It penalizes answers that are incomplete, off-topic, or contain unnecessary information. Answer relevancy is measured by generating questions from the answer and comparing them to the original.

Context Recall  
A RAGAS metric that measures whether the retrieved documents contain all the information needed to answer the question. Low context recall indicates that the retrieval step is missing relevant documents, which limits the LLM’s ability to generate complete answers.

Context Precision  
A RAGAS metric that evaluates whether the retrieved documents are relevant and ranked appropriately. High context precision means the most useful documents appear at the top of the results. It penalizes retrieval of irrelevant or loosely related documents.

LangFuse  
An open-source observability platform for LLM applications. LangFuse provides tracing, prompt management, and evaluation tools. It integrates with LangChain and other frameworks to capture detailed logs of every LLM call, retrieval, and tool invocation in a pipeline.

LangSmith  
A commercial platform by LangChain for debugging, testing, and monitoring LLM applications. LangSmith provides detailed traces, dataset management, and evaluation workflows. It is tightly integrated with the LangChain ecosystem and useful for both development and production monitoring.

Tracing  
The practice of recording the complete execution path of an LLM application, including every chain step, LLM call, retrieval, and tool use. Traces capture inputs, outputs, latency, and token usage at each step, enabling developers to identify bottlenecks and debug failures.

Observability  
The ability to understand the internal state of an LLM application from its external outputs. Observability encompasses tracing, logging, metrics, and alerting. For LLM apps, it is critical because non-deterministic outputs make traditional debugging insufficient.

------------------------------------------------------------------------

## Software Engineering

REST API  
An architectural style for building web services where resources are identified by URLs and manipulated using standard HTTP methods (GET, POST, PUT, DELETE). REST APIs are stateless, meaning each request contains all information needed to process it. They are the most common way to expose backend functionality to clients.

JWT (JSON Web Token)  
A compact, URL-safe token format used for authentication and authorization. A JWT contains a header, payload (claims), and cryptographic signature. Servers can verify the token without database lookups, making JWTs efficient for stateless authentication in distributed systems.

OAuth2  
An authorization framework that allows third-party applications to access user resources without exposing credentials. OAuth2 defines flows (authorization code, client credentials, etc.) for different use cases. It is the standard for delegated authorization on the web.

Clean Architecture  
A software design philosophy that separates concerns into concentric layers: entities, use cases, interface adapters, and frameworks. Dependencies point inward, meaning business logic never depends on infrastructure details. This makes the codebase testable, maintainable, and adaptable to changing requirements.

Dependency Injection  
A design pattern where a component receives its dependencies from the outside rather than creating them internally. DI decouples components, making them easier to test (by injecting mocks) and more flexible (by swapping implementations). It is a cornerstone of clean architecture.

TDD (Test-Driven Development)  
A development methodology where tests are written before the production code. The cycle is: write a failing test, write the minimum code to pass it, then refactor. TDD produces well-tested code and drives simpler designs by forcing developers to think about requirements before implementation.

Unit Test  
A test that verifies a single function or class in isolation from external dependencies. Unit tests are fast, deterministic, and form the base of the testing pyramid. They catch logic errors early and serve as living documentation of expected behavior.

Integration Test  
A test that verifies the interaction between multiple components or with external systems (databases, APIs). Integration tests are slower than unit tests but catch issues that unit tests miss, such as configuration errors, serialization bugs, and contract mismatches.

Design Pattern  
A reusable solution template for a commonly occurring software design problem. Patterns like Repository, Strategy, Observer, and Factory provide a shared vocabulary and proven approaches. They help developers build maintainable, extensible systems without reinventing solutions.

Repository Pattern  
A design pattern that abstracts data access behind a collection-like interface. The repository provides methods like `find`, `save`, and `delete` while hiding the underlying storage mechanism. This decouples business logic from database details and simplifies testing.

SOLID  
Five principles of object-oriented design: Single Responsibility, Open/Closed, Liskov Substitution, Interface Segregation, and Dependency Inversion. SOLID principles guide developers toward code that is modular, extensible, and resistant to fragile coupling. They are foundational to clean architecture.

------------------------------------------------------------------------

## DevOps & Infrastructure

Docker  
A platform for building, shipping, and running applications in containers. Docker packages an application with all its dependencies into a standardized unit (image) that runs identically across environments. It eliminates “works on my machine” problems and simplifies deployment.

Container  
A lightweight, isolated runtime environment that shares the host OS kernel. Containers are faster to start and more resource-efficient than virtual machines. They encapsulate an application and its dependencies, ensuring consistent behavior across development, testing, and production.

Docker Compose  
A tool for defining and running multi-container Docker applications using a YAML configuration file. Compose lets you specify services, networks, and volumes in a single `docker-compose.yml`, making it easy to spin up complex development environments with one command.

CI/CD (Continuous Integration / Continuous Deployment)  
A set of practices that automate building, testing, and deploying code. Continuous Integration merges and tests code frequently; Continuous Deployment automatically releases validated changes to production. CI/CD pipelines catch bugs early and accelerate delivery cycles.

Pipeline  
An automated sequence of stages (build, test, deploy) that code passes through on its way to production. Pipelines are defined in configuration files (e.g., `.gitlab-ci.yml`, GitHub Actions) and run on every push or merge request. They enforce quality gates and reproducible releases.

Redis  
An in-memory data store used as a cache, message broker, and session store. Redis supports data structures like strings, hashes, lists, and sorted sets. Its sub-millisecond latency makes it ideal for caching frequently accessed data and reducing database load.

Caching  
The practice of storing frequently accessed data in a fast-access layer (like Redis or an in-memory store) to reduce latency and backend load. Caching strategies include cache-aside, write-through, and write-behind. Proper cache invalidation is one of the hardest problems in computing.

SonarQube  
A platform for continuous code quality inspection. SonarQube analyzes code for bugs, vulnerabilities, code smells, and test coverage. It integrates into CI/CD pipelines and provides dashboards and quality gates to enforce coding standards across teams.

Code Quality  
A measure of how well code adheres to best practices in readability, maintainability, testability, and reliability. Tools like SonarQube, linters, and formatters automate quality checks. High code quality reduces technical debt and makes the codebase easier to evolve.

GitFlow  
A branching model that defines a structured workflow for managing releases. GitFlow uses long-lived `main` and `develop` branches plus short-lived feature, release, and hotfix branches. It provides clear conventions for parallel development and release management.

------------------------------------------------------------------------

## Cloud

Cloud Computing  
The delivery of computing resources (servers, storage, databases, networking) over the internet on a pay-as-you-go basis. Cloud computing eliminates the need for organizations to own and maintain physical infrastructure, enabling rapid scaling and global deployment.

IaaS / PaaS / SaaS  
The three main cloud service models. Infrastructure as a Service (IaaS) provides virtual machines and networks. Platform as a Service (PaaS) provides managed runtimes and databases. Software as a Service (SaaS) provides ready-to-use applications. Each level abstracts more infrastructure management away from the user.

Serverless  
A cloud execution model where the provider dynamically manages server allocation. Developers deploy functions (e.g., AWS Lambda) that run on demand and scale automatically. Serverless eliminates server management and charges only for actual compute time, making it cost-effective for event-driven workloads.

Auto Scaling  
The automatic adjustment of compute resources based on demand. Auto scaling adds instances when traffic increases and removes them when it decreases. It ensures applications maintain performance during traffic spikes while minimizing costs during quiet periods.

S3 (Simple Storage Service)  
Amazon’s object storage service for storing and retrieving any amount of data. S3 organizes data into buckets and objects, offering high durability (99.999999999%), multiple storage classes for cost optimization, and fine-grained access controls. It is widely used for backups, static assets, and data lakes.

EC2 (Elastic Compute Cloud)  
Amazon’s virtual server service that provides resizable compute capacity in the cloud. EC2 instances come in various types optimized for compute, memory, storage, or GPU workloads. Users have full control over the operating system and software stack.

IAM (Identity and Access Management)  
A cloud service for controlling who can access which resources and what actions they can perform. IAM uses policies, roles, and groups to enforce the principle of least privilege. Proper IAM configuration is critical for cloud security.

------------------------------------------------------------------------

## ML, DL & MLOps

Feature  
A measurable input value a model uses to make a prediction. It often starts life as a raw column but just as often is derived from one. In classical machine learning a human designs features by hand; in deep learning the model learns its own from raw data (see [Representation Learning](#term-Representation-Learning)).

Label  
The known answer associated with a training example — what a supervised model is trained to predict.

Overfitting  
When a model memorizes training data instead of learning patterns that generalize. The signature is training performance far above validation performance.

Regularization  
Any deliberate constraint on a model — dropout, weight decay, early stopping — that trades a little training accuracy for better generalization.

Bias-Variance Tradeoff  
The balance between a model too simple to capture the pattern (high bias, underfitting) and one so flexible it memorizes noise (high variance, overfitting).

Label Leakage  
When a feature would not actually be available at prediction time, or directly encodes the answer, making offline metrics look better than the model will perform in production.

Hyperparameter  
A configuration value set before training rather than learned from data — learning rate, tree depth, number of layers.

Gradient Boosting  
An ensemble technique (XGBoost, LightGBM) that builds trees sequentially, each one correcting the errors of the ones before it. Frequently the strongest classical ML choice on tabular data.

SMOTE (Synthetic Minority Oversampling Technique)  
A technique for class imbalance that generates synthetic minority-class examples by interpolating between real ones. Applied to the training split only — applying it before splitting leaks information into validation and test data.

Class Imbalance  
When one class vastly outnumbers another in a dataset, making accuracy a misleading metric and requiring techniques like class weighting or resampling.

Precision  
Of everything a model flagged as positive, the fraction that was actually positive. Matters most when false positives are costly.

Recall  
Of everything that was actually positive, the fraction the model caught. Matters most when false negatives are costly.

F1 Score  
The harmonic mean of precision and recall, used when both error types matter and a single summary number is needed.

ROC-AUC  
A ranking metric for classifiers — how well the model ranks positive examples above negative ones — that can be misleading on strongly imbalanced data.

PR-AUC  
Like ROC-AUC, but focused on the positive class; the more reliable choice when positives are rare.

Confusion Matrix  
A table of predicted versus actual classes, the source from which precision, recall, and F1 are computed.

Neural Network  
A model composed of layers of connected units that transform an input through learned weights, trained via gradient descent to minimize a loss function.

Backpropagation  
The algorithm that computes how much each weight in a neural network contributed to the error, by propagating gradients backward through the layers — the mechanism behind the “backward pass” in a training loop.

Epoch  
One complete pass of the training data through a model during training. Deep learning models typically train over many epochs, unlike most classical ML models which fit in a single call.

Dropout  
A regularization technique that randomly disables a fraction of neurons during training, forcing the network not to over-rely on any single one.

Batch Normalization  
A technique that normalizes the inputs to a layer during training, stabilizing and often speeding up convergence.

Transfer Learning  
Reusing a model pretrained on a large dataset as the starting point for a new, related task, rather than training from scratch. Almost always the right first move when labeled data is limited.

Representation Learning  
The deep learning approach where a model learns its own useful features directly from raw data, contrasted with the hand-designed [Feature](#term-Feature) engineering of classical ML.

Data Augmentation  
Synthetic variation (crop, flip, noise, pitch shift) applied to training data on the fly so a model does not simply memorize the exact examples it was shown. Deep learning needs substantially more of this than tabular ML.

Convolutional Neural Network (CNN)  
A neural network architecture built around convolution, which shares weights across spatial positions in an image — the standard architecture family for computer vision.

Object Detection  
The computer vision task of locating and classifying multiple objects in an image, each with a bounding box, as distinct from whole-image classification.

IoU (Intersection over Union)  
A measure of overlap between two bounding boxes, used both as a detection metric and as a threshold for deciding whether a predicted box counts as correct.

mAP (mean Average Precision)  
The standard summary metric for object detection, aggregating precision and recall across object classes and IoU thresholds.

Semantic Segmentation  
The computer vision task of classifying every pixel in an image into a category, producing a mask rather than a bounding box.

OCR (Optical Character Recognition)  
The task of reading text from an image, conventionally split into text detection (finding where text is) and text recognition (reading what it says).

CTC Loss (Connectionist Temporal Classification)  
A loss function used in OCR and speech recognition that lets a model learn without knowing in advance how many input frames correspond to one output character or word.

CER / WER (Character / Word Error Rate)  
Evaluation metrics for OCR and speech transcription respectively, measuring insertions, deletions, and substitutions against a reference transcript.

Spectrogram  
A representation of audio showing which frequencies are present over time, the standard input representation for speech models — audio’s equivalent of a normalized pixel array.

Diarization  
The task of labeling *who* spoke *when* in an audio recording, distinct from transcription accuracy — a transcript can be word-perfect and still misattribute the speaker.

Experiment Tracking  
Recording the parameters, metrics, and artifacts of every training run so results can be compared and reproduced later. MLflow is the most common open-source implementation.

Run Manifest  
The complete record of one training run — run ID, artifact location, parameters, metrics, tags, and source code reference — needed to answer “what exactly produced this number?”

Artifact URI  
The location of a run’s actual output binary (model weights, dataset snapshot). Treated as the single source of truth; application databases store only a reference to it, never the binary itself.

Dataset Lineage  
The record of which dataset version a specific training run consumed, distinct from dataset versioning itself — versioning makes a dataset reproducible, lineage records which version was actually used.

Model Registry  
A system that tracks named model versions through stages — commonly Staging, Production, and Archived — supporting promotion, rollback, and audit.

Sealed Version  
A model version that has been promoted to production and is treated as immutable — any change produces a new version rather than an edit in place, so a version number always refers to the same artifact.

Data Drift  
A shift in the statistical distribution of input data between training and production, which can silently degrade a model’s real-world accuracy even though nothing about the model changed.

Concept Drift  
A shift in the relationship between inputs and the correct output itself — the patterns the model learned no longer hold — as distinct from [Data Drift](#term-Data-Drift), which is a shift in the inputs alone.

DAG (Directed Acyclic Graph)  
A graph of tasks and dependencies with no cycles, the standard way an orchestrator (such as Apache Airflow) represents a pipeline.
