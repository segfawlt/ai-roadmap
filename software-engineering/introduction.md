# Introduction to Software Engineering

The rapid growth of cloud computing, microservices, and DevOps practices has transformed how modern software is built and deployed. Today’s software engineers need more than just coding skills — they must understand concurrency models, API development, database design, authentication, testing methodologies, and architectural patterns to build robust, scalable applications.

This track provides a comprehensive foundation in essential software engineering practices. Whether you’re building AI applications, web services, or enterprise systems, these skills are crucial for delivering production-quality software.

## Learning Path

The curriculum is organized into four progressive tiers followed by quick-reference cheatsheets and a final project exam:

### Tier 1 — Foundations

Web fundamentals, OS concepts, and Python concurrency — the building blocks every backend engineer must understand:

- HTTP protocol, client-server communication, and REST principles

- Threading, the GIL, multiprocessing, and process forking

- Python asyncio: event loops, coroutines, and cooperative concurrency

- Greenlets and lightweight context switching

- File descriptors, sockets, and event-driven I/O

- CPython internals

### Tier 2 — API Development

Building web APIs with FastAPI — request handling, data validation, and dependency injection:

- FastAPI introduction, ASGI, and Uvicorn

- Path parameters, query parameters, headers, cookies, and request bodies

- Pydantic data modeling and validation

- Dependency injection with FastAPI’s `Depends()` system

### Tier 3 — Data Persistence

PostgreSQL database setup, async sessions, schema migrations, and CRUD patterns:

- PostgreSQL with SQLAlchemy ORM setup

- Async database sessions with SQLAlchemy

- Alembic database migrations

- Complete CRUD application architecture

### Tier 4 — Security & Testing

Securing and testing production APIs:

- JWT fundamentals (HS256, RS256, token lifecycle)

- OAuth2 authorization framework and Google authentication

- Authentication patterns in FastAPI

- Unit testing with pytest, TestClient, and async mocks

### Cheatsheets

Quick-reference pages covering cross-cutting topics used throughout all tiers:

- Git collaboration workflows and branching strategies

- Relational databases, SQL, and indexing

- RESTful API design and security

- Testing methodologies and TDD

- Docker fundamentals and container orchestration

- CI/CD pipelines with GitLab CI

- Secure coding practices

- Code review best practices

- Clean architecture and layered design

- Microservices vs serverless architecture

### Final Project Exam

An end-to-end project integrating all four tiers — building a complete authenticated CRUD API with database persistence and AI integration.

## Glossary

| Term | Description |
|----|----|
| Version Control | A system that records changes to files over time, enabling collaboration and history tracking. |
| CI/CD | Continuous Integration/Continuous Deployment — automating the build, test, and release process. |
| Containerization | Packaging applications with their dependencies into isolated, portable containers. |
| Microservices | An architectural style that structures an application as a collection of loosely coupled services. |
| RESTful API | An architectural style for designing networked applications using HTTP methods and stateless communication. |
| TDD | Test-Driven Development — writing tests before implementing the actual code. |
| Clean Architecture | A software design philosophy that separates concerns into distinct layers with clear dependencies. |
| ASGI | Asynchronous Server Gateway Interface — the async successor to WSGI for Python web applications. |
| ORM | Object-Relational Mapping — a technique for querying databases using programming language objects. |
| JWT | JSON Web Token — a compact, URL-safe means of representing claims for stateless authentication. |
