# Basic AI Fundamentals - Final Assignment

## Assignment Metadata

| Field | Description |
|----|----|
| **Assignment Name** | RAG Agent for FPT Policy Document Q&A |
| **Course** | Unit 1: Basic AI Fundamentals |
| **Project Name** | fpt-policy-rag-agent |
| **Estimated Time** | 150 minutes |
| **Framework** | Python 3.11+, LangChain 1.0.5+, Amazon S3, Amazon Bedrock |

------------------------------------------------------------------------

## Learning Objectives

By completing this assignment, you will be able to:

- **Build** a complete RAG (Retrieval-Augmented Generation) pipeline from scratch

- **Implement** document loading, chunking, and embedding strategies

- **Configure** a vector database (FAISS or Chroma or S3 Vector) for efficient similarity search

- **Design** effective prompts for accurate and grounded answers

- **Apply** retrieval techniques including similarity search and metadata filtering

- **Create** a conversational agent that can answer questions about FPT policies

- **Validate** system responses for faithfulness and relevance

------------------------------------------------------------------------

## Assignment Description

### Background

FPT Corporation has various internal policy documents that employees need to reference frequently. Your task is to build a RAG-based Q&A system that allows employees to ask natural language questions and receive accurate answers based on the policy documents. Get your data from the folder ./data/FSoft_HR.pdf

### Requirements

Build a RAG Agent that:

1.  **Loads and processes** FPT policy documents (PDF/TXT format)

2.  **Chunks documents** using appropriate strategies (semantic or fixed-size)

3.  **Creates embeddings** and stores them in a vector database

4.  **Retrieves relevant** document chunks based on user queries

5.  **Generates accurate** answers grounded in the retrieved context

6.  **Handles edge cases** when information is not available

------------------------------------------------------------------------

## Technical Requirements

### Framework Requirements

| Component        | Requirement                   |
|------------------|-------------------------------|
| Python           | 3.11 or higher                |
| LangChain        | 1.0.5 or higher               |
| Vector Storage   | Amazon S3 Vector              |
| Embedding Model  | Amazon Bedrock Nova Embedding |
| LLM              | Amazon Bedrock Nova Pro       |
| Document Loaders | PyPDFLoader, TextLoader       |

### AWS Configuration

- Configure AWS credentials via AWS CLI or environment variables

- Ensure IAM permissions for Amazon Bedrock and S3 access

- Set AWS region (e.g., `us-east-1`) where Bedrock is available

### Code Quality Standards

- Use meaningful variable and function names

- Organize code into modular functions/classes

- Include docstrings for all functions

- Implement proper error handling with try/except blocks

- Use environment variables for AWS credentials (never hardcode)

------------------------------------------------------------------------

## Implementation Tasks

### Task 1: Document Loading & Chunking (30 mins)

Load policy documents from the provided dataset

Implement chunking with appropriate chunk size (500-1000 tokens)

Add metadata to chunks (source file, page number)

### Task 2: Embedding & Vector Store (30 mins)

Create embeddings using Amazon Bedrock Nova Embedding model

Store embeddings in Amazon S3 bucket

Implement persistence (save/load vectors from S3)

### Task 3: Retrieval System (30 mins)

Create a retriever with Top-K = 4

Implement similarity search

(Optional) Add metadata filtering capability

### Task 4: Generation & Prompt Engineering (30 mins)

Design a system prompt that instructs the LLM to:

- Only use information from the provided context

- Say “I don’t have information about this” when context is insufficient

- Cite the source document when possible

Build the complete RAG chain using LCEL

### Task 5: Testing & Validation (30 mins)

Test with at least 5 sample questions

Verify answers are grounded in the context (faithfulness)

Handle edge cases (out-of-scope questions)

------------------------------------------------------------------------

## Submission Requirements

### Required Deliverables

| Deliverable | Description                                               |
|-------------|-----------------------------------------------------------|
| Source Code | Jupyter Notebook (.ipynb) or Python script (.py)          |
| README.md   | Setup instructions and usage guide                        |
| Screenshots | Screenshots showing successful Q&A interactions           |
| Sample Q&A  | At least 5 question-answer pairs demonstrating the system |

### Submission Checklist

Project runs without errors

Vector database is created successfully

RAG chain generates accurate answers

System handles “unknown” questions gracefully

Code is well-documented with comments

README.md with setup instructions included

------------------------------------------------------------------------

## Evaluation Criteria

| Criteria           | Weight | Description                                 |
|--------------------|--------|---------------------------------------------|
| Functionality      | 40%    | RAG pipeline works correctly end-to-end     |
| Code Quality       | 20%    | Clean, modular, well-documented code        |
| Prompt Engineering | 15%    | Effective prompts that reduce hallucination |
| Error Handling     | 15%    | Graceful handling of edge cases and errors  |
| Documentation      | 10%    | Clear README and code comments              |

------------------------------------------------------------------------

## Sample Questions for Testing

Use these questions to test your RAG agent:

1.  “What is the leave policy for employees?”

2.  “How do I request work-from-home approval?”

3.  “What are the working hours at FPT?”

4.  “What is the process for expense reimbursement?”

5.  “Tell me about the weather today” (out-of-scope test)

------------------------------------------------------------------------

## Notes

- **Be Specific:** Ensure your retriever returns relevant chunks

- **Be Grounded:** Answers should only use information from the context

- **Be Transparent:** When information is not available, say so clearly

- **Be Efficient:** Optimize chunk size and Top-K for best results
