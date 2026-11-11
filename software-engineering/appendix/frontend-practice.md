# Frontend Practice — Chatbot Backend API

Build out the AI-backed chatbot endpoints following the reference shape below (modeled on the ChatGPT conversation API). The spec defines the base URL, core endpoints, request/response schemas, database schema, Redis caching strategy, and request flow summary.

## 🌐 Base URL

```
https://yourapp.com/backend-api/
```

------------------------------------------------------------------------

## ⚙️ 1. Core Endpoints

### **1.1 Generate Message (Send a Message)**

> **POST** `/backend-api/f/conversation` Used to send a message to the assistant and receive the generated response.

#### 🧾 Request Body

``` json
{
  "user_id": "user_123",
  "conversation_id": "690d5b6c-02d8-8321-a91e-65ea55b781f7",
  "messages": [
    {
      "id": "msg_001", // If not specified, backend generates a new message ID
      "role": "user",
      "content": {
        "content_type": "text",
        "parts": ["Hello! Can you explain Redis caching?"]
      }
    }
  ],
  "metadata": {
    "temperature": 0.7
  }
}
```

#### ✅ Response (Server-Sent Events Stream)

```
event: delta_encoding
data: "v1"
data: {"type": "resume_conversation_token", "token": "eyJhbGciOiJFUzI1NiIsInR5cCI6IkpXVCJ9...", "conversation_id": "690d5b6c-02d8-8321-a91e-65ea55b781f7"}
data: {"type": "input_message", "input_message": {"id": "msg_001", "author": {"role": "user"}, "create_time": 1762484546.479, "content": {"content_type": "text", "parts": ["Can you explain Redis caching?"]}, "status": "finished_successfully"}, "conversation_id": "690d5b6c-02d8-8321-a91e-65ea55b781f7"}
event: delta
data: {"o": "add", "v": {"message": {"id": "msg_002", "author": {"role": "assistant"}, "create_time": 1762484547.338, "update_time": 1762484547.817, "content": {"content_type": "text", "parts": [""]}, "status": "in_progress", "metadata": {"model_slug": "gpt-4-turbo-preview", "parent_id": "msg_001"}}, "conversation_id": "690d5b6c-02d8-8321-a91e-65ea55b781f7"}}
data: {"type": "message_marker", "conversation_id": "690d5b6c-02d8-8321-a91e-65ea55b781f7", "message_id": "msg_002", "marker": "user_visible_token", "event": "first"}
event: delta
data: {"v": [{"p": "/message/content/parts/0", "o": "append", "v": "Redis"}]}
event: delta
data: {"v": [{"p": "/message/content/parts/0", "o": "append", "v": " caching"}]}
event: delta
data: {"v": [{"p": "/message/content/parts/0", "o": "append", "v": " stores"}]}
event: delta
data: {"v": [{"p": "/message/content/parts/0", "o": "append", "v": " frequently"}]}
event: delta
data: {"v": [{"p": "/message/content/parts/0", "o": "append", "v": " used"}]}
event: delta
data: {"v": [{"p": "/message/content/parts/0", "o": "append", "v": " data"}]}
event: delta
data: {"v": [{"p": "/message/content/parts/0", "o": "append", "v": " in"}]}
event: delta
data: {"v": [{"p": "/message/content/parts/0", "o": "append", "v": " memory"}]}
event: delta
data: {"v": [{"p": "/message/content/parts/0", "o": "append", "v": " for"}]}
event: delta
data: {"v": [{"p": "/message/content/parts/0", "o": "append", "v": " fast"}]}
event: delta
data: {"v": [{"p": "/message/content/parts/0", "o": "append", "v": " access."}]}
event: delta
data: {"v": [{"p": "/message/status", "o": "replace", "v": "finished_successfully"}, {"p": "/message/end_turn", "o": "replace", "v": true}, {"p": "/message/metadata", "o": "append", "v": {"is_complete": true}}]}
data: {"type": "message_stream_complete", "conversation_id": "690d5b6c-02d8-8321-a91e-65ea55b781f7"}
data: [DONE]
```

#### 📖 SSE Stream Explanation

| Event Type | Purpose | Why It Matters |
|----|----|----|
| **`event:`` ``delta_encoding`** | Declares the streaming protocol version (`v1`) | Allows frontend to handle different streaming formats |
| **`resume_conversation_token`** | JWT token to resume/reconnect to this conversation stream | Enables connection recovery if stream is interrupted |
| **`input_message`** | Echoes back the user’s message that was just sent | Confirms message received and stored in DB |
| **`event:`` ``delta` (add)** | Creates a new assistant message with empty content and `status:`` ``"in_progress"` | Initializes the response container before streaming text |
| **`message_marker`** | Marks when the first visible token appears | Frontend can show “typing” indicator until this arrives |
| **`event:`` ``delta` (append)** | Streams individual text chunks (“Redis”, “ caching”, “ stores”…) | Creates the real-time typing effect by appending words progressively |
| **`event:`` ``delta` (replace status)** | Updates message status to `"finished_successfully"` and sets `end_turn:`` ``true` | Signals completion so frontend stops showing typing indicator |
| **`message_stream_complete`** | Confirms the entire message stream is done | Final acknowledgment that no more deltas are coming |
| **`[DONE]`** | Standard SSE termination signal | Closes the event stream connection |

