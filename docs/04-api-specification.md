# API Specification

## Overview

This document provides the complete API specification for the Weather Forecast MCP Server, including all MCP tools, data models, error handling, and usage examples.

## MCP Tools

### 1. get_current_weather

Get current weather conditions for a specific location.

**Tool Definition**:
```python
@mcp.tool()
async def get_current_weather(
    latitude: float, 
    longitude: float, 
    timezone: str = "auto"
) -> CurrentWeather:
    """Get current weather conditions for a location
    
    Args:
        latitude: Latitude coordinate (-90 to 90)
        longitude: Longitude coordinate (-180 to 180)
        timezone: Timezone for the response (default: "auto")
    
    Returns:
        CurrentWeather: Current weather conditions
    
    Raises:
        MCPError: If coordinates are invalid or API call fails
    """
```

**Input Parameters**:
- `latitude` (float, required): Latitude coordinate between -90 and 90
- `longitude` (float, required): Longitude coordinate between -180 and 180
- `timezone` (str, optional): Timezone identifier (default: "auto")

**Output Model**:
```python
class CurrentWeather(BaseModel):
    temperature: float  # Temperature in Celsius
    humidity: int  # Relative humidity percentage (0-100)
    wind_speed: float  # Wind speed in km/h
    wind_direction: int  # Wind direction in degrees (0-360)
    weather_code: int  # WMO weather code
    conditions: str  # Human-readable weather description
    timestamp: datetime  # Time of observation
```

**Example Usage**:
```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "tools/call",
  "params": {
    "name": "get_current_weather",
    "arguments": {
      "latitude": 52.52,
      "longitude": 13.41,
      "timezone": "Europe/Berlin"
    }
  }
}
```

**Example Response**:
```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": {
    "content": [
      {
        "type": "text",
        "text": {
          "temperature": 18.5,
          "humidity": 65,
          "wind_speed": 12.3,
          "wind_direction": 180,
          "weather_code": 1,
          "conditions": "Mainly clear",
          "timestamp": "2024-01-15T14:30:00+01:00"
        }
      }
    ]
  }
}
```

### 2. get_weather_forecast

Get daily weather forecast for a specified number of days.

**Tool Definition**:
```python
@mcp.tool()
async def get_weather_forecast(
    latitude: float,
    longitude: float,
    days: int = 7,
    timezone: str = "auto"
) -> List[WeatherForecast]:
    """Get weather forecast for specified number of days
    
    Args:
        latitude: Latitude coordinate (-90 to 90)
        longitude: Longitude coordinate (-180 to 180)
        days: Number of forecast days (1-16, default: 7)
        timezone: Timezone for the response (default: "auto")
    
    Returns:
        List[WeatherForecast]: Daily weather forecasts
    
    Raises:
        MCPError: If parameters are invalid or API call fails
    """
```

**Input Parameters**:
- `latitude` (float, required): Latitude coordinate between -90 and 90
- `longitude` (float, required): Longitude coordinate between -180 and 180
- `days` (int, optional): Number of forecast days, 1-16 (default: 7)
- `timezone` (str, optional): Timezone identifier (default: "auto")

**Output Model**:
```python
class WeatherForecast(BaseModel):
    date: date  # Forecast date
    temperature_min: float  # Minimum temperature in Celsius
    temperature_max: float  # Maximum temperature in Celsius
    precipitation: float  # Precipitation sum in mm
    weather_code: int  # WMO weather code
    conditions: str  # Human-readable weather description
```

**Example Usage**:
```json
{
  "jsonrpc": "2.0",
  "id": 2,
  "method": "tools/call",
  "params": {
    "name": "get_weather_forecast",
    "arguments": {
      "latitude": 40.7128,
      "longitude": -74.0060,
      "days": 5
    }
  }
}
```

### 3. get_hourly_forecast

Get hourly weather forecast for detailed planning.

**Tool Definition**:
```python
@mcp.tool()
async def get_hourly_forecast(
    latitude: float,
    longitude: float,
    hours: int = 24,
    timezone: str = "auto"
) -> List[HourlyWeather]:
    """Get hourly weather forecast for specified hours
    
    Args:
        latitude: Latitude coordinate (-90 to 90)
        longitude: Longitude coordinate (-180 to 180)
        hours: Number of forecast hours (1-168, default: 24)
        timezone: Timezone for the response (default: "auto")
    
    Returns:
        List[HourlyWeather]: Hourly weather forecasts
    
    Raises:
        MCPError: If parameters are invalid or API call fails
    """
```

