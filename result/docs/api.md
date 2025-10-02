# Result Service API

The Result Service provides endpoints for displaying real-time voting results.

## Base URL

```
http://localhost:4000
```

## Endpoints

### GET /

**Description**: Main results page with real-time updates  
**Method**: GET  
**Response**: HTML page displaying voting results

**Response Headers**:
```
Content-Type: text/html
```

**Example**:
```bash
curl http://localhost:4000/
```

**Response**: Interactive HTML page showing:
- Real-time vote counts for each option
- Visual progress bars
- Percentage breakdowns
- Auto-refreshing data

### GET /api/results

**Description**: Get current voting results as JSON  
**Method**: GET  
**Response**: JSON object with vote counts

**Response Format**:
```json
{
  "results": [
    {
      "option": "a", 
      "votes": 15,
      "percentage": 60.0
    },
    {
      "option": "b",
      "votes": 10, 
      "percentage": 40.0
    }
  ],
  "total_votes": 25,
  "last_updated": "2024-10-02T12:00:00Z"
}
```

**Example**:
```bash
curl http://localhost:4000/api/results
```

### GET /health

**Description**: Health check endpoint  
**Method**: GET  
**Response**: JSON health status

**Response Format**:
```json
{
  "status": "healthy",
  "service": "result",
  "timestamp": "2024-10-02T12:00:00Z",
  "database_connection": "ok",
  "version": "1.0.0"
}
```

**Status Codes**:
- `200`: Service is healthy
- `503`: Service is unhealthy (database connection failed)

**Example**:
```bash
curl http://localhost:4000/health
```

## WebSocket Events

The result service uses WebSockets for real-time updates.

### Connection

```javascript
const socket = new WebSocket('ws://localhost:4000');

socket.onopen = function() {
    console.log('Connected to results feed');
};

socket.onmessage = function(event) {
    const data = JSON.parse(event.data);
    updateResults(data);
};
```

### Message Format

**Vote Update Event**:
```json
{
  "type": "vote_update",
  "data": {
    "option": "a",
    "total_votes": 26,
    "results": [
      {"option": "a", "votes": 16, "percentage": 61.5},
      {"option": "b", "votes": 10, "percentage": 38.5}
    ]
  },
  "timestamp": "2024-10-02T12:00:00Z"
}
```

## Database Schema

The service reads from the following PostgreSQL table:

### votes table

```sql
CREATE TABLE votes (
    id SERIAL PRIMARY KEY,
    vote VARCHAR(255) NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

### Sample Queries

**Get vote counts**:
```sql
SELECT vote, COUNT(*) as count 
FROM votes 
GROUP BY vote 
ORDER BY vote;
```

**Get total votes**:
```sql
SELECT COUNT(*) as total FROM votes;
```

## Error Handling

### Database Connection Error

If PostgreSQL is unavailable:

**Response**:
```http
HTTP/1.1 503 Service Unavailable
Content-Type: application/json

{
  "status": "unhealthy",
  "service": "result",
  "error": "Database connection failed",
  "timestamp": "2024-10-02T12:00:00Z"
}
```

### Invalid API Request

For malformed requests:

**Response**:
```http
HTTP/1.1 400 Bad Request
Content-Type: application/json

{
  "error": "Invalid request format",
  "timestamp": "2024-10-02T12:00:00Z"
}
```

## Integration Details

### PostgreSQL Integration

The result service connects to PostgreSQL to:
- **Read vote data**: Query current vote counts
- **Real-time updates**: Poll for changes
- **Statistics**: Calculate percentages and totals

**Connection Configuration**:
```javascript
const pool = new Pool({
  host: process.env.POSTGRES_HOST,
  port: process.env.POSTGRES_PORT,
  user: process.env.POSTGRES_USER,
  password: process.env.POSTGRES_PASSWORD,
  database: process.env.POSTGRES_DB
});
```

### Real-time Updates

The service implements polling to detect vote changes:

```javascript
setInterval(async () => {
  const results = await getVoteResults();
  if (resultsChanged(results)) {
    broadcastUpdate(results);
  }
}, 1000); // Poll every second
```

## Performance Considerations

### Database Optimization

- Use connection pooling for database connections
- Index the `vote` column for faster queries
- Consider read replicas for high-traffic scenarios

### Caching

```javascript
// Cache results for short periods
const cache = new Map();
const CACHE_TTL = 1000; // 1 second

