# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

GPT Researcher MCP Server - A Python MCP (Model Context Protocol) server that provides AI assistants with deep web research capabilities. Uses FastMCP framework to expose GPT Researcher functionality via the MCP protocol.

## Build & Run Commands

```bash
# Install dependencies
pip install -r requirements.txt

# Copy environment template and add API keys
cp .env.example .env

# Run locally (uses STDIO transport for Claude Desktop)
python server.py

# Run with Docker (auto-uses SSE transport)
docker-compose up -d

# Run tests (requires server running on port 8000)
python tests/test_mcp_server.py
```

## Required Environment Variables

**LLM Provider (choose one):**
- `OPENAI_API_KEY` - OpenAI API key, OR
- Bedrock config: `FAST_LLM`, `SMART_LLM`, `STRATEGIC_LLM` with `bedrock:` prefix + AWS credentials

**Web Search (always required):**
- `TAVILY_API_KEY` - Tavily web search API key

**Optional:**
- `MCP_TRANSPORT` - Force transport: `stdio`, `sse`, or `streamable-http`
- `MAX_ITERATIONS` - Research depth iterations (default: 2)

## LLM Provider Configuration

This server supports two LLM providers. Choose one based on your needs:

### Option 1: OpenAI (Default)

```bash
OPENAI_API_KEY=your-key-here
```

### Option 2: AWS Bedrock

AWS Bedrock allows using Claude models via AWS infrastructure. Requires:

1. **LLM Configuration** in `.env`:
```bash
FAST_LLM=bedrock:anthropic.claude-3-sonnet-20240229-v1:0
SMART_LLM=bedrock:anthropic.claude-3-sonnet-20240229-v1:0
STRATEGIC_LLM=bedrock:anthropic.claude-3-sonnet-20240229-v1:0
EMBEDDING=bedrock:amazon.titan-embed-text-v2:0
```

2. **AWS Credentials** via one of:
   - Environment variables: `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, `AWS_DEFAULT_REGION`
   - AWS CLI profile: `AWS_PROFILE=profile-name`
   - IAM role (automatic on EC2/ECS/Lambda)

3. **Bedrock Model Access**: Ensure the models are enabled in your AWS account's Bedrock console.

Available Bedrock Claude models:
- `anthropic.claude-3-haiku-20240307-v1:0` - Fastest, cheapest
- `anthropic.claude-3-sonnet-20240229-v1:0` - Balanced
- `anthropic.claude-3-5-sonnet-20240620-v1:0` - Claude 3.5 Sonnet
- `anthropic.claude-3-opus-20240229-v1:0` - Most capable

**Note:** Do NOT set `OPENAI_API_KEY` when using Bedrock exclusively.

## Architecture

### Core Components

- **server.py** - Main MCP server implementation. Registers tools, resources, and prompts with FastMCP. Auto-detects Docker environment to choose transport (STDIO for local, SSE for Docker).

- **utils.py** - Utility functions: response formatting (`create_success_response`, `create_error_response`), research result storage (`research_store` dict), source formatting helpers.

### MCP Tools Exposed

| Tool | Purpose |
|------|---------|
| `deep_research(query)` | Comprehensive web research (~30-40 seconds) |
| `quick_search(query)` | Fast web search for speed over quality |
| `write_report(research_id, custom_prompt)` | Generate report from research |
| `get_research_sources(research_id)` | Retrieve source metadata |
| `get_research_context(research_id)` | Get full research context |

### Data Flow

1. MCP client connects via transport (STDIO/SSE)
2. Tool handler in `server.py` processes request
3. `GPTResearcher` instance conducts research via Tavily/web retrieval
4. Results cached in `mcp.researchers` dict by UUID and `research_store` by topic
5. Response returned to client

### Transport Modes

- **STDIO** (default) - For Claude Desktop and local MCP clients
- **SSE** - Auto-enabled in Docker, for web clients and n8n integration
- **Streamable HTTP** - For modern web deployments

Transport auto-detection: checks for `/.dockerenv` file or `DOCKER_CONTAINER` env var.

## Testing

Tests in `tests/test_mcp_server.py` validate SSE connection, session management, MCP initialization, and tool execution. Requires server running:

```bash
# Terminal 1: Start server with SSE transport
MCP_TRANSPORT=sse python server.py

# Terminal 2: Run tests
python tests/test_mcp_server.py
```

## Key Implementation Details

- Research results stored in-memory (no persistent database)
- Each research gets a UUID stored in `mcp.researchers` dict
- Topics also cached in `research_store` for resource API access
- Health check endpoint at `/health` for Docker healthchecks
- Python 3.11+ required due to gpt-researcher dependency