#### 🔑 Key Concepts

**Delta Operations:**

- `"o":`` ``"add"` → Create new object

- `"o":`` ``"append"` → Add text to existing content

- `"o":`` ``"replace"` → Update a field value

- `"p":`` ``"/message/content/parts/0"` → JSON path to the field being updated **Why Streaming?**

1.  **User Experience**: Shows progress immediately instead of waiting for full response

2.  **Perceived Performance**: Feels faster even if total time is the same

3.  **Long Responses**: User sees content as it generates (important for long answers)

4.  **Interruptible**: User can stop generation early if they got their answer

#### 🧠 Notes

- If no `conversation_id` is provided → backend **creates a new one**.

- Each message (both user & assistant) is stored in the **messages** table.

- The backend can stream responses for real-time UI.

------------------------------------------------------------------------

### **1.2 List Conversations (Conversation History)**

> **GET** `/backend-api/conversations?offset=0&limit=20&order=updated` Returns a paginated list of all conversations for a user.

#### ✅ Response

``` json
{
  "items": [
    {
      "id": "690d5b6c-02d8-8321-a91e-65ea55b781f7",
      "title": "Redis Caching Explained",
      "last_message": "Redis caching stores frequently used data in memory...",
      "updated_at": "2025-11-07T09:30:00Z"
    },
    {
      "id": "732dd789-f512-7e01-b82e-13aa07b4e012",
      "title": "Python Decorators",
      "last_message": "A decorator wraps another function...",
      "updated_at": "2025-11-06T21:00:00Z"
    }
  ],
  "limit": 20,
  "offset": 0,
  "total": 2
}
```

#### 🧠 Notes

- Query params (`offset`, `limit`, `order`, etc.) make it frontend-friendly.

- Can be cached with Redis:

  - Key → `user:{user_id}:conversation_list`

  - TTL → 10 min

------------------------------------------------------------------------

### **1.3 Get Conversation Detail**

> **GET** `/backend-api/conversation/{conversation_id}` Returns all messages inside a specific conversation.

#### ✅ Response

``` json
{
  "conversation_id": "690d5b6c-02d8-8321-a91e-65ea55b781f7",
  "messages": [
    {
      "id": "msg_001",
      "role": "user",
      "content": {
        "content_type": "text",
        "parts": ["Hello! Can you explain Redis caching?"]
      },
      "created_at": "2025-11-07T09:20:00Z"
    },
    {
      "id": "msg_002",
      "role": "assistant",
      "content": {
        "content_type": "text",
        "parts": [
          "Redis caching stores frequently used data in memory for fast access."
        ]
      },
      "created_at": "2025-11-07T09:22:00Z"
    }
  ]
}
```

#### 🧠 Notes

- Ideal to cache full history:

  - Key → `conv:{conversation_id}:history`

  - TTL → 10 minutes

------------------------------------------------------------------------

## 🧱 2. Database Schema (SQLAlchemy Example Structure)

| Table | Purpose | Key Fields |
|----|----|----|
| **users** | Basic user info | `id`, `name`, `email` , `created_at` , `updated_at` |
| **conversations** | Chat sessions | `id`, `user_id`, `title`, `created_at` , `updated_at` |
| **messages** | Chat content | `id`, `conversation_id`, `role`, `content`, `created_at` , `updated_at` |

------------------------------------------------------------------------

## ⚡ 3. Redis Caching Strategy

| Cache Key | Example | Description | TTL |
|----|----|----|----|
| `conv:{id}:history` | `conv:690d5b6c-02d8-8321-a91e-65ea55b781f7:history` | Cached message list | 10 min |
| `user:{id}:conversation_list` | `user:user_123:conversation_list` | Cached user conversation list | 10 min |

------------------------------------------------------------------------

## 🧩 4. Folder Layout (FastAPI-Style)

```
app/
├── main.py
├── api/
│   ├── routes/
│   │   ├── conversation.py    # /backend-api/f/conversation, /backend-api/conversations
│   │   └── message.py         # Optional
│   └── dependencies.py
├── core/
│   ├── config.py
│   ├── database.py
│   └── redis_client.py
├── models/
│   ├── user.py
│   ├── conversation.py
│   └── message.py
├── schemas/
│   ├── conversation.py
│   ├── message.py
│   └── user.py
└── services/
    ├── ai_service.py          # LLM generation logic
    ├── cache_service.py       # Redis operations
    └── conversation_service.py # CRUD logic
```

------------------------------------------------------------------------

## 🧠 5. Request Flow Summary

1.  **Frontend** sends message via `POST`` ``/backend-api/f/conversation` → Backend validates input → fetches conversation → calls AI model → returns response.

2.  **Frontend** loads conversation list via `GET`` ``/backend-api/conversations?...` → Fetch from Redis cache → fallback to DB.

3.  **Frontend** loads conversation details via `GET`` ``/backend-api/conversation/{conversation_id}` → Return full history (cached or DB).