**Input Parameters**:
- `latitude` (float, required): Latitude coordinate between -90 and 90
- `longitude` (float, required): Longitude coordinate between -180 and 180
- `hours` (int, optional): Number of forecast hours, 1-168 (default: 24)
- `timezone` (str, optional): Timezone identifier (default: "auto")

**Output Model**:
```python
class HourlyWeather(BaseModel):
    datetime: datetime  # Forecast date and time
    temperature: float  # Temperature in Celsius
    humidity: int  # Relative humidity percentage (0-100)
    precipitation: float  # Precipitation in mm
    wind_speed: float  # Wind speed in km/h
    wind_direction: int  # Wind direction in degrees (0-360)
    weather_code: int  # WMO weather code
    conditions: str  # Human-readable weather description
```

### 4. search_location

Search for locations by name or address to get coordinates.

**Tool Definition**:
```python
@mcp.tool()
async def search_location(query: str) -> List[WeatherLocation]:
    """Search for locations by name or address
    
    Args:
        query: Location name, address, or search term
    
    Returns:
        List[WeatherLocation]: Matching locations with coordinates
    
    Raises:
        MCPError: If search fails or no results found
    """
```

**Input Parameters**:
- `query` (str, required): Location name, address, or search term

**Output Model**:
```python
class WeatherLocation(BaseModel):
    latitude: float  # Latitude coordinate
    longitude: float  # Longitude coordinate
    name: str  # Location name
    country: str  # Country name
    admin1: Optional[str]  # State/province (if available)
    admin2: Optional[str]  # County/region (if available)
    timezone: str  # Timezone identifier
    elevation: Optional[float]  # Elevation in meters (if available)
```

**Example Usage**:
```json
{
  "jsonrpc": "2.0",
  "id": 3,
  "method": "tools/call",
  "params": {
    "name": "search_location",
    "arguments": {
      "query": "Tokyo, Japan"
    }
  }
}
```

### 5. get_weather_alerts

Get active weather alerts and warnings for a location.

**Tool Definition**:
```python
@mcp.tool()
async def get_weather_alerts(
    latitude: float,
    longitude: float
) -> List[WeatherAlert]:
    """Get active weather alerts for a location
    
    Args:
        latitude: Latitude coordinate (-90 to 90)
        longitude: Longitude coordinate (-180 to 180)
    
    Returns:
        List[WeatherAlert]: Active weather alerts and warnings
    
    Raises:
        MCPError: If coordinates are invalid or API call fails
    """
```

**Input Parameters**:
- `latitude` (float, required): Latitude coordinate between -90 and 90
- `longitude` (float, required): Longitude coordinate between -180 and 180

**Output Model**:
```python
class WeatherAlert(BaseModel):
    event: str  # Alert event type
    headline: str  # Alert headline
    description: str  # Detailed alert description
    severity: str  # Alert severity level
    urgency: str  # Alert urgency level
    areas: str  # Affected areas
    start_time: datetime  # Alert start time
    end_time: datetime  # Alert end time
    sender_name: str  # Alert issuing authority
```

## Data Models

### Base Models

All data models inherit from Pydantic's BaseModel for validation and serialization:

```python
from pydantic import BaseModel, Field, validator
from datetime import datetime, date
from typing import List, Optional
```

### Validation Rules

**Coordinate Validation**:
```python
class CoordinateValidation:
    @validator('latitude')
    def validate_latitude(cls, v):
        if not -90 <= v <= 90:
            raise ValueError('Latitude must be between -90 and 90')
        return v
    
    @validator('longitude')
    def validate_longitude(cls, v):
        if not -180 <= v <= 180:
            raise ValueError('Longitude must be between -180 and 180')
        return v
```

**Range Validation**:
```python
class RangeValidation:
    @validator('days')
    def validate_days(cls, v):
        if not 1 <= v <= 16:
            raise ValueError('Days must be between 1 and 16')
        return v
    
    @validator('hours')
    def validate_hours(cls, v):
        if not 1 <= v <= 168:
            raise ValueError('Hours must be between 1 and 168')
        return v
```

