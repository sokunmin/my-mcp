# Technical Architecture

## System Overview

The Weather Forecast MCP Server follows a clean, layered architecture designed for simplicity and maintainability. The system consists of three main layers: the MCP Protocol Layer, the Business Logic Layer, and the External API Layer.

```
┌─────────────────────────────────────────────────────────────────┐
│                    MCP CLIENT (LLM)                            │
└─────────────────────────┬───────────────────────────────────────┘
                          │ MCP Protocol (HTTP/SSE/stdio)
┌─────────────────────────▼───────────────────────────────────────┐
│                  WEATHER MCP SERVER                            │
├─────────────────────────────────────────────────────────────────┤
│  MCP Protocol Layer                                            │
│  ├── FastMCP Framework                                         │
│  ├── Transport Handlers (stdio/http/sse)                      │
│  └── Request/Response Serialization                           │
├─────────────────────────────────────────────────────────────────┤
│  Business Logic Layer                                          │
│  ├── Weather Tools (get_current_weather, etc.)                │
│  ├── Data Models (Pydantic)                                   │
│  ├── Input Validation                                         │
│  └── Error Handling                                           │
├─────────────────────────────────────────────────────────────────┤
│  External API Layer                                           │
│  ├── Open-Meteo API Client                                    │
│  ├── HTTP Client (httpx)                                      │
│  └── Response Parsing                                         │
└─────────────────────────┬───────────────────────────────────────┘
                          │ HTTPS
┌─────────────────────────▼───────────────────────────────────────┐
│                  OPEN-METEO.COM API                           │
│  ├── Current Weather Endpoint                                 │
│  ├── Forecast Endpoint                                        │
│  ├── Hourly Endpoint                                          │
│  └── Geocoding Endpoint                                       │
└─────────────────────────────────────────────────────────────────┘
```

## Core Components

### 1. MCP Protocol Layer

**Purpose**: Handle MCP protocol communication with LLM clients

**Components**:
- **FastMCP Framework**: Core MCP server implementation
- **Transport Handlers**: Support for stdio, HTTP, and SSE transports
- **Request Router**: Route MCP requests to appropriate tools
- **Response Serializer**: Convert Python objects to MCP responses

**Key Files**:
- `src/weather_mcp/server.py` - Main server factory and configuration
- `deployments/*/main.py` - Environment-specific entry points

### 2. Business Logic Layer

**Purpose**: Implement weather-specific business logic and data validation

**Components**:
- **Weather Tools**: MCP tool implementations for weather operations
- **Data Models**: Pydantic models for type safety and validation
- **Input Validation**: Ensure valid coordinates, date ranges, etc.
- **Error Handling**: Convert API errors to user-friendly messages

**Key Files**:
- `src/weather_mcp/weather/tools.py` - MCP tool implementations
- `src/weather_mcp/weather/models.py` - Pydantic data models
- `src/weather_mcp/config/settings.py` - Configuration management

### 3. External API Layer

**Purpose**: Interface with open-meteo.com weather API

**Components**:
- **API Client**: Async HTTP client for weather API calls
- **Request Builder**: Construct API requests with proper parameters
- **Response Parser**: Parse and validate API responses
- **Error Handler**: Handle API-specific errors and timeouts

**Key Files**:
- `src/weather_mcp/weather/client.py` - Open-Meteo API client

## Data Flow

### Request Flow
1. **MCP Client** sends tool request (e.g., get_current_weather)
2. **FastMCP Framework** receives and validates MCP request
3. **Weather Tool** extracts parameters and validates input
4. **API Client** makes HTTP request to open-meteo.com
5. **Response Parser** validates and transforms API response
6. **Data Model** serializes response to Pydantic model
7. **FastMCP Framework** returns MCP response to client

### Error Flow
1. **Error Occurs** at any layer (network, API, validation)
2. **Error Handler** catches and categorizes the error
3. **Error Transformer** converts to user-friendly message
4. **MCP Framework** returns error response to client

