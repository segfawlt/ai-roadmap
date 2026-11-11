# Final Project Exam: FPT Customer Chatbot - Backend API System

## overview

| Field | Value |
|----|----|
| **Course** | Building Monolith API with FastAPI |
| **Project Name** | `fpt-customer-chatbot-api` |
| **Duration** | 360 minutes (6 hours) |
| **Passing Score** | 70% |
| **Total Points** | 100 |
| **Framework** | Python 3.10+, FastAPI, SQLAlchemy, SQLite, Pydantic, pytest |

------------------------------------------------------------------------

## Description

You have been hired as a **Backend Engineer** at **FPT Software**, tasked with building the **Backend API Layer** for the FPT Customer Service Chatbot system.

This exam builds upon the **AI Core** developed in the **LangGraph and Agentic AI** module. Your task is to create a production-ready FastAPI backend that:

1.  Provides REST API endpoints for the chatbot system

2.  Implements database persistence with SQLAlchemy

3.  Handles authentication and authorization

4.  Integrates with the AI Core (from LangGraph module)

5.  Implements proper error handling and logging

This exam assumes you have completed (or have access to) the **AI Core** from the LangGraph and Agentic AI module exam. You will integrate the AI agents with your FastAPI backend.

------------------------------------------------------------------------

## Objectives

By completing this exam, you will demonstrate mastery of:

- **FastAPI Fundamentals**: Building REST APIs with proper routing and request handling

- **Database Modeling**: Designing and implementing SQLAlchemy models

- **CRUD Operations**: Implementing Create, Read, Update, Delete operations

- **Dependency Injection**: Using FastAPI’s dependency injection system

- **Authentication**: Implementing JWT-based authentication

- **Input Validation**: Using Pydantic for request/response validation

- **Error Handling**: Implementing proper HTTP error responses

- **Unit Testing**: Writing tests with pytest

------------------------------------------------------------------------

## Problem Description

Build the **Backend API** for the FPT Customer Service Chatbot named `fpt-customer-chatbot-api` that includes:

| Component | Responsibilities |
|----|----|
| **Ticket API** | REST endpoints for support ticket CRUD operations |
| **Booking API** | REST endpoints for meeting room booking CRUD operations |
| **Chat API** | Endpoints to interact with the AI chatbot core |
| **User API** | User registration and authentication |
| **Database Layer** | SQLAlchemy models and database session management |

------------------------------------------------------------------------

## Prerequisites

- Completed FastAPI module assignments (recommended)

- Python 3.10+ with virtual environment

- SQLite (bundled with Python)

- AI Core from LangGraph module (or mock implementation for standalone testing)

------------------------------------------------------------------------

## Technical Requirements

### Environment Setup

- Python 3.10 or higher

- Required packages:

  - `fastapi` \>= 0.100.0

  - `uvicorn` \>= 0.23.0

  - `sqlalchemy` \>= 2.0.0

  - `pydantic` \>= 2.0.0

  - `python-jose[cryptography]` \>= 3.3.0

  - `passlib[bcrypt]` \>= 1.7.0

  - `pytest` \>= 7.0.0

  - `httpx` \>= 0.24.0

### Database Schema

Implement the following tables using SQLAlchemy:

**User Table:**

| Column          | Type     | Constraints                 |
|-----------------|----------|-----------------------------|
| id              | Integer  | Primary Key, Auto-increment |
| email           | String   | Unique, Required            |
| hashed_password | String   | Required                    |
| full_name       | String   | Required                    |
| phone           | String   | Optional                    |
| is_active       | Boolean  | Default: True               |
| created_at      | DateTime | Auto-set                    |

**Ticket Table:**

| Column      | Type     | Constraints                          |
|-------------|----------|--------------------------------------|
| id          | Integer  | Primary Key, Auto-increment          |
| ticket_id   | String   | Unique, Auto-generated UUID          |
| content     | String   | Required                             |
| description | Text     | Optional                             |
| user_id     | Integer  | Foreign Key to User                  |
| status      | Enum     | Pending/InProgress/Resolved/Canceled |
| created_at  | DateTime | Auto-set                             |
| updated_at  | DateTime | Auto-update                          |

