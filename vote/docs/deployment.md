# Deployment Guide

This guide covers deploying the Vote Service in various environments.

## Deployment Options

- [Docker](#docker-deployment)
- [Kubernetes](#kubernetes-deployment)
- [Docker Compose](#docker-compose-deployment)
- [Cloud Platforms](#cloud-deployment)

## Docker Deployment

### Building the Image

```bash
# Build the Docker image
docker build -t vote-service:latest .

# Build with specific tag
docker build -t vote-service:v1.0.0 .

# Build for multi-platform
docker buildx build --platform linux/amd64,linux/arm64 -t vote-service:latest .
```

### Running with Docker

#### Basic Run

```bash
docker run -d \
  --name vote-service \
  -p 5000:80 \
  -e REDIS_HOST=redis \
  -e REDIS_PORT=6379 \
  vote-service:latest
```

#### With Environment File

Create `.env` file:
```env
REDIS_HOST=redis
REDIS_PORT=6379
OPTION_A=Cats
OPTION_B=Dogs
```

Run with environment file:
```bash
docker run -d \
  --name vote-service \
  -p 5000:80 \
  --env-file .env \
  vote-service:latest
```

#### With Networks

```bash
# Create network
docker network create voting-network

# Run with network
docker run -d \
  --name vote-service \
  --network voting-network \
  -p 5000:80 \
  -e REDIS_HOST=redis \
  vote-service:latest
```

## Docker Compose Deployment

### Basic Compose Setup

Create `docker-compose.yml`:

```yaml
version: '3.8'

services:
  vote:
    build: 
      context: .
      dockerfile: Dockerfile
    image: vote-service:latest
    ports:
      - "5000:80"
    environment:
      - REDIS_HOST=redis
      - REDIS_PORT=6379
    depends_on:
      - redis
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost/health"]
      interval: 30s
      timeout: 10s
      retries: 3

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
    restart: unless-stopped
    volumes:
      - redis_data:/data

volumes:
  redis_data:

networks:
  default:
    name: voting-network
```

### Deploy with Compose

```bash
# Start services
docker-compose up -d

# Check status
docker-compose ps

# View logs
docker-compose logs vote

# Scale the service
docker-compose up -d --scale vote=3

# Stop services
docker-compose down
```

### Production Compose

For production, use `docker-compose.prod.yml`:

```yaml
version: '3.8'

services:
  vote:
    image: vote-service:v1.0.0
    ports:
      - "80:80"
    environment:
      - REDIS_HOST=redis-cluster
      - REDIS_PORT=6379
    deploy:
      replicas: 3
      restart_policy:
        condition: on-failure
        delay: 5s
        max_attempts: 3
      resources:
        limits:
          cpus: '0.5'
          memory: 512M
        reservations:
          cpus: '0.25'
          memory: 256M

  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf
      - ./ssl:/etc/nginx/ssl
    depends_on:
      - vote
```

## Kubernetes Deployment

### Namespace

```yaml
# namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: voting-app
```

### ConfigMap

```yaml
# configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: vote-config
  namespace: voting-app
data:
  REDIS_HOST: "redis-service"
  REDIS_PORT: "6379"
  OPTION_A: "Cats"
  OPTION_B: "Dogs"
```

### Deployment

```yaml
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: vote-service
  namespace: voting-app
  labels:
    app: vote
    component: frontend
spec:
  replicas: 3
  selector:
    matchLabels:
      app: vote
  template:
    metadata:
      labels:
        app: vote
        component: frontend
    spec:
      containers:
      - name: vote
        image: vote-service:v1.0.0
        ports:
        - containerPort: 80
          name: http
        envFrom:
        - configMapRef:
            name: vote-config
        resources:
          requests:
            memory: "128Mi"
            cpu: "100m"
          limits:
            memory: "256Mi"
            cpu: "500m"
        livenessProbe:
          httpGet:
            path: /health
            port: 80
          initialDelaySeconds: 30
          periodSeconds: 10
        readinessProbe:
          httpGet:
            path: /health
            port: 80
          initialDelaySeconds: 5
          periodSeconds: 5
```

### Service

```yaml
# service.yaml
apiVersion: v1
kind: Service
metadata:
  name: vote-service
  namespace: voting-app
  labels:
    app: vote
spec:
  selector:
    app: vote
  ports:
  - name: http
    port: 80
    targetPort: 80
  type: ClusterIP
```

### Ingress

```yaml
# ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: vote-ingress
  namespace: voting-app
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
    cert-manager.io/cluster-issuer: "letsencrypt-prod"
spec:
  tls:
  - hosts:
    - vote.votingtech.com
    secretName: vote-tls
  rules:
  - host: vote.votingtech.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: vote-service
            port:
              number: 80
```

### Deploy to Kubernetes

```bash
# Apply all manifests
kubectl apply -f namespace.yaml
kubectl apply -f configmap.yaml
kubectl apply -f deployment.yaml
kubectl apply -f service.yaml
kubectl apply -f ingress.yaml

# Check deployment
kubectl get pods -n voting-app
kubectl get svc -n voting-app
kubectl get ingress -n voting-app

# View logs
kubectl logs -f deployment/vote-service -n voting-app

# Scale deployment
kubectl scale deployment vote-service --replicas=5 -n voting-app
```

### Helm Chart

Create a Helm chart for easier deployment:

```bash
helm create vote-service
```

`values.yaml`:
```yaml
replicaCount: 3

image:
  repository: vote-service
  tag: v1.0.0
  pullPolicy: IfNotPresent

service:
  type: ClusterIP
  port: 80

ingress:
  enabled: true
  className: "nginx"
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
  hosts:
    - host: vote.votingtech.com
      paths:
        - path: /
          pathType: Prefix
  tls:
    - secretName: vote-tls
      hosts:
        - vote.votingtech.com

resources:
  limits:
    cpu: 500m
    memory: 256Mi
  requests:
    cpu: 100m
    memory: 128Mi

env:
  REDIS_HOST: redis-service
  REDIS_PORT: "6379"
```

Deploy with Helm:
```bash
helm install vote-service ./vote-service -n voting-app
helm upgrade vote-service ./vote-service -n voting-app
```

## Cloud Deployment

### AWS ECS

Create task definition:

```json
{
  "family": "vote-service",
  "networkMode": "awsvpc",
  "requiresCompatibilities": ["FARGATE"],
  "cpu": "256",
  "memory": "512",
  "executionRoleArn": "arn:aws:iam::ACCOUNT:role/ecsTaskExecutionRole",
  "containerDefinitions": [
    {
      "name": "vote",
      "image": "ACCOUNT.dkr.ecr.REGION.amazonaws.com/vote-service:latest",
      "portMappings": [
        {
          "containerPort": 80,
          "protocol": "tcp"
        }
      ],
      "environment": [
        {
          "name": "REDIS_HOST",
          "value": "redis.cluster.cache.amazonaws.com"
        }
      ],
      "logConfiguration": {
        "logDriver": "awslogs",
        "options": {
          "awslogs-group": "/ecs/vote-service",
          "awslogs-region": "us-west-2",
          "awslogs-stream-prefix": "ecs"
        }
      }
    }
  ]
}
```

### Google Cloud Run

Deploy to Cloud Run:

```bash
# Build and push to Container Registry
gcloud builds submit --tag gcr.io/PROJECT_ID/vote-service

# Deploy to Cloud Run
gcloud run deploy vote-service \
  --image gcr.io/PROJECT_ID/vote-service \
  --platform managed \
  --region us-central1 \
  --allow-unauthenticated \
  --set-env-vars REDIS_HOST=redis-memorystore-ip
```

### Azure Container Instances

```bash
# Create resource group
az group create --name voting-app --location eastus

# Deploy container
az container create \
  --resource-group voting-app \
  --name vote-service \
  --image vote-service:latest \
  --dns-name-label vote-votingtech \
  --ports 80 \
  --environment-variables REDIS_HOST=redis-cache.redis.cache.windows.net
```

## Environment Configuration

### Development

```env
FLASK_ENV=development
FLASK_DEBUG=1
REDIS_HOST=localhost
REDIS_PORT=6379
```

### Staging

```env
FLASK_ENV=production
REDIS_HOST=redis-staging
REDIS_PORT=6379
OPTION_A=Feature_A
OPTION_B=Feature_B
```

### Production

```env
FLASK_ENV=production
REDIS_HOST=redis-cluster
REDIS_PORT=6379
REDIS_PASSWORD=secure_password
OPTION_A=Cats
OPTION_B=Dogs
```

## Security Considerations

### Container Security

```dockerfile
# Use non-root user
RUN addgroup -g 1001 -S appuser && \
    adduser -u 1001 -S appuser -G appuser
USER appuser

# Use minimal base image
FROM python:3.11-alpine

# Don't run as root
USER 1001:1001
```

### Network Security

- Use internal networks for service communication
- Implement TLS/SSL for external traffic
- Configure firewall rules
- Use secrets management for sensitive data

### Secrets Management

#### Kubernetes Secrets

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: vote-secrets
  namespace: voting-app
type: Opaque
data:
  redis-password: <base64-encoded-password>
```

#### Docker Secrets

```bash
echo "redis_password" | docker secret create redis_password -
```

## Monitoring and Logging

### Health Checks

Configure health checks for your deployment:

```yaml
healthcheck:
  test: ["CMD", "curl", "-f", "http://localhost/health"]
  interval: 30s
  timeout: 10s
  retries: 3
  start_period: 40s
```

### Logging

Configure centralized logging:

```yaml
logging:
  driver: "fluentd"
  options:
    fluentd-address: localhost:24224
    tag: vote-service
```

### Metrics

Expose metrics for monitoring:

```python
from prometheus_client import Counter, Histogram
from flask import Flask

vote_counter = Counter('votes_total', 'Total votes cast')
request_duration = Histogram('request_duration_seconds', 'Request duration')
```

## Scaling

### Horizontal Scaling

```bash
# Docker Compose
docker-compose up -d --scale vote=5

# Kubernetes
kubectl scale deployment vote-service --replicas=10

# Auto-scaling
kubectl autoscale deployment vote-service --cpu-percent=70 --min=3 --max=10
```

### Load Balancing

Configure load balancer for multiple instances:

```nginx
upstream vote_backend {
    server vote-1:80;
    server vote-2:80;
    server vote-3:80;
}

server {
    listen 80;
    location / {
        proxy_pass http://vote_backend;
    }
}
```

## Troubleshooting

### Common Issues

1. **Service won't start**
   ```bash
   # Check logs
   docker logs vote-service
   kubectl logs deployment/vote-service
   ```

2. **Can't connect to Redis**
   ```bash
   # Test Redis connectivity
   docker exec vote-service redis-cli -h redis ping
   ```

3. **Performance issues**
   ```bash
   # Monitor resources
   docker stats vote-service
   kubectl top pods
   ```

### Debugging

Enable debug mode for troubleshooting:

```bash
# Environment variable
export FLASK_DEBUG=1

# In Kubernetes
kubectl set env deployment/vote-service FLASK_DEBUG=1
```

## Rollback Strategy

### Docker

```bash
# Tag current version
docker tag vote-service:latest vote-service:backup

# Rollback to previous version
docker stop vote-service
docker rm vote-service
docker run -d --name vote-service vote-service:v1.0.0
```

### Kubernetes

```bash
# View rollout history
kubectl rollout history deployment/vote-service

# Rollback to previous version
kubectl rollout undo deployment/vote-service

# Rollback to specific revision
kubectl rollout undo deployment/vote-service --to-revision=2
```

## Backup and Recovery

### Data Backup

Since this service is stateless, focus on:
- Configuration backup
- Redis data backup (if needed)
- Container images backup

### Disaster Recovery

1. **Service recovery**
   ```bash
   # Restart service
   kubectl delete pod -l app=vote
   docker-compose restart vote
   ```

2. **Full environment recovery**
   ```bash
   # Redeploy from scratch
   kubectl apply -f manifests/
   docker-compose up -d
   ```

## Performance Optimization

### Container Optimization

```dockerfile
# Multi-stage build
FROM python:3.11-slim as builder
COPY requirements.txt .
RUN pip install --user -r requirements.txt

FROM python:3.11-slim
COPY --from=builder /root/.local /root/.local
COPY . .
CMD ["python", "app.py"]
```

### Resource Limits

```yaml
resources:
  requests:
    memory: "64Mi"
    cpu: "50m"
  limits:
    memory: "128Mi"
    cpu: "100m"
```

This deployment guide covers all major deployment scenarios for the Vote Service. Choose the method that best fits your infrastructure and requirements.