### Weather Code Mapping

WMO Weather Codes to human-readable descriptions:

```python
WEATHER_CODE_DESCRIPTIONS = {
    0: "Clear sky",
    1: "Mainly clear",
    2: "Partly cloudy",
    3: "Overcast",
    45: "Fog",
    48: "Depositing rime fog",
    51: "Light drizzle",
    53: "Moderate drizzle",
    55: "Dense drizzle",
    56: "Light freezing drizzle",
    57: "Dense freezing drizzle",
    61: "Slight rain",
    63: "Moderate rain",
    65: "Heavy rain",
    66: "Light freezing rain",
    67: "Heavy freezing rain",
    71: "Slight snow fall",
    73: "Moderate snow fall",
    75: "Heavy snow fall",
    77: "Snow grains",
    80: "Slight rain showers",
    81: "Moderate rain showers",
    82: "Violent rain showers",
    85: "Slight snow showers",
    86: "Heavy snow showers",
    95: "Thunderstorm",
    96: "Thunderstorm with slight hail",
    99: "Thunderstorm with heavy hail"
}

def weather_code_to_description(code: int) -> str:
    """Convert WMO weather code to human-readable description"""
    return WEATHER_CODE_DESCRIPTIONS.get(code, f"Unknown weather code: {code}")
```

## Error Handling

### Error Types

**MCPError**: Base error class for all MCP-related errors
```python
class MCPError(Exception):
    """Base class for MCP server errors"""
    def __init__(self, message: str, code: Optional[str] = None):
        self.message = message
        self.code = code
        super().__init__(message)
```

**WeatherAPIError**: Weather API specific errors
```python
class WeatherAPIError(MCPError):
    """Weather API related errors"""
    pass
```

**ValidationError**: Input validation errors
```python
class ValidationError(MCPError):
    """Input validation errors"""
    pass
```

### Error Response Format

All errors are returned in standard MCP error format:

```json
{
  "jsonrpc": "2.0",
  "id": 1,
  "error": {
    "code": -32602,
    "message": "Invalid coordinates: Latitude must be between -90 and 90",
    "data": {
      "type": "ValidationError",
      "details": "Provided latitude 95.0 is outside valid range"
    }
  }
}
```

### Common Error Scenarios

**Invalid Coordinates**:
```json
{
  "error": {
    "code": -32602,
    "message": "Invalid coordinates: Latitude must be between -90 and 90"
  }
}
```

**API Timeout**:
```json
{
  "error": {
    "code": -32603,
    "message": "Weather service is currently unavailable (timeout)"
  }
}
```

**Invalid Date Range**:
```json
{
  "error": {
    "code": -32602,
    "message": "Invalid forecast period: Days must be between 1 and 16"
  }
}
```

**Location Not Found**:
```json
{
  "error": {
    "code": -32603,
    "message": "No locations found for query: 'InvalidPlace123'"
  }
}
```

## Usage Examples

### Complete MCP Client Interaction

**1. Search for a location**:
```json
Request:
{
  "jsonrpc": "2.0",
  "id": 1,
  "method": "tools/call",
  "params": {
    "name": "search_location",
    "arguments": {
      "query": "Paris, France"
    }
  }
}

Response:
{
  "jsonrpc": "2.0",
  "id": 1,
  "result": {
    "content": [
      {
        "type": "text",
        "text": [
          {
            "latitude": 48.8566,
            "longitude": 2.3522,
            "name": "Paris",
            "country": "France",
            "admin1": "Île-de-France",
            "timezone": "Europe/Paris",
            "elevation": 42.0
          }
        ]
      }
    ]
  }
}
```

**2. Get current weather for the location**:
```json
Request:
{
  "jsonrpc": "2.0",
  "id": 2,
  "method": "tools/call",
  "params": {
    "name": "get_current_weather",
    "arguments": {
      "latitude": 48.8566,
      "longitude": 2.3522,
      "timezone": "Europe/Paris"
    }
  }
}

Response:
{
  "jsonrpc": "2.0",
  "id": 2,
  "result": {
    "content": [
      {
        "type": "text",
        "text": {
          "temperature": 15.2,
          "humidity": 78,
          "wind_speed": 8.5,
          "wind_direction": 225,
          "weather_code": 2,
          "conditions": "Partly cloudy",
          "timestamp": "2024-01-15T16:00:00+01:00"
        }
      }
    ]
  }
}
```

