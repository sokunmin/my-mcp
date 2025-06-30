# Docker Deployment Guide

## Overview

This guide provides comprehensive instructions for deploying the Weather Forecast MCP Server using Docker containers. Docker deployment is ideal for production environments, cloud deployments, and scenarios where you need HTTP/SSE transport instead of stdio.

## Prerequisites

### System Requirements
- **Docker**: 20.10 or higher
- **Docker Compose**: 2.0 or higher (optional but recommended)
- **Git**: For cloning the repository
- **Internet Connection**: For pulling base images and accessing weather APIs

### Installing Docker

**On Windows**:
1. Download Docker Desktop from [docker.com](https://www.docker.com/products/docker-desktop)
2. Install and start Docker Desktop
3. Verify installation: `docker --version`

**On macOS**:
```bash
# Using Homebrew
brew install --cask docker

# Or download from docker.com
```

**On Linux (Ubuntu/Debian)**:
```bash
# Install Docker
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh

# Add user to docker group
sudo usermod -aG docker $USER

# Install Docker Compose
sudo apt-get install docker-compose-plugin
```

### Verify Installation
```bash
docker --version
docker compose version
```

## Quick Start

### 1. Clone and Build
```bash
# Clone the repository
git clone <repository-url>
cd weather-mcp-server

# Build the Docker image
docker build -f deployments/docker/Dockerfile -t weather-mcp-server .
```

### 2. Run with Docker
```bash
# Run the container
docker run -p 8000:8000 weather-mcp-server
```

### 3. Test the Deployment
```bash
# Test health endpoint
curl http://localhost:8000/health

# Test MCP endpoint
curl -X POST http://localhost:8000/mcp \
  -H "Content-Type: application/json" \
  -d '{
    "jsonrpc": "2.0",
    "id": 1,
    "method": "tools/list"
  }'
```

## Docker Configuration

### Dockerfile

The multi-stage Dockerfile optimizes for both development and production:

```dockerfile
# deployments/docker/Dockerfile
FROM python:3.12-slim as base

# Set environment variables
ENV PYTHONUNBUFFERED=1 \
    PYTHONDONTWRITEBYTECODE=1 \
    PIP_NO_CACHE_DIR=1 \
    PIP_DISABLE_PIP_VERSION_CHECK=1

# Install system dependencies
RUN apt-get update && apt-get install -y \
    curl \
    && rm -rf /var/lib/apt/lists/*

# Create app directory
WORKDIR /app

# Development stage
FROM base as development

# Copy requirements first for better caching
COPY requirements.txt .
RUN pip install -r requirements.txt

# Copy source code
COPY src/ ./src/
COPY deployments/docker/entrypoint.py .

# Create non-root user
RUN useradd -m -u 1000 weather && chown -R weather:weather /app
USER weather

# Expose port
EXPOSE 8000

# Health check
HEALTHCHECK --interval=30s --timeout=10s --start-period=5s --retries=3 \
    CMD curl -f http://localhost:8000/health || exit 1

# Default command
CMD ["python", "entrypoint.py"]

# Production stage
FROM base as production

# Copy only necessary files
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Copy application
COPY src/ ./src/
COPY deployments/docker/entrypoint.py .

# Create non-root user
RUN useradd -m -u 1000 weather && chown -R weather:weather /app
USER weather

# Expose port
EXPOSE 8000

# Health check
HEALTHCHECK --interval=30s --timeout=10s --start-period=5s --retries=3 \
    CMD curl -f http://localhost:8000/health || exit 1

# Run application
CMD ["python", "entrypoint.py"]
```

### Entry Point

The Docker entry point configures the server for containerized deployment:

```python
# deployments/docker/entrypoint.py
import asyncio
import logging
import os
import signal
import sys
from typing import Optional

from weather_mcp.server import create_mcp_server

# Global server instance for graceful shutdown
server_instance: Optional[object] = None

def setup_logging():
    """Configure logging for containerized environment"""
    log_level = os.getenv('LOG_LEVEL', 'INFO').upper()
    
    # Configure structured logging for containers
    logging.basicConfig(
        level=getattr(logging, log_level, logging.INFO),
        format='%(asctime)s - %(name)s - %(levelname)s - %(message)s',
        handlers=[logging.StreamHandler(sys.stdout)]
    )
    
    # Disable unnecessary logging
    logging.getLogger('httpx').setLevel(logging.WARNING)
    logging.getLogger('httpcore').setLevel(logging.WARNING)

def setup_signal_handlers():
    """Set up signal handlers for graceful shutdown"""
    def signal_handler(signum, frame):
        logging.info(f"Received signal {signum}, initiating graceful shutdown...")
        if server_instance:
            # Implement graceful shutdown logic
            pass
        sys.exit(0)
    
    signal.signal(signal.SIGTERM, signal_handler)
    signal.signal(signal.SIGINT, signal_handler)

async def health_check_handler(request):
    """Health check endpoint for container orchestration"""
    return {
        "status": "healthy",
        "service": "weather-mcp-server",
        "version": "1.0.0",
        "timestamp": datetime.utcnow().isoformat()
    }

async def main():
    """Main application entry point"""
    setup_logging()
    setup_signal_handlers()
    
    logger = logging.getLogger(__name__)
    logger.info("Starting Weather MCP Server (Docker)")
    
    # Environment validation
    required_env_vars = ['OPEN_METEO_BASE_URL']
    missing_vars = [var for var in required_env_vars if not os.getenv(var)]
    if missing_vars:
        logger.error(f"Missing required environment variables: {missing_vars}")
        sys.exit(1)
    
    try:
        # Create and configure server
        global server_instance
        server_instance = create_mcp_server()
        
        # Add health check endpoint
        server_instance.add_route('/health', health_check_handler, methods=['GET'])
        
        logger.info("Server configuration:")
        logger.info(f"  Transport: HTTP")
        logger.info(f"  Host: 0.0.0.0")
        logger.info(f"  Port: 8000")
        logger.info(f"  Tools: {len(server_instance.tools)}")
        
        # Start server
        await server_instance.run(
            transport="http",
            host="0.0.0.0",
            port=8000
        )
        
    except Exception as e:
        logger.error(f"Failed to start server: {e}")
        sys.exit(1)

if __name__ == "__main__":
    asyncio.run(main())
```

## Docker Compose

### Basic Configuration

```yaml
# deployments/docker/docker-compose.yml
version: '3.8'

services:
  weather-mcp:
    build:
      context: ../..
      dockerfile: deployments/docker/Dockerfile
      target: production
    ports:
      - "8000:8000"
    environment:
      - OPEN_METEO_BASE_URL=https://api.open-meteo.com/v1
      - GEOCODING_BASE_URL=https://geocoding-api.open-meteo.com/v1
      - REQUEST_TIMEOUT=10.0
      - LOG_LEVEL=INFO
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8000/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s
    restart: unless-stopped
    networks:
      - weather-network

networks:
  weather-network:
    driver: bridge
```

### Development Configuration

```yaml
# deployments/docker/docker-compose.dev.yml
version: '3.8'

services:
  weather-mcp-dev:
    build:
      context: ../..
      dockerfile: deployments/docker/Dockerfile
      target: development
    ports:
      - "8000:8000"
    environment:
      - OPEN_METEO_BASE_URL=https://api.open-meteo.com/v1
      - GEOCODING_BASE_URL=https://geocoding-api.open-meteo.com/v1
      - REQUEST_TIMEOUT=10.0
      - LOG_LEVEL=DEBUG
    volumes:
      - ../../src:/app/src:ro  # Mount source for development
      - ../../tests:/app/tests:ro
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8000/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s
    networks:
      - weather-network

networks:
  weather-network:
    driver: bridge
```

### Production Configuration with Reverse Proxy

```yaml
# deployments/docker/docker-compose.prod.yml
version: '3.8'

services:
  weather-mcp:
    build:
      context: ../..
      dockerfile: deployments/docker/Dockerfile
      target: production
    environment:
      - OPEN_METEO_BASE_URL=https://api.open-meteo.com/v1
      - GEOCODING_BASE_URL=https://geocoding-api.open-meteo.com/v1
      - REQUEST_TIMEOUT=10.0
      - LOG_LEVEL=INFO
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8000/health"]
      interval: 30s
      timeout: 10s
      retries: 3
      start_period: 40s
    restart: unless-stopped
    networks:
      - weather-network
    depends_on:
      - nginx

  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
      - ./ssl:/etc/nginx/ssl:ro
    networks:
      - weather-network
    restart: unless-stopped

networks:
  weather-network:
    driver: bridge
```

## Deployment Commands

### Basic Deployment

```bash
# Build and run with Docker
docker build -f deployments/docker/Dockerfile -t weather-mcp-server .
docker run -d -p 8000:8000 --name weather-mcp weather-mcp-server

# Or use Docker Compose
cd deployments/docker
docker compose up -d
```

### Development Deployment

```bash
# Run in development mode with volume mounts
docker compose -f docker-compose.dev.yml up -d

# View logs
docker compose -f docker-compose.dev.yml logs -f

# Rebuild after code changes
docker compose -f docker-compose.dev.yml up --build -d
```

### Production Deployment

```bash
# Deploy production configuration
docker compose -f docker-compose.prod.yml up -d

# Scale the service
docker compose -f docker-compose.prod.yml up -d --scale weather-mcp=3

# Update deployment
docker compose -f docker-compose.prod.yml pull
docker compose -f docker-compose.prod.yml up -d
```

## Environment Configuration

### Environment Variables

**Required Variables**:
```bash
OPEN_METEO_BASE_URL=https://api.open-meteo.com/v1
GEOCODING_BASE_URL=https://geocoding-api.open-meteo.com/v1
```

**Optional Variables**:
```bash
REQUEST_TIMEOUT=10.0          # API request timeout in seconds
LOG_LEVEL=INFO                # Logging level (DEBUG, INFO, WARNING, ERROR)
MAX_FORECAST_DAYS=16          # Maximum forecast days allowed
MAX_HOURLY_HOURS=168          # Maximum hourly forecast hours
HEALTH_CHECK_INTERVAL=30      # Health check interval in seconds
```

### Environment Files

**Production (.env.prod)**:
```bash
OPEN_METEO_BASE_URL=https://api.open-meteo.com/v1
GEOCODING_BASE_URL=https://geocoding-api.open-meteo.com/v1
REQUEST_TIMEOUT=10.0
LOG_LEVEL=INFO
MAX_FORECAST_DAYS=16
MAX_HOURLY_HOURS=168
```

**Development (.env.dev)**:
```bash
OPEN_METEO_BASE_URL=https://api.open-meteo.com/v1
GEOCODING_BASE_URL=https://geocoding-api.open-meteo.com/v1
REQUEST_TIMEOUT=15.0
LOG_LEVEL=DEBUG
MAX_FORECAST_DAYS=16
MAX_HOURLY_HOURS=168
```

**Load Environment File**:
```bash
# With Docker Compose
docker compose --env-file .env.prod up -d

# With Docker run
docker run --env-file .env.prod -p 8000:8000 weather-mcp-server
```

## Networking and Security

### Network Configuration

**Custom Network**:
```bash
# Create custom network
docker network create weather-network

# Run container on custom network
docker run -d --network weather-network -p 8000:8000 weather-mcp-server
```

**Internal Communication**:
```yaml
# docker-compose.yml
services:
  weather-mcp:
    networks:
      - internal
      - external
  
  database:
    networks:
      - internal

networks:
  internal:
    internal: true
  external:
```

### Security Configuration

**Non-root User**:
```dockerfile
# Create and use non-root user
RUN useradd -m -u 1000 weather && chown -R weather:weather /app
USER weather
```

**Security Options**:
```bash
# Run with security options
docker run -d \
  --security-opt no-new-privileges:true \
  --cap-drop ALL \
  --read-only \
  --tmpfs /tmp \
  -p 8000:8000 \
  weather-mcp-server
```

**Resource Limits**:
```yaml
# docker-compose.yml
services:
  weather-mcp:
    deploy:
      resources:
        limits:
          cpus: '0.5'
          memory: 512M
        reservations:
          cpus: '0.25'
          memory: 256M
```

## Monitoring and Logging

### Health Checks

**Container Health Check**:
```dockerfile
HEALTHCHECK --interval=30s --timeout=10s --start-period=5s --retries=3 \
    CMD curl -f http://localhost:8000/health || exit 1
```

**Custom Health Check**:
```python
async def detailed_health_check():
    """Comprehensive health check"""
    health_status = {
        "status": "healthy",
        "timestamp": datetime.utcnow().isoformat(),
        "checks": {}
    }
    
    # Check API connectivity
    try:
        async with httpx.AsyncClient(timeout=5.0) as client:
            response = await client.get(
                "https://api.open-meteo.com/v1/forecast",
                params={"latitude": 0, "longitude": 0, "current": "temperature_2m"}
            )
            health_status["checks"]["api"] = "healthy" if response.status_code == 200 else "unhealthy"
    except Exception:
        health_status["checks"]["api"] = "unhealthy"
    
    # Check memory usage
    import psutil
    memory_percent = psutil.virtual_memory().percent
    health_status["checks"]["memory"] = "healthy" if memory_percent < 80 else "warning"
    
    return health_status
```

### Logging Configuration

**Structured Logging**:
```python
import structlog

def configure_logging():
    structlog.configure(
        processors=[
            structlog.stdlib.filter_by_level,
            structlog.stdlib.add_logger_name,
            structlog.stdlib.add_log_level,
            structlog.stdlib.PositionalArgumentsFormatter(),
            structlog.processors.TimeStamper(fmt="iso"),
            structlog.processors.StackInfoRenderer(),
            structlog.processors.format_exc_info,
            structlog.processors.UnicodeDecoder(),
            structlog.processors.JSONRenderer()
        ],
        context_class=dict,
        logger_factory=structlog.stdlib.LoggerFactory(),
        wrapper_class=structlog.stdlib.BoundLogger,
        cache_logger_on_first_use=True,
    )
```

**Log Aggregation**:
```yaml
# docker-compose.yml with logging driver
services:
  weather-mcp:
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"
    # Or use external logging
    logging:
      driver: "syslog"
      options:
        syslog-address: "tcp://logserver:514"
```

### Metrics Collection

**Prometheus Metrics**:
```python
from prometheus_client import Counter, Histogram, generate_latest

# Metrics
REQUEST_COUNT = Counter('weather_requests_total', 'Total weather requests', ['tool', 'status'])
REQUEST_DURATION = Histogram('weather_request_duration_seconds', 'Request duration')

async def metrics_handler(request):
    """Prometheus metrics endpoint"""
    return Response(generate_latest(), media_type="text/plain")
```

## Troubleshooting

### Common Issues

**1. Container Won't Start**:
```bash
# Check container logs
docker logs weather-mcp

# Check container status
docker ps -a

# Inspect container
docker inspect weather-mcp
```

**2. Health Check Failures**:
```bash
# Test health endpoint manually
docker exec weather-mcp curl -f http://localhost:8000/health

# Check health status
docker inspect --format='{{.State.Health.Status}}' weather-mcp
```

**3. Network Connectivity Issues**:
```bash
# Test external connectivity from container
docker exec weather-mcp curl -I https://api.open-meteo.com

# Check network configuration
docker network ls
docker network inspect weather-network
```

**4. Performance Issues**:
```bash
# Check resource usage
docker stats weather-mcp

# Check container processes
docker exec weather-mcp ps aux

# Monitor logs for errors
docker logs -f weather-mcp
```

### Debug Mode

**Enable Debug Logging**:
```bash
# Set debug environment variable
docker run -e LOG_LEVEL=DEBUG -p 8000:8000 weather-mcp-server

# Or with Docker Compose
LOG_LEVEL=DEBUG docker compose up -d
```

**Interactive Debugging**:
```bash
# Run container interactively
docker run -it --entrypoint /bin/bash weather-mcp-server

# Execute commands in running container
docker exec -it weather-mcp /bin/bash
```

### Performance Optimization

**Multi-stage Build Optimization**:
```dockerfile
# Use specific Python version
FROM python:3.12-slim

# Optimize pip installation
RUN pip install --no-cache-dir --upgrade pip

# Use .dockerignore to exclude unnecessary files
```

**Resource Optimization**:
```yaml
# docker-compose.yml
services:
  weather-mcp:
    deploy:
      resources:
        limits:
          memory: 256M
        reservations:
          memory: 128M
```

## Production Deployment

### Container Registry

**Build and Push to Registry**:
```bash
# Build for multiple platforms
docker buildx build --platform linux/amd64,linux/arm64 \
  -f deployments/docker/Dockerfile \
  -t ghcr.io/your-org/weather-mcp-server:latest \
  --push .

# Pull and run from registry
docker pull ghcr.io/your-org/weather-mcp-server:latest
docker run -d -p 8000:8000 ghcr.io/your-org/weather-mcp-server:latest
```

### Orchestration

**Docker Swarm**:
```yaml
# docker-stack.yml
version: '3.8'

services:
  weather-mcp:
    image: ghcr.io/your-org/weather-mcp-server:latest
    ports:
      - "8000:8000"
    deploy:
      replicas: 3
      update_config:
        parallelism: 1
        delay: 10s
      restart_policy:
        condition: on-failure
    networks:
      - weather-network

networks:
  weather-network:
    driver: overlay
```

**Kubernetes Deployment**:
```yaml
# k8s-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: weather-mcp-server
spec:
  replicas: 3
  selector:
    matchLabels:
      app: weather-mcp-server
  template:
    metadata:
      labels:
        app: weather-mcp-server
    spec:
      containers:
      - name: weather-mcp
        image: ghcr.io/your-org/weather-mcp-server:latest
        ports:
        - containerPort: 8000
        env:
        - name: OPEN_METEO_BASE_URL
          value: "https://api.open-meteo.com/v1"
        livenessProbe:
          httpGet:
            path: /health
            port: 8000
          initialDelaySeconds: 30
          periodSeconds: 10
```

## Next Steps

After successful Docker deployment:

1. **Monitor Performance**: Set up monitoring and alerting
2. **Scale as Needed**: Use container orchestration for scaling
3. **Secure Deployment**: Implement proper security measures
4. **Backup Strategy**: Plan for data and configuration backup
5. **Update Process**: Establish update and rollback procedures

For Cloudflare Workers deployment, see [Cloudflare Deployment Guide](./07-cloudflare-deployment.md).
For CI/CD automation, see [CI/CD Pipeline Guide](./09-cicd-pipeline.md).