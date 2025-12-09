# 🛡️ Generative AI Hub (Indeed AI Toolbox) Showcase

![Python](https://img.shields.io/badge/Python-3.10+-blue.svg)
![Streamlit](https://img.shields.io/badge/Streamlit-Enterprise-red)
![Azure](https://img.shields.io/badge/Region-Germany%20West%20Central-0078D4)
![Security](https://img.shields.io/badge/Security-AES--256-green)

> **Portfolio Showcase:** An internal, multipage GenAI platform architected for **cross-functional innovation teams** (Designers, Engineers, Strategists).
> This repository demonstrates an **evolving Hub-and-Spoke architecture** built to strict **enterprise-grade standards** (GDPR, SSO, AES-256), serving 20+ specialized AI tools to streamline high-stakes client workflows.
---

---

## 📖 Purpose & Scope
The **Indeed AI Toolbox**(Gen AI Hub) is an internal, multipage Streamlit application that provides employees with a unified interface to multiple AI providers. It implements authentication-gated access to 20+ functional pages, organized across six primary domains:

1.  **Audio Transcription:** Automated speech-to-text pipelines.
2.  **Image Generation:** OpenAI and Gemini-based image creation.
3.  **Video Generation:** Content creation using models like Veo.
4.  **Conversational Chat Systems:** Reasoning and standard chat interfaces.
5.  **Knowledge-Base Agents:** Specialized agents for ISO standards and compliance.
6.  **Environmental Data Exploration:** Interfaces for the IDEMAT sustainability database.

**Development Philosophy: Workflow-Centric Evolution**
Unlike static software suites, this platform functions as a **living ecosystem**. It is developed in tight feedback loops with domain experts, ensuring that every tool maps directly to a specific professional workflow.

### Key Capabilities
* **Dual-Provider Strategy:** Routes complex reasoning/agents to **OpenAI** and multimedia tasks to **Google Gemini**.
* **Selective Persistence:** Only chat implementations persist conversations to Supabase (`simple_chat.py`), while others operate statelessly to minimize data footprint.
* **Iterative Lifecycle:** The architecture supports rapid prototyping, allowing tools to evolve from "WIP" status to production-hardened modules without disrupting the core system.
* **Data Sovereignty:** Full data residency within **Azure Germany West Central**.

---

---

## 🏗️ System Architecture
The application follows a **hub-and-spoke architecture** where `streamlit_app.py` serves as the central orchestrator, managing authentication, page registration, and global state, while individual page modules implement isolated functionality.

### Application Structure Diagram
The application follows a strict execution flow: **Entry → Authentication → Registration → Navigation → Execution**.
```mermaid
graph TD
    subgraph "Entry Point"
        Entry["streamlit_app.py Main<br>orchestrator"]
    end

    subgraph "Authentication Layer"
        Entry --> Gate{"st.user.is_logged_in<br>Gate check"}
        Logout["st.logout Session<br>termination"]
        
        Gate -- false --> LoginUI["login_screen()<br>SSO prompt"]
        LoginUI --> OAuth["st.login<br>OAuth flow"]
        OAuth -.->|Redirect| Gate
    end

    subgraph "Page Registry"
        Gate -- true --> Reg["PAGES dict 6 categories<br>20+ pages"]
        Reg --- Info["Info:<br>home.py, faq.py"]
        Reg --- Audio["Audio:<br>transcribe.py..."]
        Reg --- Image["Image:<br>generate.py..."]
        Reg --- Chats["Chats:<br>simple_chat.py..."]
        Reg --- Agents["Agents:<br>ISO.py..."]
    end

    subgraph "Navigation System"
        Reg --> NavFunc["st.navigation(PAGES)<br>Returns Page object"]
        NavFunc --> Exec["pg.run()<br>Executes selected page"]
    end

    subgraph "Global Resources"
        Exec -.-> User["st.user<br>OIDC identity"]
        Exec -.-> State["st.session_state<br>Per-session storage"]
        Exec -.-> Secrets["st.secrets<br>API keys, JWT secrets"]
    end
```

---

## 🔌 AI Provider Integration Architecture
The system employs a "Dual-Provider" strategy, routing tasks to specific models based on capability and cost.

```mermaid
graph LR
    subgraph "OpenAI Services"
        GPT4["GPT-4o (Reasoning)"] -->|routes to| Py1["reasoning_chat.py"]
        Agents["Agents SDK"] -->|routes to| Py2["ISO.py, Packaging.py"]
        Dalle["DALL-E 3"] -->|routes to| Py3["generate.py"]
    end

    subgraph "Gemini Services"
        Flash["Gemini 1.5 Flash"] -->|routes to| Py4["gemini_simple_chat.py"]
        Veo["Veo 3.1 (Video)"] -->|routes to| Py5["gemini_video.py"]
        Audio["Gemini Audio"] -->|routes to| Py6["transcription.py"]
    end

    subgraph "Shared Infrastructure"
        Py1 & Py2 & Py3 & Py4 & Py5 & Py6 -->|consumes| Secrets["st.secrets API Keys"]
        Py1 & Py4 -->|persists to| DB["Supabase (PostgreSQL)"]
    end
```

---

## 🔄 Application Lifecycle & State Management
The application initialization sequence ensures a secure and consistent state before rendering UI components. The `st.set_page_config()` call must execute before any other command, followed immediately by the authentication gate.

### Initialization Sequence Diagram
```mermaid
sequenceDiagram
    participant Runtime as Python Runtime
    participant Config as st.set_page_config()
    participant Auth as Authentication Check
    participant Nav as Navigation System
    participant Page as Selected Page Module

    Runtime->>Config: Set page title, icon, layout
    Runtime->>Auth: Check st.user.is_logged_in
    
    alt Not Logged In
        Auth->>Runtime: login_screen()
        Runtime->>Runtime: st.stop()
        Note over Runtime: Execution Halts Here
    else Logged In
        Auth->>Nav: Parse PAGES dict
        Nav->>Nav: st.navigation(PAGES)
        Nav->>Page: pg.run()
        
        activate Page
        Page->>Page: Initialize page-specific keys
        Page->>Page: Render UI Components
        Page->>Page: Update st.session_state
        deactivate Page
    end
```

---

## 🔒 Authentication & Security
The application implements mandatory Microsoft Entra ID authentication using Streamlit's built-in SSO capabilities.

### Authentication Flow Diagram
```mermaid
sequenceDiagram
    actor User
    participant App as streamlit_app.py
    participant Login as login_screen()
    participant Entra as Microsoft Entra ID

    User->>App: Navigate to App
    App->>App: Check st.user.is_logged_in

    opt Not Authenticated
        App->>Login: Render Prompt
        Login-->>User: Display "Continue" Button
        User->>Login: Click "Continue"
        Login->>Entra: st.login() triggers OIDC Flow
        Entra-->>User: User Authenticates (MFA)
        Entra-->>App: Return JWT Token / User Info
    end

    App->>App: st.stop() released
    App->>App: pg.run() executes page
```

### Encryption at Rest
All data persistence layers utilize **AES-256 encryption**.

| Layer | Method | Key Management |
| :--- | :--- | :--- |
| **Streamlit Secrets** | AES-256 | Managed |
| **Azure Blob Storage** | AES-256 | Microsoft Managed |
| **Supabase DB** | AES-256 | Supabase Managed |

---

## 🤖 Specialized AI Agents: The "Guardrail" Pattern
The platform utilizes the `openai-agents` framework. A key architectural pattern is the **Guardrail System**, specifically used in the **ISO Standards Agent**.

### Agent Architecture (ISO 59020/59010)
To prevent hallucinations and scope creep, the agent uses a **Pydantic-based Guardrail** before accessing the Vector Database.

```mermaid
graph LR
    User["User Input"] --> Guard["🛡️ Guardrail Agent"]
    Guard -->|Validates Topic| Decision{"Related to<br>Circularity?"}
    
    Decision -- No --> Reject["Return 'Off-Topic' Error"]
    Decision -- Yes --> Main["Main ISO Agent"]
    
    subgraph "RAG Pipeline"
        Main -->|Query| VectorDB[("Vector Store")]
        Main -->|Search| Web("Tavily API")
        VectorDB & Web --> Synthesis["Citation-Backed Answer"]
    end
```

**Implementation Highlight:**
The guardrail enforces structured output using a Pydantic schema (`ISOQueryOutput`), ensuring the AI returns a strict Boolean `is_related` flag rather than unstructured text.

---

## 🧩 Key Design Patterns
The codebase employs four distinct architectural patterns to ensure scalability and maintainability.

### 1. Page Registry Pattern
The `PAGES` dictionary in `streamlit_app.py` acts as a **Single Source of Truth** for page metadata.
* **Metadata Co-location:** Title, icon, and access level live alongside the file path.
* **Dynamic Navigation:** Streamlit constructs the UI directly from this registry, eliminating hardcoded sidebar logic.

### 2. Tool Card Registry Pattern
The `CARDS` dictionary in `home.py` decouples tool discovery from rendering logic.
* **Separation of Concerns:** Metadata (tags, description, status) is stored separately from the UI components.
* **Multi-View Rendering:** Allows the same tool data to be rendered in "Category View" or "Workflow View" without code duplication.

### 3. Authentication Gate Pattern
The authentication check uses the **"Early Termination"** pattern to enforce security at the entry point.
```python
if not st.user.is_logged_in:
    login_screen()
    st.stop() # Prevents execution of ANY subsequent page logic
```

### 4. Stateless vs. Stateful Page Pattern
The codebase employs two distinct persistence strategies:

| Pattern | Implementation | Examples | State Location |
| :--- | :--- | :--- | :--- |
| **Stateless** | No database persistence, session state only | `reasoning_chat.py`, `generate.py`, `ISO.py` | `st.session_state` |
| **Stateful** | Supabase persistence with CRUD operations | `simple_chat.py`, `gemini_simple_chat.py` | PostgreSQL via `supabase_client.py` |

---

## 🛠️ Technology Stack
The application adopts a "best-of-breed" approach, selecting specialized libraries for specific domains.

| Library | Purpose | Usage Context |
| :--- | :--- | :--- |
| `streamlit` | Core web framework | All pages, navigation, authentication |
| `openai-agents` | Agent framework | `ISO.py`, `Packaging.py` |
| `google-genai` | Gemini API client | All Gemini-powered pages |
| `supabase` | Database client | Conversation persistence |
| `postgrest` | PostgreSQL REST API | Underlying Supabase transport |
| `PyJWT` | JWT operations | Custom token minting in `supabase_client.py` |
| `Authlib` | OAuth 2.0/OIDC | Microsoft Entra ID integration |
| `python-docx` | Document generation | Export functionality in chat pages |