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

## Authentication and Session Management

### Authentication Guard
All chat systems require authentication before execution. Each implements an identical auth guard pattern:

```python
if not getattr(st, "user", None) or not st.user.is_logged_in:
    st.stop()
```

### Key Session State Variables
The application maintains specific state variables to manage context and history across reruns.

| Variable | Used By | Purpose |
| :--- | :--- | :--- |
| `messages` | Simple Chat, Gemini Chat | List of `{"role": str, "content": str}` dicts for chat history |
| `active_thread_id` | Simple Chat, Gemini Chat | UUID of current Supabase conversation record |
| `_starting_new_chat` | Simple Chat, Gemini Chat | Flag indicating user initiated a new blank conversation |
| `reasoning_chat` | Reasoning Chat | `streamlit_openai.Chat` instance with tool configuration |
| `co2e_cumulative_kg` | Simple Chat, Gemini Chat | Running total of carbon emissions for the session |

## Message Processing Flow

### Simple Chat Message Flow
The following sequence diagram details the execution flow for `simple_chat.py`. It demonstrates how the system handles state management, database persistence (Supabase), context windowing, and streaming responses while tracking token usage for carbon footprint calculations.

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