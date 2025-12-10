# 💬 Chat Systems

## Purpose and Scope
This document provides a technical overview of the conversational AI interfaces in the Gen AI Hub. The system implements four distinct chat interfaces, each serving different use cases: basic chat with persistence, advanced reasoning with tools, Gemini-powered chat, and transcript-based conversations.

This page focuses on the architectural patterns, data flows, and implementation details common across all chat systems.

## Chat System Overview
The application provides four chat interfaces differentiated by provider, model capabilities, and persistence strategy:

| Chat System | File | Model | Persistence | Key Features |
| :--- | :--- | :--- | :--- | :--- |
| **Simple Chat** | `simple_chat.py` | `gpt-4o-mini` | Supabase (conversations + messages) | Session-based conversation history, CO2e tracking, sliding window context |
| **Reasoning Chat** | `reasoning_chat.py` | `o3-mini` | Session-only | Code interpreter, file search, web search, DOCX export, `streamlit-openai` integration |
| **Gemini Chat** | `gemini_simple_chat.py` | `gemini-2.5-flash` | Supabase (conversations + messages) | Auto-thinking heuristics, dynamic temperature, model-aware CO2e |
| **Transcript Chat** | `transcribe_chat.py` | Various | Session-only | Audio transcript grounding, example prompts |

---

## High-Level Chat Architecture
The following diagram illustrates the data flow between the User Interface, Message Handlers, Data Persistence (Supabase), and External AI Providers.

```mermaid
graph TD
    subgraph "User Interface Layer"
        Input([st.chat_input])
        UI_Msg[st.chat_message]
        Sidebar[Sidebar Controls]
    end

    subgraph "Orchestration Layer"
        Handler{Message Handler}
        Payload[_build_payload]
        Stream[_stream_model_reply]
        History[_build_gemini_history]
        Wrapper[streamlit_openai.Chat]
        Export[export_chat_docx]
    end

    subgraph "Data Persistence"
        Supabase[(Supabase DB)]
        Session[st.session_state.messages]
        
        Supabase -- CRUD --> Tables["Tables: conversations, messages"]
    end

    subgraph "AI Provider APIs"
        OpenAI_Stream[client.responses.stream]
        OpenAI_O3[OpenAI o3 Model]
        Google[Google GenAI client]
    end

    %% Connections
    Input --> Handler
    Sidebar --> Handler
    
    %% Flow for Simple/Gemini Chat
    Handler --> Supabase
    Handler --> Session
    Handler --> Payload
    
    Payload --> Stream
    History --> Stream
    
    Stream --> OpenAI_Stream
    Stream --> Google
    
    %% Flow for Reasoning Chat
    Handler --> Wrapper
    Wrapper --> OpenAI_O3
    Wrapper --> Export
    Export --> UI_Msg

    %% Feedback Loop
    OpenAI_Stream & Google & OpenAI_O3 --> UI_Msg
```

---

## Authentication and Session Management

### Authentication Guard
All chat systems require authentication before execution. Each implements an identical auth guard pattern:

```python
if not getattr(st, "user", None) or not st.user.is_logged_in:
    st.stop()
```

### Key Session State Variables

| Variable | Used By | Purpose |
| :--- | :--- | :--- |
| `messages` | Simple Chat, Gemini Chat | List of `{"role": str, "content": str}` dicts for chat history |
| `active_thread_id` | Simple Chat, Gemini Chat | UUID of current Supabase conversation record |
| `_starting_new_chat` | Simple Chat, Gemini Chat | Flag indicating user initiated a new blank conversation |
| `reasoning_chat` | Reasoning Chat | `streamlit_openai.Chat` instance with tool configuration |
| `co2e_cumulative_kg` | Simple Chat, Gemini Chat | Running total of carbon emissions for the session |

---

## Message Processing Flow

---

# 🔵 Simple Chat Message Flow

The following sequence diagram details the execution flow for `simple_chat.py`.  
It demonstrates how the system handles:

- State management  
- Database persistence (Supabase)  
- Sliding context window  
- Streaming responses  
- Token usage & carbon footprint tracking  

