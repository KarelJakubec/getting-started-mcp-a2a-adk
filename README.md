# Getting Started with MCP & A2A with ADK

[![Python](https://img.shields.io/badge/Python-3.10+-blue.svg)](https://www.python.org/)
[![Framework](https://img.shields.io/badge/Framework-ADK-4285F4.svg)](https://github.com/google/adk-python)
[![Protocol](https://img.shields.io/badge/Protocol-A2A-34A853.svg)](https://github.com/google-a2a/a2a-python)
[![Protocol](https://img.shields.io/badge/Protocol-MCP-EA4335.svg)](https://modelcontextprotocol.io/)
[![License](https://img.shields.io/badge/License-Apache_2.0-blue.svg)](LICENSE)

A sample multi-agent system demonstrating **Model Context Protocol (MCP)** and 
**Agent2Agent (A2A)** working together with **Agent Development Kit (ADK)**. 
All services can be run completely locally or fully deployed to Google Cloud Run.

![Architecture Overview](images/architecture.png)

---

## Overview

The sample aims at laying out a foundation and showcasing the capabilities
of MCP + A2A + ADK.

### <img height="20" width="20" src="images/mcp-favicon.ico" alt="MCP Logo" /> Model Context Protocol (MCP)

> MCP is an open protocol that standardizes how applications provide context to LLMs. Think of MCP like a USB-C port for AI applications. Just as USB-C provides a standardized way to connect your devices to various peripherals and accessories, MCP provides a standardized way to connect AI models to different data sources and tools. - [Anthropic](https://modelcontextprotocol.io/introduction)

The MCP server in this example exposes a tool `get_exchange_rate` that can be used to get the exchange rate between two currencies such as USD and EUR. It leverages the [Frankfurter](https://www.frankfurter.dev/) API to get the currency exchange rate. Our agent uses an MCP client to invoke this tool when needed.

### <img height="20" width="20" src="https://a2a-protocol.org/v0.2.5/assets/a2a-logo-black.svg" alt="A2A Logo" /> Agent2Agent (A2A)

> Agent2Agent (A2A) protocol addresses a critical challenge in the AI landscape: enabling gen AI agents, built on diverse frameworks by different companies running on separate servers, to communicate and collaborate effectively - as agents, not just as tools. A2A aims to provide a common language for agents, fostering a more interconnected, powerful, and innovative AI ecosystem. - [A2A](https://github.com/a2aproject/A2A)

In this sample, ADK is used to expose `currency_agent` as an A2A server (`to_a2a`) and consume it from `travel_agent` as a remote A2A agent (`RemoteA2aAgent`).

### <img height="20" width="20" src="images/adk-favicon.ico" alt="ADK Logo" /> Agent Development Kit (ADK)

> ADK is a flexible and modular framework for developing and deploying AI agents. While optimized for Gemini and the Google ecosystem, ADK is model-agnostic, deployment-agnostic, and is built for compatibility with other frameworks. - [ADK](https://github.com/google/adk-python)

ADK is used as the orchestration framework for creating our agents in this sample. It handles the conversation with the user, serves the Web UI, invokes our MCP tool when needed, and handles the A2A communication.

## 🏗️ Architecture Overview

The system consists of 1 orchestrating agent with Web UI, 1 remote agent via A2A, 1 local agent tool, and 1 MCP server:

```
+-----------------------------------------------------------------------------------+
|                                  Clients                                          |
|                     ADK Web UI (Browser)  |  HTTP Callers                         |
+------------------------------------------+----------------------------------------+
                                           | (HTTP / Web UI)
                                           v
+-----------------------------------------------------------------------------------+
| travel_agent (Port 8082 / Cloud Run)                                              |
| - Orchestrating ADK Agent serving the ADK Web UI                                  |
|                                                                                   |
|   +--> [Local AgentTool] weather_agent (travel_agent/subagents/weather_agent.py)  |
|        - Directly wrapped as AgentTool (no A2A network overhead)                  |
|        - Live weather tool via wttr.in API with fallback                          |
|   +--> [Remote AgentTool] currency_agent                                          |
|        - Communicates over A2A protocol                                           |
+------------------------------------------+----------------------------------------+
                                           | (A2A Protocol / JSON-RPC)
                                           v
+-----------------------------------------------------------------------------------+
| currency_agent (Port 8081 / Cloud Run)                                            |
| - Specialized ADK Agent exposed via A2A (to_a2a)                                  |
| - Consumes get_exchange_rate tool via FastMCP Streamable HTTP client              |
+------------------------------------------+----------------------------------------+
                                           | (MCP Streamable HTTP /mcp)
                                           v
+-----------------------------------------------------------------------------------+
| currency_mcp_server (Port 8080 / Cloud Run)                                       |
| - FastMCP server providing real-time exchange rates via Frankfurter API           |
+-----------------------------------------------------------------------------------+
```

### Key Components

- **`currency_mcp_server/`**: A FastMCP server exposing the `get_exchange_rate` tool over Streamable HTTP (`/mcp`), backed by the public [Frankfurter API](https://api.frankfurter.dev/).
- **`currency_agent/`**: An ADK agent connected to the MCP server. Exposed as an A2A service (`to_a2a`).
- **`travel_agent/`**: A travel assistant ADK agent serving the ADK Web UI. It coordinates between:
  - **`weather_agent`** (in `travel_agent/subagents/`): A **local agent wrapped as an `AgentTool`**, demonstrating in-process agent tool usage without A2A network hops.
  - **`currency_agent`**: A **remote agent wrapped as an `AgentTool`**, delegating requests over the A2A protocol.

---

## 📦 Dependency Management (`uv` Workspace)

Project dependencies are organized as a unified **`uv` Workspace**:
- Root `pyproject.toml` orchestrates workspace members (`currency_mcp_server`, `currency_agent`, `travel_agent`).
- Running `uv sync` at the root automatically resolves and installs all packages for local development into a single `.venv`.
- Each service directory has its own self-contained `pyproject.toml` and `Dockerfile` for independent Cloud Run builds.

---

## 🚀 Getting Started

### Prerequisites

- Python 3.10+
- [uv](https://docs.astral.sh/uv/getting-started/installation):
  ```bash
  # macOS / Linux
  curl -LsSf https://astral.sh/uv/install.sh | sh
  ```
- Google Cloud SDK (`gcloud`) if deploying to Cloud Run.

### Installation

1. Clone repository:
   ```bash
   git clone https://github.com/meteatamel/getting-started-mcp-a2a-adk.git
   cd getting-started-mcp-a2a-adk
   ```

2. Install all dependencies across the workspace:
   ```bash
   uv sync
   ```

3. Configure Environment Variables:
   Create a `.env` file in the project root.

   **Option A: Google AI Studio**
   ```sh
   GOOGLE_API_KEY=<your_api_key_here>
   GOOGLE_GENAI_USE_ENTERPRISE=FALSE
   ```

   **Option B: Gemini Enterprise (Google Cloud)**
   ```sh
   GOOGLE_GENAI_USE_ENTERPRISE=TRUE
   GOOGLE_CLOUD_PROJECT=<your_gcp_project_id>
   GOOGLE_CLOUD_LOCATION=global
   ```

---

## 💻 Local Execution

You can run all three services concurrently in separate terminal windows. Make
sure to run them in the order of MCP server first, then currency agent, then
travel agent.

### Step 1: Start Currency MCP Server

Start MCP server locally in terminal1:

```bash
uv run python currency_mcp_server/server.py
```

### Step 2: Start Currency Agent

Start agent locally in terminal2:

```bash
uv run python currency_agent/agent.py
```

### Step 3: Start Travel Agent

Start travel agent locally in terminal3 (starts the ADK Web UI on port 8082):

```bash
uv run python travel_agent/agent.py
```

*(Alternatively, you can run `uv run adk web travel_agent` to launch it on port 8000).*

### 🧪 Testing

Test the MCP server:

```bash
uv run python currency_mcp_server/test_server.py
```

Test the currency agent (over A2A protocol):

```bash
uv run python currency_agent/test_a2aclient.py
```

Test the travel agent (Interactive ADK Web UI):

Open your browser at `http://localhost:8082` (or `http://localhost:8000` if using `adk web`) and chat with `travel_agent`:
> *"I'm planning a trip to London. What is the weather like and how much is 200 EUR in GBP?"*

This runs end-to-end:
1. Queries travel and currency conversion (delegated via A2A to `currency_agent` -> `currency_mcp_server`).
2. Queries weather forecasts (delegated to local `weather_agent` in `travel_agent/subagents/`).

You can also test queries directly from the command line:

```bash
uv run adk run travel_agent "What is the weather in London and how much is 100 USD in EUR?"
```

---

## ☁️ Cloud Run Deployment

All three components include optimized Dockerfiles and can be deployed directly from source to Cloud Run. You can use `gcloud` to automatically capture service URLs and wire them into the next steps without manual copy-pasting.

First, set some environment variables for your Google Cloud project and Cloud Run region:

```bash
export PROJECT_ID=<YOUR_GOOGLE_CLOUD_PROJECT_ID>
export REGION=us-central1
```

Enable the required Google Cloud APIs for Cloud Run, Cloud Build, Artifact Registry, and Vertex AI:

```bash
gcloud services enable run.googleapis.com \
                       cloudbuild.googleapis.com \
                       artifactregistry.googleapis.com \
                       aiplatform.googleapis.com
```

> [!WARNING]
> For simplicity in this demo, services are deployed with `--allow-unauthenticated`. In a production system, internal services should enforce authentication with `--no-allow-unauthenticated` using Cloud Run Invoker (`roles/run.invoker`) IAM roles, service accounts, or an API gateway.

### Step 1: Deploy Currency MCP Server

```bash
# Deploy MCP server
gcloud run deploy currency-mcp-server \
  --source currency_mcp_server \
  --region $REGION \
  --allow-unauthenticated

# Capture the deployed MCP server URL
MCP_SERVER_URL=$(gcloud run services describe currency-mcp-server --region $REGION --format='value(status.url)')/mcp
echo "MCP Server URL: $MCP_SERVER_URL"
```

### Step 2: Deploy Currency Agent

Since `currency-agent`'s public URL is only generated upon its first deployment, deploy the service first, capture its URL, and then set `AGENT_URL` via a fast configuration update:

```bash
# 1. Deploy Currency Agent with the MCP Server URL
gcloud run deploy currency-agent \
  --source currency_agent \
  --region $REGION \
  --allow-unauthenticated \
  --update-env-vars MCP_SERVER_URL="$MCP_SERVER_URL",GOOGLE_GENAI_USE_ENTERPRISE="true",GOOGLE_CLOUD_PROJECT="$PROJECT_ID",GOOGLE_CLOUD_LOCATION="global"

# 2. Capture its assigned Cloud Run URL
CURRENCY_AGENT_URL=$(gcloud run services describe currency-agent --region $REGION --format='value(status.url)')
echo "Currency Agent URL: $CURRENCY_AGENT_URL"

# 3. Update AGENT_URL so the agent advertises its public HTTPS endpoint in its Agent Card
gcloud run services update currency-agent \
  --region $REGION \
  --update-env-vars AGENT_URL="$CURRENCY_AGENT_URL"
```

### Step 3: Deploy Travel Agent

Deploy `travel-agent` connected to `CURRENCY_AGENT_URL`:

```bash
# 1. Deploy Travel Agent (with ADK Web UI) connected to Currency Agent
gcloud run deploy travel-agent \
  --source travel_agent \
  --region $REGION \
  --allow-unauthenticated \
  --update-env-vars CURRENCY_AGENT_URL="$CURRENCY_AGENT_URL",GOOGLE_GENAI_USE_ENTERPRISE="true",GOOGLE_CLOUD_PROJECT="$PROJECT_ID",GOOGLE_CLOUD_LOCATION="global"

# 2. Capture its assigned Cloud Run URL
TRAVEL_AGENT_URL=$(gcloud run services describe travel-agent --region $REGION --format='value(status.url)')
echo "Travel Agent URL: $TRAVEL_AGENT_URL"
```

> [!TIP]
> You can also deploy `travel_agent` using ADK's `adk deploy cloud_run` with the `--with_ui` flag:
> ```bash
> uv run adk deploy cloud_run travel_agent \
>   --project $PROJECT_ID \
>   --region $REGION \
>   --with_ui \
>   --env CURRENCY_AGENT_URL="$CURRENCY_AGENT_URL"
> ```

### 🧪 Testing

Test the MCP server:

```bash
MCP_SERVER_URL="$MCP_SERVER_URL" uv run python currency_mcp_server/test_server.py
```

Test the currency agent (over A2A protocol):

```bash
AGENT_URL="$CURRENCY_AGENT_URL" uv run python currency_agent/test_a2aclient.py
```

Test the travel agent (Directly in Cloud Run via Web UI):

Simply open `$TRAVEL_AGENT_URL` in your browser! The ADK Web UI is served directly from Cloud Run:
> *"I'm planning a trip to Tokyo. What is the weather like and how much is 500 USD in JPY?"*

This tests the complete end-to-end multi-agent system running in Cloud Run:
1. The **ADK Web UI** loaded from `travel-agent` on Cloud Run.
2. In-process execution of the local `weather_agent` tool in Cloud Run.
3. Remote A2A invocation across Cloud Run to `currency-agent`.
4. Remote MCP invocation across Cloud Run to `currency-mcp-server`.

---

## 📄 License

This project is licensed under the Apache 2.0 License - see the [LICENSE](LICENSE) file for details.