async function getCachedResults() {
  const cached = cache.get('results');
  if (cached && Date.now() - cached.timestamp < CACHE_TTL) {
    return cached.data;
  }
  
  const fresh = await getVoteResults();
  cache.set('results', { data: fresh, timestamp: Date.now() });
  return fresh;
}
```

### WebSocket Management

- Limit concurrent WebSocket connections
- Implement connection cleanup
- Use heartbeat/ping-pong for connection health

## Security Considerations

- **No authentication**: Public results display (by design)
- **Rate limiting**: Consider implementing for API endpoints
- **Input validation**: Validate all database queries
- **Connection security**: Use SSL for database connections

## Monitoring and Observability

### Health Checks

Use the `/health` endpoint for:
- Kubernetes liveness probes
- Load balancer health checks
- Monitoring system integration

### Metrics to Monitor

- **Response time**: Average response time for API requests
- **Database queries**: Query execution time
- **WebSocket connections**: Number of active connections
- **Error rate**: Percentage of failed requests
- **Vote update frequency**: Rate of result changes

### Logging

The application logs:
- Database connection events
- WebSocket connection/disconnection
- API requests
- Error conditions

## Client-Side Integration

### JavaScript Example

```html
<!DOCTYPE html>
<html>
<head>
    <title>Voting Results</title>
</head>
<body>
    <div id="results"></div>
    
    <script>
        // Fetch initial results
        fetch('/api/results')
            .then(response => response.json())
            .then(data => updateDisplay(data));
        
        // Connect to WebSocket for real-time updates
        const socket = new WebSocket('ws://localhost:4000');
        socket.onmessage = function(event) {
            const data = JSON.parse(event.data);
            updateDisplay(data.data);
        };
        
        function updateDisplay(results) {
            const div = document.getElementById('results');
            div.innerHTML = results.results.map(r => 
                `<div>${r.option}: ${r.votes} (${r.percentage}%)</div>`
            ).join('');
        }
    </script>
</body>
</html>
```

### React Component Example

```jsx
import React, { useState, useEffect } from 'react';

function VotingResults() {
  const [results, setResults] = useState(null);
  
  useEffect(() => {
    // Fetch initial results
    fetch('/api/results')
      .then(res => res.json())
      .then(setResults);
    
    // WebSocket connection
    const socket = new WebSocket('ws://localhost:4000');
    socket.onmessage = (event) => {
      const data = JSON.parse(event.data);
      setResults(data.data);
    };
    
    return () => socket.close();
  }, []);
  
  if (!results) return <div>Loading...</div>;
  
  return (
    <div>
      <h2>Voting Results</h2>
      {results.results.map(result => (
        <div key={result.option}>
          {result.option}: {result.votes} ({result.percentage}%)
        </div>
      ))}
      <p>Total votes: {results.total_votes}</p>
    </div>
  );
}
```

## Testing

### Manual Testing

```bash
# Test results page
curl http://localhost:4000/

# Test API endpoint
curl http://localhost:4000/api/results

# Test health check
curl http://localhost:4000/health

# Test WebSocket (using websocat)
websocat ws://localhost:4000
```

### Automated Testing

```javascript
// Example test with Jest
describe('Results API', () => {
  test('should return vote results', async () => {
    const response = await request(app)
      .get('/api/results')
      .expect(200);
    
    expect(response.body).toHaveProperty('results');
    expect(response.body).toHaveProperty('total_votes');
  });
});
```

## Data Flow

1. **Vote Cast**: User votes via Vote Service
2. **Queue Processing**: Worker Service processes vote
3. **Database Update**: Vote stored in PostgreSQL
4. **Result Detection**: Result Service polls database
5. **Real-time Update**: WebSocket broadcasts to clients
6. **Display Update**: Frontend updates vote counts

This API documentation covers all endpoints and integration points for the Result Service.