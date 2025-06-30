# Development Project

## Project Overview
Developing MCP server using the latest FastMCP version, Python 3.12, and a uv venv virtual environment.

## Environment Setup

### System Environment
- OS: Windows
- Terminal: PowerShell

### Virtual Environment
- Python Version: 3.12
- Global venv path(VIRTUAL_ENV env var): D:\rovo_workspace\global-mcp-venv\

## Project Structure
```
Project_Root/
├── docs/
|    └── llms-fastmcp.txt
└── .agent.md         # This configuration file
```


## Documentation and Resource Acquisition

### Search Strategy
Always search for the latest FastMCP documentation and coding standards:
- Prioritize querying the official FastMCP documentation from `llms-fastmcp.txt`
- Obtain the latest API references and best practices
- Find code examples and patterns
- Verify version compatibility and new features
- Use `Context7 MCP server` to search more information if you cannot find any related information.
- Prioritize query the official Cloudflare documentation for Cloudflare configuration/development using `cloudflare` MCP server.


## Notes
- Use FastMCP 2.0
- MCP server is suitable for local/docker/cloudflare deployment.
- Requires Python 3.12+
- All example code uses English comments.
- When encountering problems, prioritize searching for the latest solutions via context7.
- Do not over-engineer.

## AI Agent Interaction and Documentation Workflow
- Any new or modified requirements must first be planned for developer reference. If the developer confirms it is executable, the `PLAN.md` file will be automatically updated.
- Whenever the developer says "update memory", all current discussion content will be automatically updated to `.agent.md`. The updated content must provide detailed explanations, allowing other AI agents to fully understand the entire project's context from scratch.