## Deployment Architectures

### Local Development (uv venv)
```
┌─────────────────┐    stdio    ┌─────────────────┐
│   MCP Client    │◄───────────►│  Python Process │
│   (Claude, etc) │             │  (FastMCP)      │
└─────────────────┘             └─────────────────┘
                                         │ HTTPS
                                ┌─────────▼─────────┐
                                │  open-meteo.com   │
                                └───────────────────┘
```

### Docker Deployment
```
┌─────────────────┐    HTTP     ┌─────────────────┐
│   MCP Client    │◄───────────►│ Docker Container│
│   (Remote)      │             │  (FastMCP)      │
└─────────────────┘             └─────────────────┘
                                         │ HTTPS
                                ┌─────────▼─────────┐
                                │  open-meteo.com   │
                                └───────────────────┘
```

### Cloudflare Workers
```
┌─────────────────┐    HTTPS    ┌─────────────────┐
│   MCP Client    │◄───────────►│ Cloudflare      │
│   (Remote)      │             │ Python Worker   │
└─────────────────┘             └─────────────────┘
                                         │ HTTPS
                                ┌─────────▼─────────┐
                                │  open-meteo.com   │
                                └───────────────────┘
```

## Component Details

### Weather Tools

Each weather tool follows a consistent pattern:

```python
@mcp.tool()
async def tool_name(parameters: ToolInput) -> ToolOutput:
    """Tool description for LLM"""
    # 1. Validate input parameters
    # 2. Call API client
    # 3. Transform response
    # 4. Return typed result
```

**Available Tools**:
- `get_current_weather(lat, lon, timezone)` → CurrentWeather
- `get_weather_forecast(lat, lon, days, timezone)` → List[WeatherForecast]
- `get_hourly_forecast(lat, lon, hours, timezone)` → List[HourlyWeather]
- `search_location(query)` → List[WeatherLocation]
- `get_weather_alerts(lat, lon)` → List[WeatherAlert]

### Data Models

All data models use Pydantic for validation and serialization:

```python
class WeatherLocation(BaseModel):
    latitude: float = Field(..., ge=-90, le=90)
    longitude: float = Field(..., ge=-180, le=180)
    name: str
    country: str
    timezone: str

class CurrentWeather(BaseModel):
    temperature: float
    humidity: int = Field(..., ge=0, le=100)
    wind_speed: float = Field(..., ge=0)
    wind_direction: int = Field(..., ge=0, le=360)
    weather_code: int
    conditions: str
    timestamp: datetime
```

### API Client

The Open-Meteo client handles all external API communication:

```python
class OpenMeteoClient:
    def __init__(self):
        self.base_url = "https://api.open-meteo.com/v1"
        self.geocoding_url = "https://geocoding-api.open-meteo.com/v1"
        self.timeout = 10.0
    
    async def get_current_weather(self, lat: float, lon: float, timezone: str) -> Dict:
        # Implementation with error handling
    
    async def get_forecast(self, lat: float, lon: float, days: int, timezone: str) -> Dict:
        # Implementation with error handling
```

## Configuration Management

### Environment-Specific Configuration

**Local Development**:
```python
# .env file
OPEN_METEO_BASE_URL=https://api.open-meteo.com/v1
GEOCODING_BASE_URL=https://geocoding-api.open-meteo.com/v1
REQUEST_TIMEOUT=10.0
```

**Docker**:
```yaml
# docker-compose.yml
environment:
  - OPEN_METEO_BASE_URL=https://api.open-meteo.com/v1
  - REQUEST_TIMEOUT=10.0
```

**Cloudflare Workers**:
```toml
# wrangler.toml
[vars]
OPEN_METEO_BASE_URL = "https://api.open-meteo.com/v1"
REQUEST_TIMEOUT = "10.0"
```

