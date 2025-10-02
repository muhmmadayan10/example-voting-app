# Result Service Deployment Guide

## Overview

This guide covers deployment options for the Result Service, including Docker containers, Kubernetes, and local development setups.

## Docker Deployment

### Using Docker Compose (Recommended)

The Result Service is designed to work within the complete voting app ecosystem using Docker Compose.

#### Prerequisites
- Docker Desktop installed
- Docker Compose available

#### Quick Start
```bash
# Clone the repository
git clone <repository-url>
cd example-voting-app

# Build and start all services
docker compose up

# Access the results at http://localhost:8081
```

#### Service Configuration in docker-compose.yml
```yaml
result:
  build: ./result
  ports:
    - "8081:80"
  depends_on:
    - db
  environment:
    - PORT=80
  networks:
    - back-tier
```

### Standalone Docker Container

#### Building the Image
```bash
cd result/
docker build -t voting-result .
```

#### Running the Container
```bash
# Run with PostgreSQL dependency
docker run -d \
  --name voting-result \
  -p 4000:80 \
  -e PORT=80 \
  --link postgres-db:db \
  voting-result
```

#### Multi-stage Build (Production)
```dockerfile
# Production-optimized Dockerfile
FROM node:18-slim AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production

FROM node:18-slim AS runtime
RUN apt-get update && \
    apt-get install -y --no-install-recommends curl tini && \
    rm -rf /var/lib/apt/lists/*

WORKDIR /usr/local/app
COPY --from=builder /app/node_modules ./node_modules
COPY . .

EXPOSE 80
ENTRYPOINT ["/usr/bin/tini", "--"]
CMD ["node", "server.js"]
```

## Kubernetes Deployment

### Deployment Manifest
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: result
  labels:
    app: result
spec:
  replicas: 2
  selector:
    matchLabels:
      app: result
  template:
    metadata:
      labels:
        app: result
    spec:
      containers:
      - name: result
        image: voting-result:latest
        ports:
        - containerPort: 80
        env:
        - name: PORT
          value: "80"
        resources:
          requests:
            memory: "64Mi"
            cpu: "50m"
          limits:
            memory: "128Mi"
            cpu: "100m"
        livenessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /
            port: 80
          initialDelaySeconds: 5
          periodSeconds: 5
```

### Service Manifest
```yaml
apiVersion: v1
kind: Service
metadata:
  name: result
  labels:
    app: result
spec:
  type: LoadBalancer
  ports:
  - port: 80
    targetPort: 80
    nodePort: 31001
  selector:
    app: result
```

### ConfigMap for Environment Variables
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: result-config
data:
  PORT: "80"
  DB_HOST: "db"
  DB_USER: "postgres"
  DB_PASSWORD: "postgres"
  DB_NAME: "postgres"
```

### Applying Kubernetes Manifests
```bash
# Create namespace (optional)
kubectl create namespace voting-app

# Apply all manifests
kubectl apply -f k8s-specifications/ -n voting-app

# Check deployment status
kubectl get pods -n voting-app
kubectl get services -n voting-app

# Access the service
kubectl port-forward service/result 8081:80 -n voting-app
```

## Local Development

### Prerequisites
- Node.js 18+ installed
- PostgreSQL database running
- Redis instance (for complete voting app)

### Development Setup
```bash
# Navigate to result directory
cd result/

# Install dependencies
npm install

# Install development dependencies
npm install -g nodemon

# Set environment variables
export PORT=4000
export DB_HOST=localhost
export DB_USER=postgres
export DB_PASSWORD=postgres
export DB_NAME=voting_db

# Start development server with auto-reload
nodemon server.js
```

### Database Setup for Local Development
```sql
-- Create database
CREATE DATABASE voting_db;

-- Connect to database
\c voting_db;

-- Create votes table
CREATE TABLE votes (
  id SERIAL PRIMARY KEY,
  vote VARCHAR(1) NOT NULL,
  voter_id VARCHAR(255) NOT NULL,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

-- Insert sample data
INSERT INTO votes (vote, voter_id) VALUES 
  ('a', 'user1'),
  ('b', 'user2'),
  ('a', 'user3'),
  ('a', 'user4'),
  ('b', 'user5');
```

### Environment Variables

| Variable | Default | Description |
|----------|---------|-------------|
| `PORT` | 4000 | Port for the web server |
| `DB_HOST` | db | PostgreSQL host |
| `DB_USER` | postgres | Database username |
| `DB_PASSWORD` | postgres | Database password |
| `DB_NAME` | postgres | Database name |

## Production Deployment

### Security Hardening
```dockerfile
# Use non-root user
FROM node:18-slim
RUN groupadd -r nodejs && useradd -r -g nodejs nodejs
USER nodejs

# Security updates
RUN apt-get update && \
    apt-get upgrade -y && \
    apt-get install -y --no-install-recommends curl tini && \
    rm -rf /var/lib/apt/lists/*

# Remove development dependencies
RUN npm ci --only=production && \
    npm cache clean --force
```

### Load Balancing with NGINX
```nginx
upstream result_backend {
    server result:80;
    server result-2:80;
    server result-3:80;
}

server {
    listen 80;
    server_name results.voting-app.com;

    location / {
        proxy_pass http://result_backend;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    location /socket.io/ {
        proxy_pass http://result_backend;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

### Health Checks
```yaml
# Docker Compose health check
result:
  build: ./result
  healthcheck:
    test: ["CMD", "curl", "-f", "http://localhost:80/"]
    interval: 30s
    timeout: 10s
    retries: 3
    start_period: 40s
```

### Monitoring Setup
```yaml
# Prometheus monitoring
- job_name: 'voting-result'
  static_configs:
    - targets: ['result:80']
  metrics_path: '/metrics'
  scrape_interval: 15s
```

## Scaling Considerations

### Horizontal Scaling
- Stateless design allows multiple instances
- Load balancer required for WebSocket session affinity
- Database connection pooling per instance

### Database Optimization
- Read replicas for vote counting queries
- Connection pooling configuration
- Index optimization for vote aggregation

### Caching Strategy
```javascript
// Redis caching implementation
const redis = require('redis');
const client = redis.createClient({ host: 'redis' });

function getCachedVotes() {
  return new Promise((resolve, reject) => {
    client.get('vote_counts', (err, result) => {
      if (err) reject(err);
      resolve(result ? JSON.parse(result) : null);
    });
  });
}

function setCachedVotes(votes) {
  client.setex('vote_counts', 5, JSON.stringify(votes)); // 5 second cache
}
```

## Troubleshooting

### Common Issues

#### Database Connection Failed
```bash
# Check database connectivity
docker logs <postgres-container>
docker exec -it <postgres-container> psql -U postgres

# Verify network connectivity
docker network ls
docker network inspect <network-name>
```

#### WebSocket Connection Issues
```javascript
// Client-side debugging
socket.on('connect_error', function(error) {
  console.log('Connection error:', error);
});

socket.on('disconnect', function(reason) {
  console.log('Disconnected:', reason);
});
```

#### Performance Issues
```bash
# Monitor container resources
docker stats

# Check application logs
docker logs -f voting-result

# Database query analysis
EXPLAIN ANALYZE SELECT vote, COUNT(id) AS count FROM votes GROUP BY vote;
```