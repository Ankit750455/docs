# ADK Agent Engine Deployment Guide

**Google Agent Development Kit (ADK)**  
**Cognizant Technology Solutions**  
**Version 3.0 | January 2026**

---

## 📋 Table of Contents

1. [Overview](#overview)
2. [Prerequisites](#prerequisites)
3. [Agent Preparation for Deployment](#agent-preparation-for-deployment)
4. [CLI Deployment](#cli-deployment)
5. [Interacting with Your Deployed Agent](#interacting-with-your-deployed-agent)
6. [Deployment Patterns for MCP Agents](#deployment-patterns-for-mcp-agents)
7. [Comparison: Agent Engine vs Cloud Run vs GKE](#comparison-agent-engine-vs-cloud-run-vs-gke)
8. [Troubleshooting](#troubleshooting)

---

## 🎯 Overview

### What is Vertex AI Agent Engine?

**Vertex AI Agent Engine** (formerly known as Reasoning Engine) is a **fully managed runtime** from Google Cloud, specifically optimized for AI agents. Unlike Cloud Run (which runs generic containers), Agent Engine:

- **Handles Infrastructure**: Auto-scaling, load balancing, and security are managed for you
- **Provides Session Management**: Built-in conversation memory and state persistence
- **Offers Native Integration**: Tight integration with Vertex AI services (RAG, Search, Monitoring)
- **Is Headless**: No web UI - it provides a secure **REST API endpoint** that your applications consume

### When to Use Agent Engine

| Use Case                            | Recommendation                        |
| ----------------------------------- | ------------------------------------- |
| Backend API for mobile/web apps     | ✅ **Agent Engine**                   |
| Standalone agent with web UI        | ❌ Use **Cloud Run** with `--with_ui` |
| Enterprise with existing Kubernetes | ❌ Use **GKE**                        |
| Quick prototyping with UI           | ❌ Use local `adk web`                |

### Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                        Your Applications                             │
│   (Mobile App, Dashboard, Internal Tools, Other AI Agents)          │
└─────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼ REST API / SDK
┌─────────────────────────────────────────────────────────────────────┐
│                     Vertex AI Agent Engine                           │
│  ┌─────────────────┐  ┌──────────────────┐  ┌────────────────────┐  │
│  │   Your ADK      │  │  Session/Memory  │  │   IAM & Security   │  │
│  │   Agent Code    │  │   Management     │  │   (Automatic)      │  │
│  └─────────────────┘  └──────────────────┘  └────────────────────┘  │
└─────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼ HTTP (Streamable HTTP/SSE)
┌─────────────────────────────────────────────────────────────────────┐
│                    External MCP Server (Cloud Run)                   │
│                 (Your custom tools, PowerPoint, etc.)                │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 📋 Prerequisites

### 1. Required GCP APIs

Enable these APIs before deployment:

```bash
# Enable required APIs
gcloud services enable aiplatform.googleapis.com \
    storage.googleapis.com \
    cloudresourcemanager.googleapis.com \
    cloudbuild.googleapis.com \
    --project=$GOOGLE_CLOUD_PROJECT
```

| API                                   | Purpose                                 |
| ------------------------------------- | --------------------------------------- |
| `aiplatform.googleapis.com`           | Vertex AI Agent Engine runtime          |
| `storage.googleapis.com`              | Staging bucket for deployment artifacts |
| `cloudresourcemanager.googleapis.com` | Project metadata access                 |
| `cloudbuild.googleapis.com`           | Container building (if needed)          |

### 2. Required IAM Roles

The account running the deployment needs:

| Role                           | Purpose                                  |
| ------------------------------ | ---------------------------------------- |
| `roles/aiplatform.admin`       | Create and manage Agent Engine resources |
| `roles/storage.admin`          | Upload to staging bucket                 |
| `roles/iam.serviceAccountUser` | Use service accounts for deployment      |

```bash
# Grant roles to your user account
gcloud projects add-iam-policy-binding $GOOGLE_CLOUD_PROJECT \
    --member="user:your-email@domain.com" \
    --role="roles/aiplatform.admin"
```

### 3. Staging Bucket

Agent Engine requires a GCS bucket to stage deployment artifacts:

```bash
# Create the staging bucket (one-time setup)
# MUST be in the same region as your Agent Engine deployment
gsutil mb -l us-central1 gs://$GOOGLE_CLOUD_PROJECT-adk-staging

# Verify the bucket exists
gsutil ls gs://$GOOGLE_CLOUD_PROJECT-adk-staging
```

> [!IMPORTANT]
> The staging bucket **must** be in the same region where you deploy the agent. If you deploy to `us-central1`, the bucket must also be in `us-central1`.

### 4. Authentication

```bash
# Authenticate your terminal
gcloud auth login

# Set up Application Default Credentials (for SDK and API calls)
gcloud auth application-default login

# Set your default project
gcloud config set project $GOOGLE_CLOUD_PROJECT

# Verify authentication
gcloud auth list
```

### 5. Environment Variables

Set these before running deployment commands:

```bash
export GOOGLE_CLOUD_PROJECT="your-project-id"
export GOOGLE_CLOUD_LOCATION="us-central1"
export STAGING_BUCKET="gs://$GOOGLE_CLOUD_PROJECT-adk-staging"
```

---

## 🔧 Agent Preparation for Deployment

### Project Structure

Your agent folder must follow this structure:

```
my_agent/
├── __init__.py          # Required: exports root_agent
├── agent.py             # Required: defines root_agent
├── prompt.py            # Optional: agent instructions
├── pyproject.toml       # Required: dependencies (or requirements.txt)
└── .env                 # Optional: environment variables
```

### Critical: The `__init__.py` File

This file **must** export the `root_agent`:

```python
# __init__.py
from .agent import root_agent
```

### Critical: Synchronous Agent Definition

> [!CAUTION] > **Agent Engine requires synchronous agent definition!** Asynchronous patterns like `async def get_agent()` will NOT work.

**✅ CORRECT:**

```python
# agent.py - Synchronous definition
from google.adk.agents import Agent

root_agent = Agent(
    model='gemini-2.5-flash',
    name='my_agent',
    instruction='...',
    tools=[...],
)
```

**❌ WRONG:**

```python
# This pattern DOES NOT work with Agent Engine
async def get_agent():
    toolset = await create_mcp_toolset()
    return Agent(tools=[toolset])
```

### Dependencies: pyproject.toml vs requirements.txt

Agent Engine supports both formats. Using `pyproject.toml` is recommended:

```toml
# pyproject.toml
[build-system]
requires = ["setuptools>=65", "wheel"]
build-backend = "setuptools.build_meta"

[project]
name = "my-agent"
version = "1.0.0"
requires-python = ">=3.11"

dependencies = [
    "google-adk>=1.21.0",
    "google-auth>=2.0.0",
    "google-genai>=1.0.0",
]
```

Alternatively, use a `requirements.txt`:

```text
google-adk>=1.21.0
google-auth>=2.0.0
google-genai>=1.0.0
```

---

## 🚀 CLI Deployment

### The Deployment Command

```bash
adk deploy agent_engine \
  --project=$GOOGLE_CLOUD_PROJECT \
  --region=$GOOGLE_CLOUD_LOCATION \
  --staging_bucket="$STAGING_BUCKET" \
  --display_name="My Agent Display Name" \
  ./my_agent
```

### All Available Flags

| Flag                  | Required | Description                                         |
| --------------------- | -------- | --------------------------------------------------- |
| `--project`           | Yes\*    | GCP project ID                                      |
| `--region`            | Yes\*    | GCP region (e.g., `us-central1`)                    |
| `--staging_bucket`    | Yes\*    | GCS bucket for artifacts (e.g., `gs://my-bucket`)   |
| `AGENT_PATH`          | Yes      | Path to agent directory (positional, at end)        |
| `--display_name`      | No       | Human-readable name in Cloud Console                |
| `--description`       | No       | Description for the agent                           |
| `--env_file`          | No       | Path to `.env` file for environment variables       |
| `--requirements_file` | No       | Path to `requirements.txt` (if not in agent folder) |
| `--agent_engine_id`   | No       | Existing Agent ID to update (for redeployments)     |
| `--trace_to_cloud`    | No       | Enable Cloud Trace for debugging                    |
| `--adk_app`           | No       | Custom Python file defining the app                 |

\*Required unless using Express Mode with `--api_key`

### Deployment Process

When you run the deploy command, ADK performs these steps:

1. **Packaging**: Zips your agent folder (code + dependencies)
2. **Staging**: Uploads the package to your GCS staging bucket
3. **Resource Creation**: Calls Vertex AI API to create a `ReasoningEngine` resource
4. **Provisioning**: Agent Engine provisions compute resources (takes 3-10 minutes)

### Deployment Output

```
Creating AgentEngine
Create AgentEngine backing LRO: projects/123456789/locations/us-central1/reasoningEngines/751619551677906944/operations/2356952072064073728
View progress and logs at https://console.cloud.google.com/logs/query?project=your-project
AgentEngine created. Resource name: projects/123456789/locations/us-central1/reasoningEngines/751619551677906944

To use this AgentEngine in another session:
agent_engine = vertexai.agent_engines.get('projects/123456789/locations/us-central1/reasoningEngines/751619551677906944')

Cleaning up the temp folder: /tmp/agent_engine_deploy_src/20260103_121500
```

> [!TIP]
> Save the `Resource name` - you'll need the **RESOURCE_ID** (e.g., `751619551677906944`) to interact with the agent.

### Updating an Existing Deployment

To update an already-deployed agent:

```bash
adk deploy agent_engine \
  --project=$GOOGLE_CLOUD_PROJECT \
  --region=$GOOGLE_CLOUD_LOCATION \
  --staging_bucket="$STAGING_BUCKET" \
  --agent_engine_id="751619551677906944" \
  ./my_agent
```

---

## 💬 Interacting with Your Deployed Agent

> [!WARNING]
> Agent Engine is **headless** - there is no web UI! You must use the Python SDK or REST API to communicate with your agent.

### Method 1: Python SDK (Recommended)

```python
import vertexai
from vertexai import agent_engines

# Initialize Vertex AI
vertexai.init(
    project="your-project-id",
    location="us-central1"
)

# Get your deployed agent
# Replace with your actual resource name from deployment output
agent = agent_engines.get(
    "projects/your-project-id/locations/us-central1/reasoningEngines/751619551677906944"
)

# Query the agent (stateless - no session)
response = agent.query(input="Hello, what can you do?")
print(response)

# Query with session (maintains conversation history)
response = agent.query(
    input="Create a presentation about AI",
    config={"session_id": "user-123-session-001"}
)
print(response)
```

### Method 2: REST API with curl

```bash
# Set variables
PROJECT_ID="your-project-id"
LOCATION="us-central1"
RESOURCE_ID="751619551677906944"

# Get access token
TOKEN=$(gcloud auth print-access-token)

# Query the agent
curl -X POST \
  "https://${LOCATION}-aiplatform.googleapis.com/v1/projects/${PROJECT_ID}/locations/${LOCATION}/reasoningEngines/${RESOURCE_ID}:query" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "input": {
        "messages": [
            {"role": "user", "parts": [{"text": "Hello, what can you do?"}]}
        ]
    }
  }'
```

### Method 3: Streaming Responses

For real-time streaming responses:

```python
# Streaming query
for chunk in agent.stream_query(input="Write a long story"):
    print(chunk, end="", flush=True)
```

Or via REST:

```bash
curl -X POST \
  "https://${LOCATION}-aiplatform.googleapis.com/v1/projects/${PROJECT_ID}/locations/${LOCATION}/reasoningEngines/${RESOURCE_ID}:streamQuery" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"input": {"messages": [{"role": "user", "parts": [{"text": "Hello"}]}]}}'
```

### Monitoring Your Agent

View logs and metrics in Cloud Console:

```bash
# Open the Agent Engine UI
echo "https://console.cloud.google.com/vertex-ai/agents/agent-engines?project=$GOOGLE_CLOUD_PROJECT"

# View agent logs
gcloud logging read "resource.type=aiplatform.googleapis.com/ReasoningEngine" \
  --project=$GOOGLE_CLOUD_PROJECT \
  --limit=50
```

---

## 🔌 Deployment Patterns for MCP Agents

When your agent uses **MCP (Model Context Protocol)** tools via a remote server, you need special handling for Agent Engine deployment.

### The Problem: Serialization

Agent Engine serializes your agent code for deployment. This means:

1. Code that runs at import time (module-level) gets executed during packaging
2. Non-serializable objects (like network connections) will cause failures
3. Authentication tokens fetched at import time will be stale

### The Solution: Lazy Initialization

> [!IMPORTANT]
> For agents with MCP tools connecting to remote servers, use **factory functions** instead of direct instances.

#### ❌ WRONG: Direct Token Fetching

```python
# This FAILS during Agent Engine deployment!
token = get_auth_token()  # Runs at import time

mcp_connection = StreamableHTTPConnectionParams(
    url=MCP_SERVER_URL,
    headers={"Authorization": f"Bearer {token}"},  # Stale token!
)

root_agent = Agent(
    tools=[McpToolset(connection_params=mcp_connection)],
)
```

#### ✅ CORRECT: Factory Function Pattern

```python
import os
import google.auth.transport.requests
from google.oauth2 import id_token
from google.adk.agents import Agent
from google.adk.tools.mcp_tool import McpToolset
from google.adk.tools.mcp_tool.mcp_session_manager import StreamableHTTPConnectionParams

MCP_SERVER_URL = os.environ.get("MCP_SERVER_URL")
MCP_AUDIENCE = MCP_SERVER_URL.rsplit("/", 1)[0]  # Base URL for auth

def _create_mcp_toolset():
    """Factory function - called at runtime, not import time!"""

    def get_fresh_token():
        try:
            request = google.auth.transport.requests.Request()
            return id_token.fetch_id_token(request, MCP_AUDIENCE)
        except Exception as e:
            print(f"Warning: Could not fetch ID token: {e}")
            return None

    token = get_fresh_token()

    return McpToolset(
        connection_params=StreamableHTTPConnectionParams(
            url=MCP_SERVER_URL,
            timeout=120.0,
            headers={"Authorization": f"Bearer {token}"} if token else None,
        )
    )

# Pass the FUNCTION, not the result!
root_agent = Agent(
    model='gemini-2.5-flash',
    name='my_agent',
    instruction='...',
    tools=[_create_mcp_toolset],  # Note: no parentheses!
)
```

**Key Points:**

- `_create_mcp_toolset` is passed as a **callable**, not invoked
- Token fetching happens at **runtime** when the agent processes a request
- Each request gets a fresh token, avoiding staleness

### MCP Deployment Patterns Comparison

| Pattern                  | Best For                 | Connection Type                  |
| ------------------------ | ------------------------ | -------------------------------- |
| **Remote MCP Server**    | Production, scalable     | `StreamableHTTPConnectionParams` |
| **Self-Contained Stdio** | Simple tools, filesystem | `StdioConnectionParams`          |
| **Sidecar (GKE only)**   | Kubernetes deployments   | Local HTTP                       |

#### Pattern A: Remote MCP Server (Recommended for Production)

```python
# Agent connects to MCP server deployed on Cloud Run
McpToolset(
    connection_params=StreamableHTTPConnectionParams(
        url="https://my-mcp-server.run.app/mcp",
        headers={"Authorization": f"Bearer {token}"},
        timeout=120.0,
    )
)
```

#### Pattern B: Self-Contained Stdio (Works with Agent Engine)

```python
# MCP server runs as subprocess within the agent container
# Requires Node.js installed in Agent Engine environment
McpToolset(
    connection_params=StdioConnectionParams(
        server_params=StdioServerParameters(
            command='npx',
            args=['-y', '@modelcontextprotocol/server-filesystem', '/tmp'],
        ),
        timeout=30.0,
    )
)
```

> [!WARNING]
> Stdio pattern requires the MCP server's runtime (Node.js, Python, etc.) to be available in the Agent Engine environment. This may require custom build scripts.

---

## 📊 Comparison: Agent Engine vs Cloud Run vs GKE

| Feature                |  Agent Engine  |      Cloud Run       |        GKE         |
| ---------------------- | :------------: | :------------------: | :----------------: |
| **Web UI Support**     |       ❌       |   ✅ (`--with_ui`)   |         ✅         |
| **Setup Complexity**   |    Very Low    |         Low          |        High        |
| **Scaling**            | Auto (managed) |    Auto (to zero)    |   Manual config    |
| **Session Management** |  ✅ Built-in   |  ❌ External needed  | ❌ External needed |
| **Custom Runtime**     |    Limited     |     Full Docker      |  Full Kubernetes   |
| **Cost Model**         |  Per-request   | Per-container-second |      Per-node      |
| **Best For**           |  API backends  |   Standalone apps    |     Enterprise     |

### Decision Tree

```
Do you need a Web UI?
├─ Yes → Use Cloud Run with --with_ui
└─ No → Is this for production API access?
         ├─ Yes → Use Agent Engine
         └─ No → Do you need full infrastructure control?
                  ├─ Yes → Use GKE
                  └─ No → Use Cloud Run (API only)
```

---

## 🔧 Troubleshooting

### Deployment Hangs or Times Out

**Cause**: Usually network issues or missing permissions.

**Solution**:

```bash
# Check if APIs are enabled
gcloud services list --enabled --project=$GOOGLE_CLOUD_PROJECT | grep -E "(aiplatform|storage)"

# Verify bucket is accessible
gsutil ls $STAGING_BUCKET

# Check IAM permissions
gcloud projects get-iam-policy $GOOGLE_CLOUD_PROJECT \
  --filter="bindings.members:user:$(gcloud config get-value account)"
```

### "TypeError: cannot pickle" Error

**Cause**: Your agent code has non-serializable objects at module level.

**Solution**: Use factory functions for MCP toolsets and other runtime objects. See [Deployment Patterns for MCP Agents](#deployment-patterns-for-mcp-agents).

### Agent Deployed but Tools Don't Work

**Cause**: MCP server authentication failing or URL incorrect.

**Solution**:

1. Verify the MCP server is accessible:
   ```bash
   curl -I https://your-mcp-server.run.app/health
   ```
2. Check Agent Engine has permission to call MCP server:
   ```bash
   # Grant the Agent Engine service account access
   gcloud run services add-iam-policy-binding YOUR_MCP_SERVICE \
     --member="serviceAccount:service-PROJECT_NUMBER@gcp-sa-aiplatform-re.iam.gserviceaccount.com" \
     --role="roles/run.invoker" \
     --region=$GOOGLE_CLOUD_LOCATION
   ```

### "Permission Denied" on Query

**Cause**: Your user/service account lacks `roles/aiplatform.user`.

**Solution**:

```bash
gcloud projects add-iam-policy-binding $GOOGLE_CLOUD_PROJECT \
  --member="user:your-email@domain.com" \
  --role="roles/aiplatform.user"
```

### Environment Variables Not Available

**Cause**: `.env` file not included in deployment.

**Solution**: Explicitly specify the env file:

```bash
adk deploy agent_engine \
  --env_file="./my_agent/.env" \
  ...
```

Or set variables in Cloud Console after deployment.

---

## 📚 Further Resources

- [Official ADK Documentation](https://google.github.io/adk-docs/)
- [Agent Engine Deployment Guide](https://google.github.io/adk-docs/deploy/agent-engine/deploy/)
- [MCP Tools Documentation](https://google.github.io/adk-docs/tools-custom/mcp-tools/)
- [Vertex AI Agent Engine Pricing](https://cloud.google.com/vertex-ai/pricing#vertex-ai-agent-engine)
- [Agent Engine REST API Reference](https://cloud.google.com/vertex-ai/generative-ai/docs/agent-engine/reference/rest)

---

**Last Updated**: January 03, 2026  
**Author**: Ankit Goel (2179009) | ankit.goel@cognizant.com
