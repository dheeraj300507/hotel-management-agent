# System Architecture - Hotel Management Agent

This document details the system architecture, component interactions, and data flow for the **Hotel Management Agent**.

---

## 1. System Architecture Diagram

```mermaid
flowchart TD
    subgraph Client ["Client Layer"]
        UI["Web Browser Interface (HTML5 / CSS3 / JavaScript)"]
    end

    subgraph API ["Web and API Layer (FastAPI)"]
        Server["FastAPI Application (app/main.py)"]
        ChatRoute["Chat Endpoint: POST /agent/chat"]
        RestRoutes["REST Endpoints: /rooms, /guests, /bookings"]
    end

    subgraph Agent ["AI Agent Layer (app/agent.py)"]
        AgentCore["Agent Orchestrator (run_agent_chat)"]
        LLMEngine["OpenAI Function Calling Engine"]
        FallbackEngine["Zero-Config Rule and Regex Parser"]
    end

    subgraph Tools ["Domain Logic and Tools (app/tools.py)"]
        Dispatcher["Tool Dispatcher (execute_tool)"]
        T1["check_room_availability"]
        T2["create_booking"]
        T3["get_bookings"]
        T4["cancel_booking"]
        T5["get_guests"]
        T6["get_rooms"]
    end

    subgraph Database ["Persistence Layer (SQLite and SQLAlchemy)"]
        ORM["SQLAlchemy ORM (app/models.py)"]
        SQLite[("SQLite Database (hotel.db)")]
    end

    UI -->|"Natural Language Query"| ChatRoute
    UI -.->|"Direct API Access"| RestRoutes
    ChatRoute --> Server
    Server -->|"Delegate Message"| AgentCore
    AgentCore -->|"API Key Configured"| LLMEngine
    AgentCore -->|"No API Key / Fallback"| FallbackEngine
    LLMEngine -->|"Tool Execution"| Dispatcher
    FallbackEngine -->|"Matched Tool"| Dispatcher
    Dispatcher --> T1
    Dispatcher --> T2
    Dispatcher --> T3
    Dispatcher --> T4
    Dispatcher --> T5
    Dispatcher --> T6
    T1 --> ORM
    T2 --> ORM
    T3 --> ORM
    T4 --> ORM
    T5 --> ORM
    T6 --> ORM
    RestRoutes --> ORM
    ORM -->|"Read / Write"| SQLite
```

---

## 2. Component Explanation

### 1. Client Layer (`app/static/`)
- **Technology**: Vanilla HTML5, modern CSS3, and native JavaScript.
- **Role**: Lightweight chat interface providing a single-page view. Users send natural language queries (e.g., *"Show available rooms"*, *"Book room 101 for John"*) or click suggested prompt chips. Displays real-time responses and visual badges indicating which backend tools were invoked.

### 2. Web & API Layer (`app/main.py`, `app/routes/`)
- **Technology**: Python 3.12+, FastAPI, Uvicorn, Pydantic v2.
- **Role**:
  - Hosts the HTTP server, CORS middleware, and static asset mount.
  - Exposes the conversational agent endpoint: `POST /agent/chat`.
  - Exposes standard REST endpoints for programmatic operations:
    - `GET /rooms`: List and filter rooms by type and status.
    - `GET /guests`: Search and list registered guests.
    - `GET /bookings`: Query all bookings with optional status filters.
    - `POST /bookings`: Create a new reservation directly.
    - `DELETE /bookings/{id}`: Cancel a reservation by ID.

### 3. AI Agent Layer (`app/agent.py`)
- **Technology**: OpenAI-compatible function-calling client with rule-based fallback.
- **Role**:
  - Injects runtime context (such as the current system date) and tool definitions into the conversation.
  - **LLM Function-Calling Engine**: When an `OPENAI_API_KEY` is provided, drives an autonomous iterative tool-calling loop (up to 4 iterations) that translates user intent into concrete tool invocations.
  - **Zero-Config Fallback Engine**: If no API key is configured or an API error occurs, an intelligent pattern-matching engine handles availability checks, bookings, cancellations, and queries locally.

### 4. Domain Logic & Tools (`app/tools.py`)
- **Role**: Contains the business logic of hotel operations exposed as structured tools:
  1. `check_room_availability`: Identifies open rooms, filtering by room type and checking for date overlap against existing reservations.
  2. `create_booking`: Creates a guest profile if necessary, checks for date conflicts, reserves the room, and sets room occupancy.
  3. `get_bookings`: Retrieves active, historical, or today-only bookings.
  4. `cancel_booking`: Cancels a reservation and resets room availability if no other active reservations exist for today.
  5. `get_guests`: Looks up guest contact profiles.
  6. `get_rooms`: Returns full room inventory with pricing and operational states.
  - `execute_tool`: Central dispatcher routing function calls from either the LLM or the fallback engine to the corresponding Python function.

### 5. Persistence Layer (`app/database.py`, `app/models.py`)
- **Technology**: SQLite (`hotel.db`), SQLAlchemy 2.0 ORM.
- **Role**:
  - Manages database connection pooling (`engine`), session lifecycle (`SessionLocal`, `get_db`), and schema migration (`init_db()`).
  - Maintains 3 relational tables: `guests`, `rooms`, and `bookings` with indexed primary keys, foreign key constraints, and cascade delete behavior.
  - Details of tables and ER diagrams are provided in [database-schema.md](database-schema.md).

---

## 3. End-to-End Workflow: Booking a Room

```mermaid
sequenceDiagram
    autonumber
    actor User
    participant UI as Frontend (index.html)
    participant API as FastAPI (/agent/chat)
    participant Agent as Agent (agent.py)
    participant Tools as Tools (tools.py)
    participant DB as SQLite (hotel.db)

    User->>UI: Types "Book room 101 for John"
    UI->>API: POST /agent/chat {"message": "Book room 101 for John"}
    API->>Agent: run_agent_chat("Book room 101 for John", db)
    Agent->>Agent: Identifies intent & triggers create_booking
    Agent->>Tools: execute_tool("create_booking", {"room_number": "101", "guest_name": "John"}, db)
    Tools->>DB: Check room existence & conflicting bookings
    Tools->>DB: Fetch or create Guest "John"
    Tools->>DB: Insert Booking & update Room status
    DB-->>Tools: Booking persisted (ID #2)
    Tools-->>Agent: {"success": true, "booking_id": 2, "room_number": "101", ...}
    Agent-->>API: "Booking confirmed! Booking ID #2 created for John in Room 101..."
    API-->>UI: 200 OK {"response": "...", "tools_called": ["create_booking"]}
    UI-->>User: Displays assistant response with tool execution badge
```
