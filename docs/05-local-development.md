# Local Development Guide

## Overview

This guide provides comprehensive instructions for setting up and running the Weather Forecast MCP Server locally using uv venv. This is the recommended approach for development, testing, and local MCP client integration.

## Prerequisites

### System Requirements
- **Operating System**: Windows, macOS, or Linux
- **Python**: 3.12 or higher
- **Package Manager**: uv (recommended) or pip
- **Git**: For version control

### Installing uv

**On Windows (PowerShell)**:
```powershell
# Install uv using pip
pip install uv

# Or using winget
winget install astral-sh.uv
```

**On macOS**:
```bash
# Using Homebrew
brew install uv

# Or using curl
curl -LsSf https://astral.sh/uv/install.sh | sh
```

**On Linux**:
```bash
# Using curl
curl -LsSf https://astral.sh/uv/install.sh | sh

# Or using pip
pip install uv
```

### Verify Installation
```bash
uv --version
python --version  # Should be 3.12+
```

## Project Setup

### 1. Clone the Repository
```bash
git clone <repository-url>
cd weather-mcp-server
```

### 2. Create Virtual Environment
```bash
# Create virtual environment with uv
uv venv

# Activate virtual environment
# On Windows (PowerShell):
.venv\Scripts\Activate.ps1

# On Windows (Command Prompt):
.venv\Scripts\activate.bat

# On macOS/Linux:
source .venv/bin/activate
```

### 3. Install Dependencies
```bash
# Install project in development mode with all dependencies
uv pip install -e ".[dev]"

# Verify installation
uv pip list
```

### 4. Environment Configuration

Create a `.env` file in the project root:
```bash
# .env
OPEN_METEO_BASE_URL=https://api.open-meteo.com/v1
GEOCODING_BASE_URL=https://geocoding-api.open-meteo.com/v1
REQUEST_TIMEOUT=10.0
LOG_LEVEL=INFO
```

### 5. Verify Setup
```bash
# Run basic tests to verify setup
python -m pytest tests/unit/ -v

# Check code quality
ruff check src/
mypy src/
```

## Development Workflow

### Project Structure
```
weather-mcp-server/
├── src/weather_mcp/           # Main package
│   ├── __init__.py
│   ├── server.py              # FastMCP server factory
│   ├── weather/               # Weather functionality
│   │   ├── __init__.py
│   │   ├── client.py          # API client
│   │   ├── models.py          # Data models
│   │   └── tools.py           # MCP tools
│   └── config/
│       ├── __init__.py
│       └── settings.py        # Configuration
├── deployments/local/         # Local deployment
│   ├── main.py               # Entry point
│   └── requirements.txt      # Dependencies
├── tests/                     # Test suite
├── .env                      # Environment variables
└── pyproject.toml            # Project configuration
```

### Running the Server

**Basic Server Start**:
```bash
# Navigate to local deployment directory
cd deployments/local

# Run the MCP server
python main.py
```

**With Debug Logging**:
```bash
# Set debug logging level
export LOG_LEVEL=DEBUG  # On Windows: set LOG_LEVEL=DEBUG
python main.py
```

**Expected Output**:
```
Starting Weather MCP Server (local development)
Transport: stdio
Server Name: weather-forecast-server
Version: 1.0.0
Tools Available: 5
Press Ctrl+C to stop
```

### Development Server Features

The local development server includes:

**1. Enhanced Logging**:
```python
# deployments/local/main.py
import logging
import sys
from weather_mcp.server import create_mcp_server

def setup_logging():
    log_level = os.getenv('LOG_LEVEL', 'INFO')
    logging.basicConfig(
        level=getattr(logging, log_level),
        format='%(asctime)s - %(name)s - %(levelname)s - %(message)s',
        handlers=[
            logging.StreamHandler(sys.stdout),
            logging.FileHandler('weather-mcp.log')
        ]
    )

async def main():
    setup_logging()
    logger = logging.getLogger(__name__)
    
    logger.info("Starting Weather MCP Server (local development)")
    
    server = create_mcp_server()
    logger.info(f"Server Name: {server.name}")
    logger.info(f"Tools Available: {len(server.tools)}")
    
    try:
        await server.run(transport="stdio")
    except KeyboardInterrupt:
        logger.info("Shutting down server...")
```

**2. Environment Validation**:
```python
def validate_environment():
    """Validate required environment variables and settings"""
    required_vars = [
        'OPEN_METEO_BASE_URL',
        'GEOCODING_BASE_URL'
    ]
    
    missing_vars = []
    for var in required_vars:
        if not os.getenv(var):
            missing_vars.append(var)
    
    if missing_vars:
        raise EnvironmentError(f"Missing required environment variables: {missing_vars}")
    
    # Test API connectivity
    import httpx
    try:
        response = httpx.get(os.getenv('OPEN_METEO_BASE_URL'), timeout=5)
        response.raise_for_status()
    except Exception as e:
        raise EnvironmentError(f"Cannot connect to Open-Meteo API: {e}")
```

