# Deployment Guide

## Production Deployment

### Docker Deployment

The Result Service can be deployed using Docker:

```dockerfile
FROM node:16-alpine
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production
COPY . .
EXPOSE 4000
CMD ["node", "server.js"]
```

### Environment Configuration

Set these environment variables in production:

```bash
POSTGRES_HOST=your-postgres-host
POSTGRES_DB=votes
POSTGRES_USER=your-postgres-user
POSTGRES_PASSWORD=your-postgres-password
PORT=4000
```

### Kubernetes Deployment

Example Kubernetes deployment:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: result-service
spec:
  replicas: 2
  selector:
    matchLabels:
      app: result-service
  template:
    metadata:
      labels:
        app: result-service
    spec:
      containers:
      - name: result-service
        image: result-service:latest
        ports:
        - containerPort: 4000
        env:
        - name: POSTGRES_HOST
          value: "postgres-service"
        - name: POSTGRES_DB
          value: "votes"
```

### Health Checks

The service provides a health check endpoint:

```bash
GET /health
```

### Monitoring

Monitor these metrics:
- Response time
- Database connection status
- Memory usage
- CPU usage