**Booking Table:**

| Column     | Type     | Constraints                 |
|------------|----------|-----------------------------|
| id         | Integer  | Primary Key, Auto-increment |
| booking_id | String   | Unique, Auto-generated UUID |
| reason     | String   | Required                    |
| time       | DateTime | Required, must be future    |
| user_id    | Integer  | Foreign Key to User         |
| note       | Text     | Optional                    |
| status     | Enum     | Scheduled/Finished/Canceled |
| created_at | DateTime | Auto-set                    |
| updated_at | DateTime | Auto-update                 |

**Conversation Table:**

| Column          | Type     | Constraints                 |
|-----------------|----------|-----------------------------|
| id              | Integer  | Primary Key, Auto-increment |
| conversation_id | String   | Unique, Auto-generated UUID |
| user_id         | Integer  | Foreign Key to User         |
| created_at      | DateTime | Auto-set                    |
| updated_at      | DateTime | Auto-update                 |

------------------------------------------------------------------------

## Tasks

### Task 1: Project Setup & Database Models (15 points)

**Time Allocation**: 60 minutes

Set up the FastAPI project structure and implement database models.

#### Requirements:

1.  **Create project structure** following FastAPI best practices

2.  **Implement SQLAlchemy models** for all tables (User, Ticket, Booking, Conversation)

3.  **Create database session management** with dependency injection

4.  **Implement Pydantic schemas** for request/response validation

5.  **Set up Alembic migrations** (optional but recommended)

#### Deliverables:

- `models/` directory with all SQLAlchemy models

- `schemas/` directory with all Pydantic schemas

- `database.py` - Database connection and session management

- `config.py` - Application configuration

------------------------------------------------------------------------

### Task 2: Authentication & Authorization (20 points)

**Time Allocation**: 90 minutes

Implement JWT-based authentication system.

#### Requirements:

1.  **User Registration** endpoint:

    - `POST`` ``/api/v1/auth/register`

    - Hash passwords using bcrypt

    - Return user info (without password)

2.  **User Login** endpoint:

    - `POST`` ``/api/v1/auth/login`

    - Validate credentials

    - Return JWT access token

3.  **Token Verification**:

    - Create `get_current_user` dependency

    - Protect endpoints with authentication

    - Handle token expiration

4.  **User Profile** endpoint:

    - `GET`` ``/api/v1/users/me`

    - Return current user info

#### Deliverables:

- `routers/auth.py` - Authentication endpoints

- `routers/users.py` - User endpoints

- `utils/security.py` - JWT and password utilities

- `dependencies.py` - Shared dependencies

------------------------------------------------------------------------

### Task 3: Ticket API Implementation (20 points)

**Time Allocation**: 90 minutes

Implement full CRUD operations for support tickets.

#### Requirements:

1.  **Create Ticket**:

    - `POST`` ``/api/v1/tickets`

    - Requires authentication

    - Auto-generate ticket_id (UUID)

    - Set initial status to “Pending”

2.  **List Tickets**:

    - `GET`` ``/api/v1/tickets`

    - Support pagination (skip, limit)

    - Filter by status (optional)

    - Return only user’s tickets

3.  **Get Ticket**:

    - `GET`` ``/api/v1/tickets/{ticket_id}`

    - Return 404 if not found

    - Ensure user owns the ticket

4.  **Update Ticket**:

    - `PUT`` ``/api/v1/tickets/{ticket_id}`

    - Allow status transitions only

    - Validate status transitions

5.  **Cancel Ticket**:

    - `DELETE`` ``/api/v1/tickets/{ticket_id}`

    - Soft delete (change status to Canceled)

#### Deliverables:

- `routers/tickets.py` - Ticket endpoints

- `crud/tickets.py` - Ticket CRUD operations

- Unit tests for ticket endpoints

------------------------------------------------------------------------

### Task 4: Booking API Implementation (20 points)

**Time Allocation**: 90 minutes

Implement full CRUD operations for meeting room bookings.

#### Requirements:

1.  **Create Booking**:

    - `POST`` ``/api/v1/bookings`

    - Requires authentication

    - Validate time is in the future

    - Auto-generate booking_id (UUID)

