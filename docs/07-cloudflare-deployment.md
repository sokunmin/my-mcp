# Cloudflare Workers Deployment Guide

## Overview

This guide provides comprehensive instructions for deploying the Weather Forecast MCP Server on Cloudflare Workers using native Python support. Cloudflare Workers offers global edge deployment with excellent performance and scalability.

## Prerequisites

### System Requirements
- **Node.js**: 18.0 or higher (for Wrangler CLI)
- **Python**: 3.12 or higher (for local development)
- **Cloudflare Account**: Free or paid account
- **Git**: For version control

### Cloudflare Account Setup

**1. Create Cloudflare Account**:
- Visit [cloudflare.com](https://www.cloudflare.com)
- Sign up for a free account
- Verify your email address

**2. Get Account ID**:
- Log in to Cloudflare Dashboard
- Go to the right sidebar and copy your Account ID
- Save this for later configuration

**3. Create API Token**:
- Go to "My Profile" → "API Tokens"
- Click "Create Token"
- Use "Custom token" template
- Set permissions:
  - Account: Cloudflare Workers:Edit
  - Zone: Zone:Read (if using custom domain)
- Add Account ID to "Account Resources"
- Create and save the token securely

### Install Wrangler CLI

**Using npm**:
```bash
npm install -g wrangler

# Verify installation
wrangler --version
```

**Using yarn**:
```bash
yarn global add wrangler
```

**Authenticate Wrangler**:
```bash
# Login with browser
wrangler login

# Or set API token
export CLOUDFLARE_API_TOKEN="your-api-token"
```

## Project Configuration

### Wrangler Configuration

Create the main configuration file:

```toml
# deployments/cloudflare/wrangler.toml
name = "weather-mcp-server"
main = "src/main.py"
compatibility_date = "2024-12-01"
compatibility_flags = ["python_workers"]

# Account configuration
account_id = "your-account-id"

# Environment variables
[vars]
OPEN_METEO_BASE_URL = "https://api.open-meteo.com/v1"
GEOCODING_BASE_URL = "https://geocoding-api.open-meteo.com/v1"
REQUEST_TIMEOUT = "10.0"
MAX_FORECAST_DAYS = "16"
MAX_HOURLY_HOURS = "168"

# Observability
[observability]
enabled = true

# Optional: Custom domain
# routes = [
#   { pattern = "weather.yourdomain.com/*", custom_domain = true }
# ]

# Optional: Environment-specific configurations
[env.staging]
name = "weather-mcp-server-staging"
vars = { LOG_LEVEL = "DEBUG" }

[env.production]
name = "weather-mcp-server-prod"
vars = { LOG_LEVEL = "INFO" }
```

### Python Dependencies

```txt
# deployments/cloudflare/requirements.txt
fastmcp>=2.9.0
pydantic>=2.0.0
pydantic-settings>=2.0.0
httpx>=0.25.0
```

### Worker Entry Point

```python
# deployments/cloudflare/src/main.py
import json
import logging
from datetime import datetime
from typing import Dict, Any

# Import the MCP server factory
from weather_mcp.server import create_mcp_server
from weather_mcp.config.settings import get_settings

# Configure logging for Workers
logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

# Create server instance at module level for reuse
mcp_server = None

def get_server():
    """Get or create MCP server instance"""
    global mcp_server
    if mcp_server is None:
        mcp_server = create_mcp_server()
        logger.info(f"Created MCP server with {len(mcp_server.tools)} tools")
    return mcp_server

async def handle_mcp_request(request_data: Dict[str, Any]) -> Dict[str, Any]:
    """Handle MCP protocol requests"""
    server = get_server()
    
    try:
        # Process MCP request
        response = await server.handle_mcp_request(request_data)
        return response
    except Exception as e:
        logger.error(f"MCP request failed: {e}")
        return {
            "jsonrpc": "2.0",
            "id": request_data.get("id"),
            "error": {
                "code": -32603,
                "message": f"Internal error: {str(e)}"
            }
        }

async def handle_health_check() -> Dict[str, Any]:
    """Health check endpoint"""
    server = get_server()
    settings = get_settings()
    
    return {
        "status": "healthy",
        "service": "weather-mcp-server",
        "version": "1.0.0",
        "timestamp": datetime.utcnow().isoformat(),
        "tools_count": len(server.tools),
        "api_base_url": settings.open_meteo_base_url
    }

async def on_fetch(request, env, ctx):
    """Cloudflare Workers fetch handler"""
    try:
        url = request.url
        method = request.method
        
        logger.info(f"{method} {url}")
        
        # Parse URL path
        from urllib.parse import urlparse
        parsed_url = urlparse(url)
        path = parsed_url.path
        
        # Health check endpoint
        if path == "/health" and method == "GET":
            health_data = await handle_health_check()
            return Response(
                json.dumps(health_data),
                headers={"Content-Type": "application/json"}
            )
        
        # MCP endpoint
        if path in ["/mcp", "/"] and method == "POST":
            try:
                request_data = await request.json()
                response_data = await handle_mcp_request(request_data)
                
                return Response(
                    json.dumps(response_data),
                    headers={
                        "Content-Type": "application/json",
                        "Access-Control-Allow-Origin": "*",
                        "Access-Control-Allow-Methods": "POST, OPTIONS",
                        "Access-Control-Allow-Headers": "Content-Type"
                    }
                )
            except json.JSONDecodeError:
                return Response(
                    json.dumps({
                        "jsonrpc": "2.0",
                        "error": {
                            "code": -32700,
                            "message": "Parse error: Invalid JSON"
                        }
                    }),
                    status=400,
                    headers={"Content-Type": "application/json"}
                )
        
        # CORS preflight
        if method == "OPTIONS":
            return Response(
                None,
                headers={
                    "Access-Control-Allow-Origin": "*",
                    "Access-Control-Allow-Methods": "POST, OPTIONS",
                    "Access-Control-Allow-Headers": "Content-Type"
                }
            )
        
        # Default response
        return Response(
            json.dumps({
                "service": "weather-mcp-server",
                "version": "1.0.0",
                "endpoints": {
                    "health": "/health",
                    "mcp": "/mcp"
                }
            }),
            headers={"Content-Type": "application/json"}
        )
        
    except Exception as e:
        logger.error(f"Request failed: {e}")
        return Response(
            json.dumps({
                "error": "Internal server error",
                "message": str(e)
            }),
            status=500,
            headers={"Content-Type": "application/json"}
        )

# Export for Cloudflare Workers
export = {
    "fetch": on_fetch
}
```

## Deployment Methods

### Method 1: Direct GitHub Deployment

**1. Prepare Repository**:
```bash
# Ensure all files are committed
git add .
git commit -m "Prepare for Cloudflare deployment"
git push origin main
```

**2. Deploy from Local**:
```bash
cd deployments/cloudflare

# Deploy to staging
wrangler deploy --env staging

# Deploy to production
wrangler deploy --env production

# Deploy with custom name
wrangler deploy --name weather-mcp-custom
```

**3. Verify Deployment**:
```bash
# Test health endpoint
curl https://weather-mcp-server.your-subdomain.workers.dev/health

# Test MCP endpoint
curl -X POST https://weather-mcp-server.your-subdomain.workers.dev/mcp \
  -H "Content-Type: application/json" \
  -d '{
    "jsonrpc": "2.0",
    "id": 1,
    "method": "tools/list"
  }'
```

### Method 2: GitHub Actions Deployment

**1. Set up GitHub Secrets**:
- Go to GitHub repository → Settings → Secrets and variables → Actions
- Add secrets:
  - `CLOUDFLARE_API_TOKEN`: Your Cloudflare API token
  - `CLOUDFLARE_ACCOUNT_ID`: Your Cloudflare account ID

**2. Create GitHub Actions Workflow**:
```yaml
# .github/workflows/cloudflare-deploy.yml
name: Deploy to Cloudflare Workers

on:
  push:
    tags: ['v*']
    branches: ['main']
  workflow_dispatch:
    inputs:
      environment:
        description: 'Environment to deploy to'
        required: true
        default: 'staging'
        type: choice
        options:
          - staging
          - production

jobs:
  deploy:
    runs-on: ubuntu-latest
    
    steps:
      - name: Checkout code
        uses: actions/checkout@v4
      
      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '18'
          cache: 'npm'
      
      - name: Install Wrangler
        run: npm install -g wrangler
      
      - name: Deploy to Cloudflare Workers
        working-directory: deployments/cloudflare
        run: |
          if [[ "${{ github.ref }}" == refs/tags/* ]]; then
            wrangler deploy --env production
          elif [[ "${{ github.event.inputs.environment }}" == "production" ]]; then
            wrangler deploy --env production
          else
            wrangler deploy --env staging
          fi
        env:
          CLOUDFLARE_API_TOKEN: ${{ secrets.CLOUDFLARE_API_TOKEN }}
          CLOUDFLARE_ACCOUNT_ID: ${{ secrets.CLOUDFLARE_ACCOUNT_ID }}
      
      - name: Test deployment
        run: |
          sleep 10  # Wait for deployment to propagate
          if [[ "${{ github.ref }}" == refs/tags/* ]] || [[ "${{ github.event.inputs.environment }}" == "production" ]]; then
            WORKER_URL="https://weather-mcp-server-prod.your-subdomain.workers.dev"
          else
            WORKER_URL="https://weather-mcp-server-staging.your-subdomain.workers.dev"
          fi
          
          # Test health endpoint
          curl -f "$WORKER_URL/health"
          
          # Test MCP endpoint
          curl -f -X POST "$WORKER_URL/mcp" \
            -H "Content-Type: application/json" \
            -d '{"jsonrpc":"2.0","id":1,"method":"tools/list"}'
```

### Method 3: Docker Registry Deployment

**1. Build Multi-platform Image**:
```bash
# Build for multiple platforms
docker buildx build --platform linux/amd64,linux/arm64 \
  -f deployments/docker/Dockerfile \
  -t ghcr.io/your-org/weather-mcp-server:latest \
  --push .
```

**2. Deploy from Registry**:
```yaml
# deployments/cloudflare/wrangler-docker.toml
name = "weather-mcp-server-docker"
main = "src/main.py"
compatibility_date = "2024-12-01"
compatibility_flags = ["python_workers"]

# Use Docker image
[build]
command = "docker run --rm -v $(pwd):/workspace ghcr.io/your-org/weather-mcp-server:latest cp -r /app/src /workspace/"
```

## Environment Management

### Environment Variables

**Set via Wrangler**:
```bash
# Set individual variables
wrangler secret put OPEN_METEO_API_KEY --env production

# Set multiple variables
wrangler secret put GEOCODING_API_KEY --env production
```

**Set via wrangler.toml**:
```toml
[env.production.vars]
OPEN_METEO_BASE_URL = "https://api.open-meteo.com/v1"
LOG_LEVEL = "INFO"
REQUEST_TIMEOUT = "10.0"

[env.staging.vars]
OPEN_METEO_BASE_URL = "https://api.open-meteo.com/v1"
LOG_LEVEL = "DEBUG"
REQUEST_TIMEOUT = "15.0"
```

### Secrets Management

**For sensitive data**:
```bash
# Add secrets (not visible in dashboard)
wrangler secret put DATABASE_PASSWORD --env production
wrangler secret put API_KEY --env production

# List secrets
wrangler secret list --env production

# Delete secret
wrangler secret delete API_KEY --env production
```

## Custom Domains

### Setup Custom Domain

**1. Add Domain to Cloudflare**:
- Add your domain to Cloudflare
- Update nameservers
- Ensure SSL/TLS is configured

**2. Configure in wrangler.toml**:
```toml
# wrangler.toml
routes = [
  { pattern = "weather-api.yourdomain.com/*", custom_domain = true }
]

# Or use zone_id for more control
routes = [
  { pattern = "weather-api.yourdomain.com/*", zone_id = "your-zone-id" }
]
```

**3. Deploy with Custom Domain**:
```bash
wrangler deploy --env production
```

### SSL/TLS Configuration

**Automatic SSL**:
- Cloudflare provides automatic SSL certificates
- No additional configuration needed
- Certificates auto-renew

**Custom Certificates**:
```bash
# Upload custom certificate (if needed)
wrangler ssl upload --cert cert.pem --key key.pem
```

## Monitoring and Observability

### Built-in Analytics

**Enable in wrangler.toml**:
```toml
[observability]
enabled = true
```

**View Analytics**:
- Cloudflare Dashboard → Workers & Pages → Your Worker → Analytics
- View requests, errors, duration, and more

### Custom Metrics

**Add Custom Metrics**:
```python
# In your worker code
async def on_fetch(request, env, ctx):
    start_time = time.time()
    
    try:
        # Handle request
        response = await handle_request(request)
        
        # Log success metric
        ctx.waitUntil(log_metric("request_success", {
            "duration": time.time() - start_time,
            "status": response.status
        }))
        
        return response
    except Exception as e:
        # Log error metric
        ctx.waitUntil(log_metric("request_error", {
            "duration": time.time() - start_time,
            "error": str(e)
        }))
        raise

async def log_metric(name: str, data: dict):
    """Log custom metrics"""
    # Implement custom logging logic
    pass
```

### Logging

**Structured Logging**:
```python
import json
import logging

class CloudflareHandler(logging.Handler):
    def emit(self, record):
        log_entry = {
            "timestamp": record.created,
            "level": record.levelname,
            "message": record.getMessage(),
            "module": record.module,
            "function": record.funcName
        }
        print(json.dumps(log_entry))

# Configure logging
logger = logging.getLogger()
logger.addHandler(CloudflareHandler())
logger.setLevel(logging.INFO)
```

**View Logs**:
```bash
# Real-time logs
wrangler tail --env production

# Filter logs
wrangler tail --env production --format pretty

# Save logs to file
wrangler tail --env production > worker-logs.txt
```

## Performance Optimization

### Cold Start Optimization

**Module-level Initialization**:
```python
# Initialize expensive objects at module level
mcp_server = create_mcp_server()
http_client = httpx.AsyncClient()

async def on_fetch(request, env, ctx):
    # Reuse initialized objects
    response = await mcp_server.handle_request(request)
    return response
```

**Connection Pooling**:
```python
# Reuse HTTP connections
class GlobalHTTPClient:
    _instance = None
    
    @classmethod
    def get_client(cls):
        if cls._instance is None:
            cls._instance = httpx.AsyncClient(
                timeout=10.0,
                limits=httpx.Limits(max_connections=10)
            )
        return cls._instance
```

### Memory Optimization

**Efficient Data Structures**:
```python
# Use generators for large datasets
def process_weather_data(data):
    for item in data:
        yield transform_item(item)

# Avoid loading large objects in memory
async def stream_response(data):
    async def generate():
        for chunk in data:
            yield json.dumps(chunk) + "\n"
    
    return StreamingResponse(generate(), media_type="application/json")
```

### Request Optimization

**Parallel Requests**:
```python
import asyncio

async def get_multiple_forecasts(locations):
    """Get forecasts for multiple locations in parallel"""
    tasks = [
        get_weather_forecast(lat, lon)
        for lat, lon in locations
    ]
    return await asyncio.gather(*tasks)
```

## Troubleshooting

### Common Issues

**1. Deployment Failures**:
```bash
# Check deployment logs
wrangler deploy --env production --verbose

# Validate configuration
wrangler validate

# Check account/zone permissions
wrangler whoami
```

**2. Runtime Errors**:
```bash
# View real-time logs
wrangler tail --env production

# Check specific error
wrangler tail --env production --search "error"

# Debug locally
wrangler dev --env staging
```

**3. Performance Issues**:
```bash
# Monitor performance
wrangler tail --env production --format json | grep duration

# Check memory usage
wrangler tail --env production --search "memory"
```

**4. API Connectivity**:
```python
# Test API connectivity in worker
async def test_api_connection():
    try:
        async with httpx.AsyncClient() as client:
            response = await client.get("https://api.open-meteo.com/v1/forecast?latitude=0&longitude=0&current=temperature_2m")
            return {"status": "ok", "response_time": response.elapsed.total_seconds()}
    except Exception as e:
        return {"status": "error", "message": str(e)}
```

### Debug Mode

**Local Development**:
```bash
# Run locally with Wrangler
cd deployments/cloudflare
wrangler dev --env staging

# Test locally
curl http://localhost:8787/health
```

**Remote Debugging**:
```bash
# Enable debug logging
wrangler secret put LOG_LEVEL DEBUG --env staging

# View debug logs
wrangler tail --env staging --format pretty
```

### Performance Monitoring

**Response Time Monitoring**:
```python
import time

async def monitor_performance(request, env, ctx):
    start_time = time.time()
    
    try:
        response = await handle_request(request)
        duration = time.time() - start_time
        
        # Log performance metrics
        if duration > 1.0:  # Log slow requests
            print(f"Slow request: {duration:.2f}s - {request.url}")
        
        return response
    except Exception as e:
        duration = time.time() - start_time
        print(f"Failed request: {duration:.2f}s - {request.url} - {e}")
        raise
```

## Security Considerations

### Access Control

**IP Allowlisting**:
```python
ALLOWED_IPS = ["192.168.1.0/24", "10.0.0.0/8"]

async def check_ip_access(request):
    client_ip = request.headers.get("CF-Connecting-IP")
    if not is_ip_allowed(client_ip, ALLOWED_IPS):
        return Response("Access denied", status=403)
```

**Rate Limiting**:
```python
# Simple rate limiting
from collections import defaultdict
import time

request_counts = defaultdict(list)

async def rate_limit(request):
    client_ip = request.headers.get("CF-Connecting-IP")
    now = time.time()
    
    # Clean old requests
    request_counts[client_ip] = [
        req_time for req_time in request_counts[client_ip]
        if now - req_time < 60  # 1 minute window
    ]
    
    # Check rate limit
    if len(request_counts[client_ip]) >= 100:  # 100 requests per minute
        return Response("Rate limit exceeded", status=429)
    
    request_counts[client_ip].append(now)
```

### Input Validation

**Request Validation**:
```python
from pydantic import BaseModel, ValidationError

class MCPRequest(BaseModel):
    jsonrpc: str
    id: int
    method: str
    params: dict

async def validate_request(request_data):
    try:
        return MCPRequest(**request_data)
    except ValidationError as e:
        raise ValueError(f"Invalid request: {e}")
```

## Production Checklist

### Pre-deployment

- [ ] All environment variables configured
- [ ] Secrets properly set
- [ ] Custom domain configured (if needed)
- [ ] SSL/TLS certificates valid
- [ ] Rate limiting implemented
- [ ] Error handling comprehensive
- [ ] Logging configured
- [ ] Monitoring enabled

### Post-deployment

- [ ] Health check endpoint responding
- [ ] MCP tools working correctly
- [ ] Performance within acceptable limits
- [ ] Error rates low
- [ ] Logs showing expected behavior
- [ ] Analytics data flowing
- [ ] Alerts configured

### Monitoring

- [ ] Response time < 1 second (95th percentile)
- [ ] Error rate < 1%
- [ ] Memory usage stable
- [ ] API connectivity healthy
- [ ] Custom domain resolving correctly

## Next Steps

After successful Cloudflare Workers deployment:

1. **Monitor Performance**: Set up alerts and monitoring
2. **Optimize for Scale**: Implement caching and optimization
3. **Secure Access**: Add authentication and rate limiting
4. **Custom Domain**: Configure your own domain
5. **CI/CD Integration**: Automate deployments

For CI/CD automation, see [CI/CD Pipeline Guide](./09-cicd-pipeline.md).
For development guidelines, see [Development Guidelines](./08-development-guidelines.md).