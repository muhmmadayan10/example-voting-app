# Vote Service API

The Vote Service provides a simple REST API for vote submission and health monitoring.

## Base URL

```
http://localhost:5000
```

## Endpoints

### GET /

**Description**: Main voting page  
**Method**: GET  
**Response**: HTML page with voting interface

**Response Headers**:
```
Content-Type: text/html
```

**Example**:
```bash
curl http://localhost:5000/
```

### POST /

**Description**: Submit a vote  
**Method**: POST  
**Content-Type**: `application/x-www-form-urlencoded`

**Request Parameters**:
| Parameter | Type | Description | Required |
|-----------|------|-------------|----------|
| `vote` | string | Vote choice ('a' or 'b') | Yes |

**Request Example**:
```bash
curl -X POST http://localhost:5000/ \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "vote=a"
```

**Response**: 
- **Success**: HTTP 200 with updated voting page
- **Error**: HTTP 500 if Redis is unavailable

### GET /health

**Description**: Health check endpoint  
**Method**: GET  
**Response**: JSON health status

**Response Format**:
```json
{
  "status": "healthy",
  "service": "vote",
  "timestamp": "2024-01-01T00:00:00Z",
  "redis_connection": "ok",
  "version": "1.0.0"
}
```

**Status Codes**:
- `200`: Service is healthy
- `503`: Service is unhealthy (Redis connection failed)

**Example**:
```bash
curl http://localhost:5000/health
```

## Request/Response Examples

### Casting a Vote

**Request**:
```http
POST / HTTP/1.1
Host: localhost:5000
Content-Type: application/x-www-form-urlencoded

vote=a
```

**Response**:
```http
HTTP/1.1 200 OK
Content-Type: text/html

<!DOCTYPE html>
<html>
  <head>
    <title>Vote - VotingTech</title>
  </head>
  <body>
    <h1>Thank you for voting!</h1>
    <!-- Updated voting interface -->
  </body>
</html>
```

### Health Check

**Request**:
```http
GET /health HTTP/1.1
Host: localhost:5000
```

**Response**:
```http
HTTP/1.1 200 OK
Content-Type: application/json

{
  "status": "healthy",
  "service": "vote",
  "timestamp": "2024-10-02T11:50:00Z",
  "redis_connection": "ok",
  "version": "1.0.0"
}
```

## Error Handling

### Redis Connection Failure

If Redis is unavailable:

**Response**:
```http
HTTP/1.1 503 Service Unavailable
Content-Type: application/json

{
  "status": "unhealthy",
  "service": "vote",
  "error": "Redis connection failed",
  "timestamp": "2024-10-02T11:50:00Z"
}
```

### Invalid Vote Value

If an invalid vote value is submitted:

**Response**:
```http
HTTP/1.1 400 Bad Request
Content-Type: text/html

<!-- Error page displayed -->
```

## Integration Details

### Redis Integration

The vote service connects to Redis for:
- **Session storage**: Tracking user voting sessions
- **Vote publishing**: Publishing votes to queue for worker processing

**Redis Keys**:
- `vote:session:{session_id}`: User session data
- `vote:queue`: Vote message queue

### Message Format

Votes published to Redis queue:
```json
{
  "vote": "a",
  "voter_id": "session_abc123",
  "timestamp": "2024-10-02T11:50:00Z",
  "service": "vote"
}
```

## Security Considerations

- **Session-based voting**: Prevents duplicate votes per session
- **Input validation**: Vote values are validated
- **No authentication**: Public voting (by design)
- **Rate limiting**: Consider implementing for production use

## Monitoring and Observability

### Health Check

Use the `/health` endpoint for:
- Kubernetes liveness probes
- Load balancer health checks
- Monitoring system integration

### Metrics to Monitor

- **Response time**: Average response time for voting requests
- **Error rate**: Percentage of failed requests
- **Vote submission rate**: Votes per minute
- **Redis connection status**: Connection health
- **Active sessions**: Number of concurrent users

### Logging

The application logs:
- Vote submissions
- Redis connection events
- Error conditions
- Health check requests

## Testing

### Manual Testing

```bash
# Test voting interface
curl http://localhost:5000/

# Test vote submission
curl -X POST http://localhost:5000/ -d "vote=a"

# Test health check
curl http://localhost:5000/health
```

### Automated Testing

```bash
# Run unit tests
python -m pytest tests/

# Run integration tests with Redis
docker-compose up -d redis
python -m pytest tests/integration/
```