2.  **List Bookings**:

    - `GET`` ``/api/v1/bookings`

    - Support pagination

    - Filter by status and date range

    - Return only user’s bookings

3.  **Get Booking**:

    - `GET`` ``/api/v1/bookings/{booking_id}`

    - Return 404 if not found

4.  **Update Booking**:

    - `PUT`` ``/api/v1/bookings/{booking_id}`

    - Allow updates only for “Scheduled” bookings

    - Validate new time is in future

5.  **Cancel Booking**:

    - `DELETE`` ``/api/v1/bookings/{booking_id}`

    - Soft delete (change status to Canceled)

#### Deliverables:

- `routers/bookings.py` - Booking endpoints

- `crud/bookings.py` - Booking CRUD operations

- Unit tests for booking endpoints

------------------------------------------------------------------------

### Task 5: Chat API & AI Integration (15 points)

**Time Allocation**: 60 minutes

Create endpoints to interact with the AI chatbot core.

#### Requirements:

1.  **Start Conversation**:

    - `POST`` ``/api/v1/chat/conversations`

    - Create new conversation record

    - Initialize AI graph with user context

2.  **Send Message**:

    - `POST`` ``/api/v1/chat/conversations/{conversation_id}/messages`

    - Forward message to AI core

    - Return AI response

    - Handle HITL interrupts

3.  **Confirm Action** (for HITL):

    - `POST`` ``/api/v1/chat/conversations/{conversation_id}/confirm`

    - Resume interrupted operations

    - Accept “confirm” or “cancel”

4.  **Get Conversation History**:

    - `GET`` ``/api/v1/chat/conversations/{conversation_id}`

    - Return all messages in conversation

If the AI Core is not available, implement mock responses that simulate the multi-agent behavior.

#### Deliverables:

- `routers/chat.py` - Chat endpoints

- `services/chat_service.py` - AI integration service

- `services/ai_adapter.py` - Adapter for AI Core (or mock)

------------------------------------------------------------------------

### Task 6: Error Handling & Testing (10 points)

**Time Allocation**: 30 minutes

Implement proper error handling and write unit tests.

#### Requirements:

1.  **Global Exception Handler**:

    - Handle validation errors

    - Handle database errors

    - Return proper HTTP status codes

2.  **Custom Exceptions**:

    - `NotFoundException` - 404 responses

    - `UnauthorizedException` - 401 responses

    - `ForbiddenException` - 403 responses

3.  **Unit Tests**:

    - Test all CRUD operations

    - Test authentication flow

    - Use pytest fixtures for database setup

#### Deliverables:

- `utils/exceptions.py` - Custom exception classes

- `tests/` directory with all test files

- Test coverage report

------------------------------------------------------------------------

## Test Scenarios

Complete these test scenarios to demonstrate system functionality:

### Scenario 1: User Registration & Authentication

```
1. Register new user with email and password
2. Login with credentials
3. Access protected endpoint with JWT token
4. Verify 401 response without token
```

### Scenario 2: Ticket Lifecycle

```
1. Create a support ticket
2. List all user's tickets
3. Update ticket status to "InProgress"
4. Resolve the ticket
5. Verify status transitions
```

### Scenario 3: Booking with Validation

```
1. Attempt to book with past time → 400 error
2. Create valid booking for future time
3. Update booking time
4. Cancel the booking
5. Verify cannot update canceled booking
```

### Scenario 4: Chat with AI (if integrated)

```
1. Start new conversation
2. Send IT support query → AI responds
3. Request to create ticket → HITL interrupt
4. Confirm action → Ticket created in database
5. Verify ticket exists via Ticket API
```

------------------------------------------------------------------------

## Questions to Answer

Include written responses to these questions in `ANSWERS.md`:

1.  **Database Design**: Why did you choose this table structure? What are the trade-offs of using foreign keys for user_id?

2.  **Authentication**: Explain the JWT token flow. What are the security considerations for token storage on the client side?

3.  **API Design**: Why use UUIDs for ticket_id/booking_id instead of auto-increment IDs? What are the pros and cons?

4.  **Error Handling**: How would you implement rate limiting for the API? What HTTP status code would you return?