```mermaid
sequenceDiagram
    actor User
    participant Input as "st.chat_input()"
    participant Handler as "Message Handler"
    participant DB as "Supabase (_insert_message)"
    participant Logic as "_build_payload()"
    participant Stream as "_stream_model_reply()"
    participant API as "OpenAI Responses API"
    participant CO2 as "add_co2e...()"
    participant State as "st.session_state"

    User->>Input: Type message
    Input->>Handler: user_input
    
    %% User Message Handling
    Handler->>State: Append user message to messages[]
    Handler->>DB: _insert_message(thread_id | "user" | content)
    
    %% Context Construction
    Handler->>Logic: _build_payload(messages)
    activate Logic
    Logic->>Logic: Append system prompt
    Logic->>Logic: Apply sliding window (MEMORY_MAX_TURNS=40)
    Logic-->>Handler: payload
    deactivate Logic

    %% Streaming Response
    Handler->>Stream: client.responses.stream(model | input)
    activate Stream
    
    loop Stream chunks
        Stream->>API: Request stream
        API-->>Stream: event.delta (text token)
        Stream-->>User: Yield token (st.write_stream)
    end
    
    API-->>Stream: event.response.usage (final)
    Stream->>State: Save openai_last_usage
    Stream-->>Handler: full_text
    deactivate Stream

    %% Assistant Message Persistence
    Handler->>State: Append assistant message
    Handler->>DB: _insert_message(thread_id | "assistant" | full_text)

    %% Eco-Tracking
    Handler->>CO2: add_co2e_from_usage_modelaware(usage | MODEL)
    CO2->>State: Update co2e_cumulative_kg
    Handler-->>User: Display eco caption
```

---

# 🔵 Gemini Chat Message Flow

The `gemini_simple_chat.py` system dynamically adjusts generation parameters using **Auto-Thinking Heuristics**, builds context-aware Gemini-compatible history, and processes a richer token stream containing structured *thoughts* and *outputs*.

```mermaid
sequenceDiagram
    actor User
    participant Handler as "Message Handler"
    participant Heuristics as "Auto Heuristics"
    participant History as "_build_gemini_history()"
    participant StreamGen as "generate_content_stream()"
    participant API as "Gemini API"
    participant State as "st.session_state"

    User->>Handler: user_input
    
    %% Heuristics Phase (Dynamic Configuration)
    Handler->>Heuristics: choose_temperature(user_input)
    Heuristics-->>Handler: temperature (0.3-0.7)
    
    Handler->>Heuristics: choose_thinking_budget(user_input | last_usage)
    Heuristics-->>Handler: budget (0 | AUTO_MEDIUM_BUDGET | or -1)

    %% History Conversion
    Handler->>History: _build_gemini_history(messages)
    activate History
    History->>History: Convert st.session_state to List[types.Content]
    History-->>Handler: contents
    deactivate History

    %% Streaming & Thought Extraction
    Handler->>StreamGen: generate_content_stream(contents | config)
    
    loop Stream chunks
        StreamGen->>API: Request stream
        API-->>StreamGen: chunk
        
        StreamGen->>StreamGen: Extract thought parts (if include_thoughts)
        StreamGen->>StreamGen: Extract text parts
        
        StreamGen->>User: Display text incrementally
        
        StreamGen->>State: Update thoughts in expander
    end
    
    API-->>StreamGen: (final response)
    
    StreamGen-->>Handler: full_text, full_thoughts, last_usage
    
    %% Persistence and State Update
    Handler->>Handler: Append assistant message
    Handler->>Handler: Update co2e_cumulative_kg
    Handler->>State: Save updated state
```

---

# 🗄️ Conversation Persistence Architecture

Only **Simple Chat** and **Gemini Chat** implement full database persistence.  
The Reasoning Chat and Transcript Chat remain session-only for security and simplicity.

---

## Database Schema (Supabase / PostgreSQL)

```mermaid
graph TD
    subgraph DB["Supabase Database"]
        Conversations[("conversations")]
        Messages[("messages")]
    end

    Conversations -->|1:N| Messages
```

### Row-Level Security (RLS)

- Conversations and messages are filtered via `user_key`  
- The `sub` claim in the user's JWT automatically applies identity scoping  
- Enforced on all inserts, updates, deletes, and selects  

---

## Conversation Management Functions