**3. Development Utilities**:
```python
def print_server_info(server):
    """Print useful development information"""
    print(f"Server Name: {server.name}")
    print(f"Version: {server.version}")
    print(f"Tools Available: {len(server.tools)}")
    print("Available Tools:")
    for tool_name in server.tools:
        print(f"  - {tool_name}")
    print(f"Configuration loaded from: {os.path.abspath('.env')}")
```

## Testing the Server

### Manual Testing with MCP Client

**1. Test Current Weather**:
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

**2. Test Location Search**:
```json
{
  "jsonrpc": "2.0",
  "id": 2,
  "method": "tools/call",
  "params": {
    "name": "search_location",
    "arguments": {
      "query": "Tokyo, Japan"
    }
  }
}
```

### Automated Testing

**Run Unit Tests**:
```bash
# Run all unit tests
python -m pytest tests/unit/ -v

# Run specific test file
python -m pytest tests/unit/test_tools.py -v

# Run with coverage
python -m pytest tests/unit/ --cov=src/weather_mcp --cov-report=html
```

**Run Integration Tests**:
```bash
# Run integration tests (requires internet connection)
python -m pytest tests/integration/ -v

# Skip slow tests
python -m pytest tests/integration/ -v -m "not slow"
```

**Run All Tests**:
```bash
# Run complete test suite
python -m pytest tests/ -v

# Run with coverage and generate report
python -m pytest tests/ --cov=src/weather_mcp --cov-report=html --cov-report=term
```

### Test Configuration

**pytest.ini**:
```ini
[tool:pytest]
testpaths = tests
python_files = test_*.py
python_classes = Test*
python_functions = test_*
addopts = 
    -v
    --tb=short
    --strict-markers
markers =
    slow: marks tests as slow (deselect with '-m "not slow"')
    integration: marks tests as integration tests
    unit: marks tests as unit tests
asyncio_mode = auto
```

## Development Tools

### Code Quality

**Linting with Ruff**:
```bash
# Check code style
ruff check src/ tests/

# Fix auto-fixable issues
ruff check --fix src/ tests/

# Format code
ruff format src/ tests/
```

**Type Checking with mypy**:
```bash
# Run type checking
mypy src/

# Check specific file
mypy src/weather_mcp/weather/tools.py
```

**Pre-commit Setup**:
```bash
# Install pre-commit hooks
pre-commit install

# Run hooks manually
pre-commit run --all-files
```

### Debugging

**Debug Configuration for VS Code**:
```json
{
  "version": "0.2.0",
  "configurations": [
    {
      "name": "Debug MCP Server",
      "type": "python",
      "request": "launch",
      "program": "${workspaceFolder}/deployments/local/main.py",
      "console": "integratedTerminal",
      "cwd": "${workspaceFolder}",
      "env": {
        "LOG_LEVEL": "DEBUG"
      }
    },
    {
      "name": "Debug Tests",
      "type": "python",
      "request": "launch",
      "module": "pytest",
      "args": ["tests/unit/", "-v"],
      "console": "integratedTerminal",
      "cwd": "${workspaceFolder}"
    }
  ]
}
```

**Debug with pdb**:
```python
# Add breakpoint in code
import pdb; pdb.set_trace()

# Or use built-in breakpoint() (Python 3.7+)
breakpoint()
```

### Performance Profiling

**Profile Server Performance**:
```bash
# Install profiling tools
uv pip install py-spy

# Profile running server
py-spy record -o profile.svg -- python deployments/local/main.py
```

**Memory Profiling**:
```bash
# Install memory profiler
uv pip install memory-profiler

# Profile memory usage
python -m memory_profiler deployments/local/main.py
```

## Common Development Tasks

### Adding a New Weather Tool

**1. Define the Tool Function**:
```python
# src/weather_mcp/weather/tools.py
@mcp.tool()
async def get_weather_history(
    latitude: float,
    longitude: float,
    start_date: str,
    end_date: str
) -> List[HistoricalWeather]:
    """Get historical weather data for a date range"""
    # Implementation here
```

**2. Add Data Model**:
```python
# src/weather_mcp/weather/models.py
class HistoricalWeather(BaseModel):
    date: date
    temperature_avg: float
    precipitation: float
    # Additional fields
```

**3. Update API Client**:
```python
# src/weather_mcp/weather/client.py
async def get_historical_weather(self, lat: float, lon: float, start: str, end: str):
    """Get historical weather data"""
    # API call implementation
```