**3. Get 7-day forecast**:
```json
Request:
{
  "jsonrpc": "2.0",
  "id": 3,
  "method": "tools/call",
  "params": {
    "name": "get_weather_forecast",
    "arguments": {
      "latitude": 48.8566,
      "longitude": 2.3522,
      "days": 7,
      "timezone": "Europe/Paris"
    }
  }
}
```

### LLM Integration Examples

**Natural Language Query**: "What's the weather like in Tokyo right now?"

**MCP Tool Chain**:
1. `search_location("Tokyo")` → Get coordinates
2. `get_current_weather(35.6762, 139.6503)` → Get current conditions

**Natural Language Query**: "Will it rain in London this weekend?"

**MCP Tool Chain**:
1. `search_location("London")` → Get coordinates
2. `get_weather_forecast(51.5074, -0.1278, 7)` → Get 7-day forecast
3. Filter for weekend days and check precipitation

## Rate Limiting and Performance

### Open-Meteo API Limits
- **Free Tier**: 10,000 requests per day
- **Rate Limit**: No specific rate limit for reasonable usage
- **Response Time**: Typically 100-300ms

### Performance Targets
- **Current Weather**: < 1 second response time
- **Forecast Requests**: < 2 seconds response time
- **Location Search**: < 1 second response time
- **Error Responses**: < 500ms response time

### Optimization Strategies
- **Connection Pooling**: Reuse HTTP connections
- **Timeout Management**: 10-second timeout for API calls
- **Async Operations**: All I/O operations are asynchronous
- **Minimal Processing**: Direct API response transformation

## Security Considerations

### Input Validation
- **Coordinate Bounds**: Strict validation of latitude/longitude ranges
- **String Sanitization**: Clean location search queries
- **Parameter Limits**: Enforce maximum values for days/hours
- **Type Checking**: Pydantic model validation

### API Security
- **HTTPS Only**: All external API calls use HTTPS
- **No Credentials**: Leverage open-meteo.com's free tier
- **Error Sanitization**: No sensitive data in error messages
- **Request Validation**: Validate all incoming MCP requests

### Deployment Security
- **Environment Variables**: Configuration via environment variables
- **No Hardcoded Secrets**: No credentials in source code
- **Minimal Dependencies**: Keep dependency tree small
- **Regular Updates**: Keep dependencies updated

## Testing

### Unit Test Examples

**Tool Testing**:
```python
@pytest.mark.asyncio
async def test_get_current_weather_success():
    mock_response = {
        "current": {
            "temperature_2m": 20.5,
            "relative_humidity_2m": 65,
            "wind_speed_10m": 10.2,
            "wind_direction_10m": 180,
            "weather_code": 1,
            "time": "2024-01-01T12:00:00"
        }
    }
    
    with patch('weather_mcp.weather.client.OpenMeteoClient.get_current_weather') as mock_client:
        mock_client.return_value = mock_response
        
        result = await get_current_weather(52.52, 13.41)
        
        assert isinstance(result, CurrentWeather)
        assert result.temperature == 20.5
        assert result.conditions == "Mainly clear"
```

**Validation Testing**:
```python
def test_coordinate_validation():
    with pytest.raises(ValidationError):
        WeatherLocation(
            latitude=95.0,  # Invalid latitude
            longitude=0.0,
            name="Test",
            country="Test",
            timezone="UTC"
        )
```

### Integration Test Examples

**API Integration**:
```python
@pytest.mark.asyncio
async def test_real_api_integration():
    client = OpenMeteoClient()
    
    # Test with known coordinates (Berlin)
    result = await client.get_current_weather(52.52, 13.41)
    
    assert "current" in result
    assert "temperature_2m" in result["current"]
    assert isinstance(result["current"]["temperature_2m"], (int, float))
```

This API specification provides complete documentation for implementing and using the Weather Forecast MCP Server. All tools, models, and error handling are fully specified with examples and validation rules.