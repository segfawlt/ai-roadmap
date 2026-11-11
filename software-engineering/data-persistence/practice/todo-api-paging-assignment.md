# Exercise: Pagination, Filtering, Validation & Error Handling

## Goal

Improve the API for real-world use: list endpoints with **pagination**, **filtering (completed, search by title)**, and strong **input validation** + **proper HTTP errors**.

## Tasks

1.  **List endpoint with query params**

    - `GET`` ``/todos?limit=20&offset=0&completed=true&search=read`

    - Return metadata: `total`, `limit`, `offset` and `items`.

2.  **Validation**

    - Enforce `limit` \[1..100\], `offset` ≥ 0.

    - Title length validated in creation/update.

3.  **Error handling**

    - Return 400 for invalid query params.

    - 404 for missing resource with consistent error structure: `{`` ``"detail":`` ``"...",`` ``"code":`` ``"NOT_FOUND"`` ``}`.

4.  **Sorting**

    - Support `sort=-created_at` or `sort=title`.

5.  **OpenAPI docs**

    - Describe each query param with examples.

## Acceptance Criteria

- Pagination and filters work and are documented in `/docs`.

- Errors are consistent with status codes.

## Deliverables

- Updated source + docs.

- Sample curl calls demonstrating pagination + filters.

## Rubric (50 pts)

- Pagination

- Filtering & sorting

- Validation & error handling

- Docs clarity