5.  **Integration**: Describe how you would deploy this API with the AI Core in a production environment. Consider: containerization, scaling, monitoring.

------------------------------------------------------------------------

## Submission Requirements

### Directory Structure

```
fpt-customer-chatbot-api/
├── routers/
│   ├── auth.py
│   ├── users.py
│   ├── tickets.py
│   ├── bookings.py
│   └── chat.py
├── models/
│   ├── user.py
│   ├── ticket.py
│   ├── booking.py
│   └── conversation.py
├── schemas/
│   ├── user.py
│   ├── ticket.py
│   ├── booking.py
│   └── chat.py
├── crud/
│   ├── users.py
│   ├── tickets.py
│   └── bookings.py
├── services/
│   ├── chat_service.py
│   └── ai_adapter.py
├── utils/
│   ├── security.py
│   └── exceptions.py
├── tests/
│   ├── test_auth.py
│   ├── test_tickets.py
│   ├── test_bookings.py
│   └── conftest.py
├── main.py
├── database.py
├── config.py
├── dependencies.py
├── requirements.txt
├── README.md
└── ANSWERS.md
```

### Required Deliverables

Complete source code following directory structure

`README.md` with:

- Setup instructions (environment, dependencies)

- API documentation (or link to Swagger UI)

- Notes on AI Core integration

`ANSWERS.md` with written responses to all 5 questions

`requirements.txt` with all dependencies

Demo video or screenshots showing:

- User registration and login flow

- Ticket CRUD operations

- Booking CRUD operations

- Chat with AI (if integrated)

### Submission Checklist

All code runs without errors

Database models properly defined

Authentication flow works

All CRUD endpoints functional

Input validation working

Error handling implemented

Unit tests passing

Documentation complete

------------------------------------------------------------------------

## Evaluation Criteria

| Criteria | Points | Excellent (100%) | Good (75%) | Needs Improvement (50%) |
|----|----|----|----|----|
| **Project Setup & Models** (Task 1) | 15 | Perfect structure, all models, proper schemas | Working but minor issues in structure | Basic setup, missing some models |
| **Authentication** (Task 2) | 20 | Complete auth flow with proper security | Auth works but minor security gaps | Basic auth without token validation |
| **Ticket API** (Task 3) | 20 | Full CRUD with validation and tests | Most operations work, some validation missing | Only 1-2 operations functional |
| **Booking API** (Task 4) | 20 | Full CRUD with time validation and tests | Most operations work, time validation issues | Only 1-2 operations functional |
| **Chat API & Integration** (Task 5) | 15 | Full integration with AI Core working | Chat works with mock responses | Basic endpoint without proper integration |
| **Error Handling & Testing** (Task 6) | 10 | Comprehensive error handling and 80%+ coverage | Basic error handling and some tests | Minimal error handling, few tests |
| **Total** | **100** |  |  |  |

------------------------------------------------------------------------

## Hints

- Use `APIRouter` for organizing endpoints by resource

- Use Pydantic’s `BaseModel` for all request/response schemas

- Keep database models separate from Pydantic schemas

<!-- -->

- Use `passlib` with bcrypt for password hashing

- Use `python-jose` for JWT token creation/verification

- Store secret key in environment variables

<!-- -->

- Use `Depends(get_db)` for database session injection

- Consider using `AsyncSession` for better performance

- Use `relationship()` for foreign key relationships

<!-- -->

- Use `TestClient` from FastAPI for endpoint testing

- Create a separate test database

- Use `pytest.fixture` for test setup/teardown

<!-- -->

- Create an adapter layer to abstract AI Core

- Handle async operations properly

- Implement proper error handling for AI failures

------------------------------------------------------------------------

## References

- [FastAPI Documentation](https://fastapi.tiangolo.com/)

- [SQLAlchemy 2.0 Documentation](https://docs.sqlalchemy.org/en/20/)

- [Pydantic V2 Documentation](https://docs.pydantic.dev/latest/)

- [FastAPI Security](https://fastapi.tiangolo.com/tutorial/security/)

- [pytest Documentation](https://docs.pytest.org/)

- [JWT Token Best Practices](https://auth0.com/blog/a-look-at-the-latest-draft-for-jwt-bcp/)