**4. Add Tests**:
```python
# tests/unit/test_tools.py
@pytest.mark.asyncio
async def test_get_weather_history():
    # Test implementation
```

**5. Register Tool**:
```python
# src/weather_mcp/weather/tools.py
def register_weather_tools(mcp: FastMCP):
    # Register existing tools
    # Register new tool
    register_weather_history_tool(mcp)
```

### Updating Dependencies

**Update All Dependencies**:
```bash
# Update to latest compatible versions
uv pip install --upgrade -e ".[dev]"

# Check for outdated packages
uv pip list --outdated
```

**Update Specific Package**:
```bash
# Update specific package
uv pip install --upgrade fastmcp

# Update with version constraint
uv pip install "fastmcp>=2.10.0"
```

### Environment Management

**Multiple Python Versions**:
```bash
# Create environment with specific Python version
uv venv --python 3.12
uv venv --python 3.13

# Switch between environments
source .venv-312/bin/activate
source .venv-313/bin/activate
```

**Clean Environment**:
```bash
# Remove virtual environment
rm -rf .venv

# Recreate clean environment
uv venv
uv pip install -e ".[dev]"
```

## Troubleshooting

### Common Issues

**1. Import Errors**:
```bash
# Ensure package is installed in development mode
uv pip install -e .

# Check Python path
python -c "import sys; print(sys.path)"
```

**2. Environment Variable Issues**:
```bash
# Check environment variables
python -c "import os; print(os.environ.get('OPEN_METEO_BASE_URL'))"

# Verify .env file loading
python -c "from weather_mcp.config.settings import get_settings; print(get_settings())"
```

**3. API Connection Issues**:
```bash
# Test API connectivity
curl "https://api.open-meteo.com/v1/forecast?latitude=52.52&longitude=13.41&current=temperature_2m"

# Test with Python
python -c "import httpx; print(httpx.get('https://api.open-meteo.com/v1/forecast?latitude=52.52&longitude=13.41&current=temperature_2m').json())"
```

**4. Permission Issues**:
```bash
# On Windows, run PowerShell as Administrator
Set-ExecutionPolicy -ExecutionPolicy RemoteSigned -Scope CurrentUser

# On Unix systems, check file permissions
chmod +x deployments/local/main.py
```

### Debug Logging

**Enable Detailed Logging**:
```bash
# Set environment variable
export LOG_LEVEL=DEBUG

# Or modify .env file
echo "LOG_LEVEL=DEBUG" >> .env
```

**Log Configuration**:
```python
# Custom logging configuration
import logging

logging.basicConfig(
    level=logging.DEBUG,
    format='%(asctime)s - %(name)s - %(levelname)s - %(message)s',
    handlers=[
        logging.StreamHandler(),
        logging.FileHandler('debug.log')
    ]
)

# Enable httpx logging
logging.getLogger("httpx").setLevel(logging.DEBUG)
```

### Performance Issues

**Check Response Times**:
```python
import time
import asyncio
from weather_mcp.weather.client import OpenMeteoClient

async def benchmark_api():
    client = OpenMeteoClient()
    
    start_time = time.time()
    result = await client.get_current_weather(52.52, 13.41)
    end_time = time.time()
    
    print(f"API call took: {end_time - start_time:.2f} seconds")

asyncio.run(benchmark_api())
```

**Monitor Memory Usage**:
```bash
# Install psutil
uv pip install psutil

# Monitor memory
python -c "
import psutil
import os
process = psutil.Process(os.getpid())
print(f'Memory usage: {process.memory_info().rss / 1024 / 1024:.2f} MB')
"
```

## Integration with MCP Clients

### Claude Desktop Integration

**1. Configure Claude Desktop**:
```json
{
  "mcpServers": {
    "weather": {
      "command": "python",
      "args": ["/path/to/weather-mcp-server/deployments/local/main.py"],
      "cwd": "/path/to/weather-mcp-server"
    }
  }
}
```

**2. Test Integration**:
- Start Claude Desktop
- Ask: "What's the weather like in Paris?"
- Verify the weather tool is called

### Other MCP Clients

**Generic MCP Client Configuration**:
```bash
# Start server in stdio mode
python deployments/local/main.py

# Connect with any MCP-compatible client
# The server will communicate via stdin/stdout
```

## Next Steps

After setting up local development:

1. **Explore the Code**: Familiarize yourself with the project structure
2. **Run Tests**: Ensure everything works correctly
3. **Make Changes**: Start implementing new features or fixes
4. **Test Changes**: Use the test suite to validate your work
5. **Deploy**: Move to Docker or Cloudflare Workers deployment

For Docker deployment, see [Docker Deployment Guide](./06-docker-deployment.md).
For Cloudflare Workers deployment, see [Cloudflare Deployment Guide](./07-cloudflare-deployment.md).