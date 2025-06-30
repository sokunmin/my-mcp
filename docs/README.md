# Weather Forecast MCP Server Documentation

This directory contains comprehensive documentation for the Weather Forecast MCP Server project - a pure Python implementation that provides real-time weather data to LLM clients through the Model Context Protocol (MCP).

## Documentation Structure

### Core Documentation
- **[Project Overview](./01-project-overview.md)** - High-level project description, goals, and architecture
- **[Technical Architecture](./02-technical-architecture.md)** - Detailed system design and component relationships
- **[Implementation Plan](./03-implementation-plan.md)** - Step-by-step development roadmap with phases
- **[API Specification](./04-api-specification.md)** - Complete MCP tools and data models specification

### Deployment Guides
- **[Local Development](./05-local-development.md)** - Setting up and running locally with uv venv
- **[Docker Deployment](./06-docker-deployment.md)** - Containerized deployment configuration
- **[Cloudflare Workers](./07-cloudflare-deployment.md)** - Native Python Workers deployment

### Development Resources
- **[Development Guidelines](./08-development-guidelines.md)** - Coding standards, testing, and best practices
- **[CI/CD Pipeline](./09-cicd-pipeline.md)** - Automated testing and deployment workflows
- **[Troubleshooting](./10-troubleshooting.md)** - Common issues and solutions

## Quick Start for AI Agents

If you're an AI agent tasked with implementing this project:

1. **Start with [Project Overview](./01-project-overview.md)** to understand the goals and scope
2. **Review [Technical Architecture](./02-technical-architecture.md)** to understand the system design
3. **Follow [Implementation Plan](./03-implementation-plan.md)** for step-by-step execution
4. **Reference [API Specification](./04-api-specification.md)** for exact implementation details
5. **Use deployment guides** for your target environment

## Key Principles

- **Pure Python**: No JavaScript, single language across all deployments
- **No Caching**: Direct API calls to open-meteo.com for real-time data
- **Simple Architecture**: Stateless MCP server with direct API integration
- **Multiple Deployments**: Same codebase works locally, in Docker, and on Cloudflare Workers

## Project Status

This is a planning document. Implementation should follow the phases outlined in the Implementation Plan, starting with Phase 1: Core Foundation.