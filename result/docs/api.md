# Result Service API Documentation

## Overview

The Result Service provides a real-time web interface for displaying voting results. It exposes both HTTP endpoints for serving the web interface and WebSocket connections for real-time updates.

## HTTP Endpoints

### GET /
**Description**: Serves the main voting results web interface

**Response**: HTML page with real-time voting visualization

**Content-Type**: `text/html`

**Features**:
- Real-time vote percentage display
- Responsive design for mobile and desktop
- WebSocket integration for live updates
- Vote count statistics

**Example Response**:
```html
<!DOCTYPE html>
<html ng-app="catsvsdogs">
  <head>
    <title>Cats vs Dogs -- Result</title>
    <!-- Voting results interface -->
  </head>
  <body ng-controller="statsCtrl">
    <!-- Live voting visualization -->
  </body>
</html>
```

### Static Assets
**Path**: `/stylesheets/*`, `/angular.min.js`, `/app.js`, `/socket.io.js`

**Description**: Serves static frontend assets including:
- CSS stylesheets for UI styling
- AngularJS framework
- Custom application JavaScript
- Socket.IO client library

## WebSocket API

### Connection Endpoint
**URL**: `ws://localhost:4000/socket.io/`

**Protocol**: Socket.IO WebSocket implementation

### Events

#### Client → Server Events

##### `subscribe`
**Description**: Subscribe to a specific channel for targeted updates

**Payload**:
```json
{
  "channel": "string"
}
```

**Example**:
```javascript
socket.emit('subscribe', { channel: 'votes' });
```

#### Server → Client Events

##### `message`
**Description**: Welcome message sent upon connection

**Payload**:
```json
{
  "text": "Welcome!"
}
```

**Trigger**: Automatically sent when client connects

##### `scores`
**Description**: Real-time vote count and percentage updates

**Payload**: JSON string containing vote tallies
```json
{
  "a": 45,  // Vote count for option A (Cats)
  "b": 32   // Vote count for option B (Dogs)
}
```

**Frequency**: Updates every 1 second when vote data changes

**Example**:
```javascript
socket.on('scores', function(data) {
  const votes = JSON.parse(data);
  console.log('Cats:', votes.a, 'Dogs:', votes.b);
});
```

## Database Schema

### Votes Table
The service reads from the `votes` table in PostgreSQL:

```sql
CREATE TABLE votes (
  id SERIAL PRIMARY KEY,
  vote VARCHAR(1) NOT NULL,  -- 'a' for Cats, 'b' for Dogs
  voter_id VARCHAR(255) NOT NULL,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

### Query Used
```sql
SELECT vote, COUNT(id) AS count 
FROM votes 
GROUP BY vote;
```

**Result Format**:
```
 vote | count 
------+-------
 a    |    45
 b    |    32
```

## Frontend JavaScript API

### AngularJS Controller
The frontend uses an AngularJS controller (`statsCtrl`) that manages:

#### Scope Variables
- `$scope.aPercent`: Percentage of votes for option A
- `$scope.bPercent`: Percentage of votes for option B  
- `$scope.total`: Total number of votes cast

#### Functions

##### `updateScores()`
Handles incoming vote data via WebSocket:
```javascript
socket.on('scores', function (json) {
   data = JSON.parse(json);
   var a = parseInt(data.a || 0);
   var b = parseInt(data.b || 0);
   
   var percentages = getPercentages(a, b);
   
   // Update UI elements
   bg1.style.width = percentages.a + "%";
   bg2.style.width = percentages.b + "%";
   
   $scope.$apply(function () {
     $scope.aPercent = percentages.a;
     $scope.bPercent = percentages.b;
     $scope.total = a + b;
   });
});
```

##### `getPercentages(a, b)`
Calculates vote percentages:
```javascript
function getPercentages(a, b) {
  var result = {};
  
  if (a + b > 0) {
    result.a = Math.round(a / (a + b) * 100);
    result.b = 100 - result.a;
  } else {
    result.a = result.b = 50;  // Default when no votes
  }
  
  return result;
}
```

## Error Handling

### Database Connection Errors
- Automatic retry with exponential backoff
- Graceful degradation when database is unavailable
- Error logging for debugging

### WebSocket Connection Errors
- Automatic reconnection on connection loss
- Client-side error handling
- Fallback display states

## Rate Limiting

**Current Implementation**: None

**Recommended**: Implement rate limiting for:
- WebSocket connections per IP
- Database query frequency
- Static asset requests

## CORS Configuration

**Current Status**: Not explicitly configured

**Recommendation**: Configure CORS headers for cross-origin requests:
```javascript
app.use(function(req, res, next) {
  res.header("Access-Control-Allow-Origin", "*");
  res.header("Access-Control-Allow-Headers", "Content-Type");
  next();
});
```

## Authentication

**Current Implementation**: None - public access

**Security Note**: In production environments, consider implementing:
- API key authentication
- JWT token validation
- Role-based access control
- Session management