### Settings Management

```python
class Settings(BaseSettings):
    open_meteo_base_url: str = "https://api.open-meteo.com/v1"
    geocoding_base_url: str = "https://geocoding-api.open-meteo.com/v1"
    request_timeout: float = 10.0
    max_forecast_days: int = 16
    max_hourly_hours: int = 168
    
    class Config:
        env_file = ".env"
```

## Error Handling Strategy

### Error Categories

1. **Input Validation Errors**
   - Invalid coordinates (lat/lon out of range)
   - Invalid date ranges
   - Missing required parameters

2. **API Communication Errors**
   - Network timeouts
   - HTTP status errors (4xx, 5xx)
   - Connection failures

3. **Data Processing Errors**
   - Invalid API response format
   - Missing expected fields
   - Type conversion failures

### Error Handling Pattern

```python
async def weather_tool(params: ToolParams) -> ToolResult:
    try:
        # Validate input
        validated_params = validate_input(params)
        
        # Call API
        api_response = await api_client.call(validated_params)
        
        # Process response
        result = process_response(api_response)
        
        return result
        
    except ValidationError as e:
        raise MCPError(f"Invalid input: {e.message}")
    except httpx.TimeoutException:
        raise MCPError("Weather service is currently unavailable")
    except httpx.HTTPStatusError as e:
        raise MCPError(f"Weather service error: {e.response.status_code}")
    except Exception as e:
        raise MCPError(f"Unexpected error: {str(e)}")
```

## Performance Considerations

### Response Time Targets
- **Current Weather**: < 1 second
- **Forecast Requests**: < 2 seconds
- **Location Search**: < 1 second
- **Error Responses**: < 500ms

### Optimization Strategies
1. **Async Operations**: All I/O operations are async
2. **Connection Pooling**: httpx client reuses connections
3. **Timeout Management**: Reasonable timeouts prevent hanging
4. **Minimal Processing**: Direct API response transformation

### Scalability Considerations
- **Stateless Design**: No shared state between requests
- **Resource Limits**: Respect open-meteo.com rate limits
- **Error Recovery**: Graceful degradation on API failures

## Security Considerations

### API Security
- **HTTPS Only**: All external API calls use HTTPS
- **No API Keys**: Leverage open-meteo.com's free tier
- **Input Validation**: Strict validation of all inputs
- **Error Sanitization**: No sensitive data in error messages

### Deployment Security
- **Environment Variables**: Sensitive config via env vars
- **No Secrets**: No hardcoded credentials or tokens
- **Minimal Attack Surface**: Simple, focused functionality

## Monitoring and Observability

### Logging Strategy
- **Structured Logging**: JSON format for machine parsing
- **Log Levels**: DEBUG, INFO, WARNING, ERROR
- **Request Tracing**: Track requests through the system
- **Error Logging**: Detailed error information for debugging

### Metrics
- **Request Count**: Number of tool calls
- **Response Time**: Latency for each tool
- **Error Rate**: Percentage of failed requests
- **API Health**: Open-meteo.com availability

### Health Checks
- **Liveness**: Server is running and responsive
- **Readiness**: Can successfully call external APIs
- **Dependency Health**: Open-meteo.com API status

## Testing Strategy

### Unit Tests
- **Tool Logic**: Test each MCP tool independently
- **Data Models**: Validate Pydantic model behavior
- **API Client**: Mock external API calls
- **Error Handling**: Test error scenarios

### Integration Tests
- **End-to-End**: Full request/response cycle
- **API Integration**: Real calls to open-meteo.com
- **Transport Testing**: Test different MCP transports
- **Environment Testing**: Test all deployment configurations

### Performance Tests
- **Load Testing**: Multiple concurrent requests
- **Timeout Testing**: Verify timeout behavior
- **Error Recovery**: Test resilience to API failures

For implementation details, see [Implementation Plan](./03-implementation-plan.md).