| Function | Purpose | Key Logic |
| :--- | :--- | :--- |
| `_list_conversations()` | Fetch user's conversations | Ordered by `updated_at DESC`, limit 200 |
| `_get_conversation()` | Fetch single conversation by ID | RLS filters by user |
| `_create_conversation()` | Creates new conversation entry | Uses UUID; user_key implicitly added |
| `_update_conversation_title_once()` | Generates short title | Only updates if title still placeholder |
| `_insert_message()` | Writes message to DB | Also updates parent `updated_at` |
| `_list_messages()` | Loads history for UI | Ordered by `created_at ASC` |
| `_delete_conversation()` | Hard-delete conversation | Cascades to messages |
| `_delete_all_my_chats()` | Bulk deletion | RLS ensures only user rows deleted |

---

# 📝 Conversation Title Generation Strategy

Three-stage upgrade strategy:

| Step | Purpose |
|---|---|
| **1. Initial Title** | Using first-line summary or timestamp placeholder |
| **2. AI Upgrade** | After assistant reply → summarize to 4–8 word title |
| **3. One-Time Update** | Only updates if placeholder still matches pattern (regex) |

Ensures efficiency:  
- Low API usage  
- Stable, user-editable titles  

---

# 🧠 Gemini Auto-Thinking Heuristics

The **Gemini Chat** dynamically selects:

- Temperature  
- Thinking budget  
- Safety blocks  
- Model configuration  

## Temperature Logic

```python
def choose_temperature(prompt: str) -> float:
    if _matches_any(prompt, _CREATIVE_HINTS):
        return 0.7
    if _matches_any(prompt, _FACTY_HINTS):
        return 0.3
    return 0.4
```

## Thinking Budget Logic

- 0 → For simple, factual queries  
- `AUTO_MEDIUM_BUDGET = 1024` → For moderately complex reasoning  
- `AUTO_HARD_BUDGET = -1` → Fully dynamic budget  
- **Cost spike protection:**  
  If last token usage > 20k → Downshift next request  

Constants:

- `MAX_THOUGHT_BUDGET_CAP = 8192`  
- `COST_SPIKE_TOTAL_TOKENS = 20000`  

---

# 🤖 Reasoning Chat Architecture

The Reasoning Chat system uses **streamlit-openai** as a wrapper to provide:

- Tool use  
- Code interpreter  
- File search  
- Web search  
- Image generation  
- DOCX export  

## Component Diagram

```mermaid
graph LR
    %% Nodes
    Script[reasoning_chat.py]
    Chat[streamlit_openai.Chat]
    Model[OpenAI o3 model]
    Tools[Tools]
    Sections[_sections list]
    Export["export_chat_docx()"]

    %% Subgraph for Capabilities
    subgraph Caps [Tool Capabilities]
        direction TB
        Code[allow_code_interpreter=True]
        File[allow_file_search=True]
        Web[allow_web_search=True]
        Img[allow_image_generation=True]
    end

    %% Flow/Edges
    Script --> Chat
    Chat -- API calls --> Model
    Chat -- configured with --> Tools
    Chat -- stores in --> Sections
    Sections --> Export

    Model --> Code
    Tools --> File
    Tools --> Web
    Tools --> Img
```

## Initialization

```python
st.session_state.reasoning_chat = streamlit_openai.Chat(
    api_key=api_key,
    model="o3",
    welcome_message="Welcome to the Reasoning Chat! I'm here to assist you with complex reasoning.",
    allow_code_interpreter=True,
    allow_file_search=True,
    allow_web_search=True,
    allow_image_generation=True,
)
```

---
# Related Documents
| File | Description |
|------|-------------|
| [**architecture.md**](../architecture.md) | High-level system overview, Hub-and-Spoke architecture, lifecycle diagrams |
| [**chat-systems.md**](./chat-systems.md) | Architecture of all chat systems (OpenAI, Gemini, Reasoning, Transcript) |
| [**agents.md**](./agents.md) | ISO Agent, Guardrail system, RAG pipeline, Packaging Agent |
| [**authentication.md**](./authentication.md) | Entra ID SSO, JWT identity mapping, RLS integration |
| [**persistence.md**](./persistence.md) | Supabase schema, CRUD functions, auto-purge retention logic |
| [**heuristics.md**](./heuristics.md) | Gemini auto-thinking heuristics (temperature, budgets, cost protection) |
| [**provider-routing.md**](./provider-routing.md) | Dual-provider routing between OpenAI & Gemini |
| [**patterns.md**](./patterns.md) | Core design patterns across the Hub |
| [**ui-showcase.md**](./ui-showcase.md) | Screenshots, video demo references, UI patterns |