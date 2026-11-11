# Final Exam

## overview

- **Course**: Advanced Microservices Architecture

- **Duration**: 180 minutes

- **Passing Score**: 70.0%

- **Total Points**: 100.0

## Description

You have been hired as a Lead Backend Engineer for ‘FinFlow’, a fintech startup. Your task is to build a high-performance transaction processing system. The system requires an API Gateway, a Transaction Service, and a Fraud Detection Service. You must implement secure access, asynchronous messaging for decoupling, and a Saga pattern to handle distributed consistency. You have 180 minutes to complete the implementation.

## Prerequisites

- Python 3.10+

- Docker & Docker Compose installed

- Experience with FastAPI and Pydantic

- Knowledge of RabbitMQ or Kafka

- Knowledge of Redis

------------------------------------------------------------------------

## Tasks

### Task 1: Secure API Gateway Implementation

**Time**: 0.75 hours

Initialize the project structure and create a centralized API Gateway using FastAPI. This gateway will serve as the single entry point for all client requests.

#### Requirements:

- Create a FastAPI application acting as the API Gateway.

- Implement a ‘Reverse Proxy’ mechanism that routes requests to downstream services (Transaction Service and Fraud Service) based on path prefixes (e.g., /api/v1/transactions).

- Implement JWT Authentication middleware at the Gateway level. Requests without a valid token should be rejected with HTTP 401.

- Create a mock ‘Auth Service’ endpoint within the Gateway (or separate) to issue tokens for testing.

#### Deliverables:

- gateway_service/ directory

- JWT verification logic

- Proxy routing logic

**Hints**:

- Use `httpx` for async proxying within FastAPI.

- Keep the secret key in an environment variable.

------------------------------------------------------------------------

### Task 2: Async Communication & Service Decoupling

**Time**: 1.0 hours

Implement the core Transaction Service and Fraud Detection Service. Use an event-driven approach to decouple the transaction creation from the fraud check.

#### Requirements:

- Create the ‘Transaction Service’ (FastAPI). It should accept a POST /transactions request, save the transaction with status ‘PENDING’ to a local database (SQLite/Postgres), and immediately return a 202 Accepted response.

- Upon saving, publish a ‘transaction_created’ event to a Message Broker (RabbitMQ).

- Create the ‘Fraud Service’. It must consume the ‘transaction_created’ event.

- The Fraud Service should contain logic to approve (amount \< 10000) or reject (amount \>= 10000) the transaction.

#### Deliverables:

- transaction_service/ directory

- fraud_service/ directory

- RabbitMQ producer/consumer implementation

**Hints**:

- Use Pydantic models to validate the message payload.

- Ensure the consumer runs as a background task or separate process.

------------------------------------------------------------------------

### Task 3: Distributed Transactions (Saga Pattern)

**Time**: 0.75 hours

Implement a Choreography-based Saga pattern to ensure data consistency across services.

#### Requirements:

- Modify the Fraud Service: After processing, it must publish a result event (‘fraud_check_passed’ or ‘fraud_check_failed’).

- Update the Transaction Service to consume these result events.

- If ‘fraud_check_passed’ is received, update the local transaction status to ‘CONFIRMED’.

- If ‘fraud_check_failed’ is received, update the local transaction status to ‘REJECTED’.

- Ensure idempotency: processing the same event twice should not corrupt the state.

#### Deliverables:

- Updated consumer logic in Transaction Service

- Updated producer logic in Fraud Service

**Hints**:

- Include the transaction_id in all messages to correlate events.

------------------------------------------------------------------------

### Task 4: Performance & Observability

**Time**: 0.5 hours

Enhance the system with caching and basic observability to monitor health and performance.

#### Requirements:

- Implement Redis caching in the Transaction Service for GET `/transactions/{id}` endpoints. Cache TTL should be 60 seconds.

- Add a structured logger to all services that includes the `correlation_id` (passed from the Gateway) in every log entry.

- Expose a standard `/health` endpoint in all services that checks database and broker connectivity.

#### Deliverables:

- Redis integration code

- Middleware for correlation ID extraction/logging

- Health check endpoints

**Hints**:

- Use Python’s `logging` library with a custom formatter or `structlog`.

------------------------------------------------------------------------

## Submission Rules

- All code must be submitted in a single Git repository.

- Include a `docker-compose.yml` file that orchestrates the Gateway, Transaction Service, Fraud Service, RabbitMQ, and Redis.

- Include a `README.md` with instructions on how to build, run, and test the endpoints using cURL or Postman.

- Ensure all dependencies are listed in `requirements.txt` or `pyproject.toml` for each service.

## Grading Rubrics

| Criterion | Weight | Excellent | Good | Satisfactory | Needs Improvement |
|----|----|----|----|----|----|
| Architecture & Microservices Patterns | 35.0% | Clear separation of concerns; correct implementation of API Gateway; seamless async communication; robust Saga pattern implementation. | Services are separated but logic may leak between layers; Async communication works but lacks robustness; Saga pattern works for happy path only. | Monolithic structure disguised as microservices; synchronous communication used where async was required. | Failed to implement microservices architecture; direct database access between services. |
| Functionality & Correctness | 35.0% | All endpoints work as specified; Security is enforced; Saga compensates correctly on failure; Cache hits/misses work as expected. | Core flows work; some edge cases in the Saga pattern fail; Security is present but permissive. | Basic CRUD works; Async messaging is flaky or incomplete; Caching is missing. | Code does not run or major features are missing. |
| Code Quality & Standards | 20.0% | PEP8 compliant; Type hinting used throughout; Effective error handling; Clean project structure. | Readable code; some typing missing; minor linting errors; adequate error handling. | Messy code structure; no type hinting; minimal error handling (lots of bare excepts). | Unreadable code; no modularity. |
| DevOps & Deliverables | 10.0% | Docker Compose brings up the whole stack with one command; README is comprehensive. | Docker configuration requires minor manual tweaks; README is brief. | Missing Docker files; manual startup of 5 terminals required. | No build instructions provided. |
