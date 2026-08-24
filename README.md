# TripMate AI ✈️🗺️

TripMate AI is a state-of-the-art, multi-agent travel planning system built on **LangGraph**, **FastAPI**, and the **Model Context Protocol (MCP)**. It uses a supervisor-directed agent topology with integrated input guardrails and Human-in-the-Loop (HITL) approval steps to curate practical, personalized, and budget-aware travel itineraries.

---

## 🏗️ Architecture & Agent Workflow

The system is designed as a stateful graph where a central Supervisor orchestrates specialized agents and interacts with the user:

```mermaid
graph TD
    Start([User Query]) --> SG[Input Guardrail]
    SG -->|Invalid/Harmful| BlockedNode[Guardrail Blocked Node]
    SG -->|Valid Travel Request| Supervisor[Supervisor Agent]
    
    Supervisor -->|Routes to Active Specialists| Flight[Flight Agent]
    Supervisor -->|Routes to Active Specialists| Hotel[Hotel Agent]
    Supervisor -->|Routes to Active Specialists| Weather[Weather Agent]
    Supervisor -->|Routes to Active Specialists| Budget[Budget Agent]
    
    Flight --> Itinerary[Itinerary Agent]
    Hotel --> Itinerary
    Weather --> Itinerary
    Budget --> Itinerary
    
    Itinerary --> HITL[Human-in-the-Loop Approval]
    HITL -->|Wait for Feedback / Approval| Final[Final Agent]
    BlockedNode --> End([Final Response])
    Final --> End
```

### Key Components

1. **Input Guardrail**: Evaluates queries before processing to ensure they are travel-related and safe. Unrelated queries are politely rejected.
2. **Supervisor Agent**: Extracts trip constraints (destination, duration, budget, preferences) and dynamically selects which specialist agents are required.
3. **Specialist Agents**:
   - **Flight Agent**: Consults the **AviationStack MCP** to provide route suggestions, airline information, and estimated airfare guidance.
   - **Hotel Agent**: Utilizes **Tavily MCP** to perform live searches for accommodations and neighborhood recommendations.
   - **Weather Agent**: Calls a custom **OpenWeather MCP Server** to retrieve current weather metrics and a 5-day forecast.
   - **Budget Agent**: Performs cost-estimation and budget feasibility checks based on the results from the other specialists.
4. **Itinerary Agent**: Consolidates constraints and specialist recommendations into an initial draft plan.
5. **Human-in-the-Loop Approval**: Pauses the graph using LangGraph's native `interrupt`. The user can inspect the draft, accept it, or supply revision feedback (which sends the graph back to re-generate).
6. **Final Agent**: Polishes and formats the final approved itinerary beautifully into distinct sections.

---

## 📁 Repository Structure

```text
TripMate/
├── app.py                         # FastAPI web server & api endpoints
├── backend.py                     # LangGraph workflow, agent nodes, and Postgres checkpointer
├── custom_weather_mcp_server.py   # Custom OpenWeather FastMCP tool provider
├── mcp_client.py                  # MCP MultiServer Client wrapper (Tavily, AviationStack, Weather)
├── requirements.txt               # Project python dependencies
├── templates/
│   └── index.html                 # FastAPI web application UI template
└── static/
    ├── script.js                  # Frontend Javascript handling agent chat, HITL state, and API endpoints
    └── style.css                  # Modern UI styles
```

---

## 🛠️ Setup & Installation

### 1. Prerequisites
- **Python**: Version `3.10` or higher (Conda environment recommended).
- **uv**: Recommended for managing external tools (`uvx`).
- **PostgreSQL**: An active instance (e.g., Render PostgreSQL, Supabase, or local instance) for LangGraph state persistence.

### 2. Clone and Install Dependencies
Clone the repository and install the dependencies:
```bash
pip install -r requirements.txt
```

### 3. Environment Configuration
Create a `.env` file in the root directory and populate it with your API keys:
```env
# LangGraph Persistent Store (PostgreSQL)
DATABASE_URL="postgresql://username:password@hostname:port/database?sslmode=require"

# Large Language Model (Groq API)
GROQ_API_KEY="your_groq_api_key"

# Specialist Search & APIs
TAVILY_API_KEY="your_tavily_api_key"
OPENWEATHER_API_KEY="your_openweather_api_key"
AVIATIONSTACK_API_KEY="your_aviationstack_api_key"

# Optional: LangSmith Tracing & Observability
LANGSMITH_TRACING="true"
LANGSMITH_ENDPOINT="https://api.smith.langchain.com"
LANGSMITH_API_KEY="your_langsmith_api_key"
LANGSMITH_PROJECT="TripMate-AI"
```

> [!NOTE]
> Make sure the path to your python executable inside [`mcp_client.py`](file:///c:/Users/Pratik/Desktop/Tech/Agentic%20AI/TripMate/mcp_client.py#L103) is configured correctly for your environment (especially the path to the interpreter running the `custom_weather_mcp_server.py`).

---

## 🚀 Running the Application

### 1. Start the FastAPI Server
Run the FastAPI application from the project root:
```bash
python app.py
```
By default, the server runs on `http://127.0.0.1:8000`.

### 2. Access the Web Interface
Open your browser and navigate to `http://127.0.0.1:8000/`.

---

## ⚙️ How it works: Under the Hood

### Pausing & Resuming State
The application implements state persistence using `PostgresSaver`. 
- When the graph reaches the `human_approval` node, it calls the `interrupt()` function and saves the state to PostgreSQL mapped to a unique `thread_id`.
- The FastAPI frontend displays the intermediate draft and approval form.
- When the user clicks **Approve** or **Submit Revisions**, the frontend sends a POST request to `/api/travel/approve`, invoking `resume_travel_agent` which feeds the user's action back into the LangGraph engine to resume processing.

### MCP (Model Context Protocol)
The system leverages the Model Context Protocol to seamlessly interface with external data APIs:
* **Tavily MCP** is run as a `streamable_http` client.
* **AviationStack MCP** is run as a subprocess using `uvx aviationstack-mcp`.
* **Weather** is run as a local `stdio` server launching [`custom_weather_mcp_server.py`](file:///c:/Users/Pratik/Desktop/Tech/Agentic%20AI/TripMate/custom_weather_mcp_server.py).
