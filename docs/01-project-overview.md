# Project Overview

## Project Description

The Weather Forecast MCP Server is a pure Python implementation of a Model Context Protocol (MCP) server that provides real-time weather data to Large Language Models (LLMs). The server acts as a bridge between LLMs and the open-meteo.com weather API, enabling AI assistants to access current weather conditions, forecasts, and location data.

## Goals and Objectives

### Primary Goals
- **Real-time Weather Access**: Provide LLMs with current, accurate weather information
- **Multi-deployment Support**: Enable deployment across local, Docker, and Cloudflare Workers environments
- **Pure Python Implementation**: Maintain a single-language codebase for simplicity and maintainability
- **Zero Caching Complexity**: Direct API calls for always-fresh weather data

### Secondary Goals
- **Open Source Compatibility**: Use free, open APIs (open-meteo.com)
- **Production Ready**: Include proper error handling, logging, and monitoring
- **Developer Friendly**: Clear documentation and easy local development setup
- **CI/CD Automation**: Automated testing and deployment pipelines

## Target Users

### Primary Users
- **LLM Applications**: AI assistants, chatbots, and automated systems requiring weather data
- **Developers**: Building weather-aware applications with MCP-compatible LLMs
- **AI Researchers**: Experimenting with weather-context in AI applications

### Use Cases
- **Personal Assistants**: "What's the weather like in Tokyo today?"
- **Travel Planning**: "Should I pack a jacket for my trip to London next week?"
- **Event Planning**: "Will it rain during our outdoor event on Saturday?"
- **Agricultural Applications**: "What's the forecast for the next 7 days in Iowa?"
- **Emergency Preparedness**: "Are there any weather alerts for my area?"

## Key Features

### Weather Tools
1. **Current Weather**: Real-time conditions for any location
2. **Weather Forecasts**: Daily forecasts up to 16 days ahead
3. **Hourly Forecasts**: Detailed hourly data up to 7 days
4. **Location Search**: Geocoding to find coordinates from place names
5. **Weather Alerts**: Active warnings and advisories

### Technical Features
- **MCP Protocol Compliance**: Full compatibility with MCP clients
- **HTTP/SSE Transport**: Support for both HTTP and Server-Sent Events
- **Error Handling**: Robust error management and user-friendly messages
- **Environment Configuration**: Flexible configuration for different deployment environments
- **Comprehensive Testing**: Unit, integration, and deployment tests

## Technology Stack

### Core Technologies
- **Python 3.12+**: Primary programming language
- **FastMCP 2.9+**: MCP server framework
- **Pydantic**: Data validation and serialization
- **httpx**: Async HTTP client for API calls
- **open-meteo.com**: Free weather API service

### Development Tools
- **uv**: Python package and environment management
- **pytest**: Testing framework
- **ruff**: Code linting and formatting
- **mypy**: Static type checking
- **GitHub Actions**: CI/CD automation

### Deployment Platforms
- **Local Development**: uv venv with stdio transport
- **Docker**: Containerized deployment with HTTP transport
- **Cloudflare Workers**: Native Python Workers runtime

## Architecture Principles

### Design Principles
1. **Simplicity First**: Avoid over-engineering and unnecessary complexity
2. **Stateless Design**: No persistent state or caching between requests
3. **Direct Integration**: Minimal layers between MCP client and weather API
4. **Environment Agnostic**: Same core code works across all deployment targets
5. **Real-time Data**: Always fetch fresh data from the source API

### Non-Goals
- **Data Caching**: No caching layer to avoid stale data
- **Historical Data**: Focus on current and forecast data only
- **Multiple Weather Sources**: Single, reliable API source
- **Complex Authentication**: Leverage open-meteo.com's free tier
- **Data Processing**: Minimal transformation, pass through API responses

## Success Criteria

### Functional Requirements
- [ ] All weather tools return accurate, real-time data
- [ ] MCP clients can successfully connect and make requests
- [ ] Error handling provides clear, actionable messages
- [ ] All deployment methods work seamlessly
- [ ] Response times under 2 seconds for typical requests

### Technical Requirements
- [ ] Pure Python implementation (no JavaScript dependencies)
- [ ] Test coverage above 90%
- [ ] Automated CI/CD pipeline operational
- [ ] Documentation complete and accurate
- [ ] Code passes all linting and type checking

### Operational Requirements
- [ ] Local development setup works in under 5 minutes
- [ ] Docker deployment completes successfully
- [ ] Cloudflare Workers deployment from GitHub works
- [ ] All environments handle errors gracefully
- [ ] Monitoring and logging provide adequate visibility

## Project Constraints

### Technical Constraints
- **Python Only**: No JavaScript or other languages in the core implementation
- **Open-Meteo API**: Must use open-meteo.com as the weather data source
- **MCP Protocol**: Must comply with MCP specification
- **No Caching**: Direct API calls only, no intermediate caching

### Resource Constraints
- **Free Tier APIs**: Must work within open-meteo.com's free usage limits
- **Cloudflare Limits**: Must work within Workers free tier constraints
- **Development Time**: Prioritize core functionality over advanced features

### Operational Constraints
- **No Authentication**: Leverage open APIs to avoid auth complexity
- **Minimal Dependencies**: Keep dependency tree small and manageable
- **Cross-Platform**: Must work on Windows, macOS, and Linux

## Risk Assessment

### Technical Risks
- **API Availability**: open-meteo.com service interruptions
- **Rate Limiting**: Potential API rate limits under high usage
- **Cloudflare Changes**: Changes to Python Workers support
- **FastMCP Updates**: Breaking changes in FastMCP framework

### Mitigation Strategies
- **Error Handling**: Comprehensive error handling for API failures
- **Documentation**: Clear error messages for troubleshooting
- **Testing**: Extensive testing to catch issues early
- **Monitoring**: Observability to detect and diagnose issues

## Next Steps

1. **Review Technical Architecture**: Understand the system design and component relationships
2. **Study Implementation Plan**: Follow the phased development approach
3. **Set Up Development Environment**: Prepare local development tools
4. **Begin Phase 1**: Start with core foundation implementation

For detailed technical information, proceed to [Technical Architecture](./02-technical-architecture.md).