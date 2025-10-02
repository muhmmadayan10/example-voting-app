# Result Service Development Guide

## Overview

This guide provides comprehensive information for developers working on the Result Service, including development setup, coding standards, testing procedures, and contribution guidelines.

## Development Environment Setup

### Prerequisites
- Node.js 18.x or higher
- npm 8.x or higher
- Docker and Docker Compose
- Git
- PostgreSQL (for local development)

### Local Development Setup

#### 1. Clone and Setup
```bash
# Clone the repository
git clone <repository-url>
cd example-voting-app/result

# Install dependencies
npm install

# Install development tools globally
npm install -g nodemon eslint prettier
```

#### 2. Environment Configuration
Create a `.env` file for local development:
```bash
# .env file
PORT=4000
NODE_ENV=development
DB_HOST=localhost
DB_PORT=5432
DB_USER=postgres
DB_PASSWORD=postgres
DB_NAME=voting_dev

# Enable debugging
DEBUG=*
```

#### 3. Database Setup
```bash
# Start PostgreSQL locally or via Docker
docker run --name postgres-dev \
  -e POSTGRES_PASSWORD=postgres \
  -e POSTGRES_DB=voting_dev \
  -p 5432:5432 \
  -d postgres:13

# Create votes table
psql -h localhost -U postgres -d voting_dev -c "
  CREATE TABLE votes (
    id SERIAL PRIMARY KEY,
    vote VARCHAR(1) NOT NULL,
    voter_id VARCHAR(255) NOT NULL,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
  );
"
```

#### 4. Start Development Server
```bash
# Start with auto-reload
nodemon server.js

# Or with custom configuration
nodemon --watch . --ext js,json --exec "node server.js"
```

## Project Structure

```
result/
├── docs/                    # Documentation
│   ├── index.md            # Service overview
│   ├── api.md              # API documentation
│   ├── architecture.md     # Architecture details
│   ├── deployment.md       # Deployment guide
│   └── development.md      # This file
├── tests/                   # Test files
│   ├── Dockerfile          # Test container
│   ├── render.js           # PhantomJS render test
│   └── tests.sh            # Test runner script
├── views/                   # Frontend assets
│   ├── index.html          # Main HTML template
│   ├── app.js              # AngularJS application
│   ├── angular.min.js      # AngularJS framework
│   ├── socket.io.js        # Socket.IO client
│   └── stylesheets/
│       └── style.css       # Application styles
├── server.js               # Main application server
├── package.json            # Node.js dependencies
├── Dockerfile              # Container configuration
└── .dockerignore           # Docker ignore rules
```

## Code Architecture

### Server Architecture (`server.js`)

#### 1. Dependencies and Initialization
```javascript
var express = require('express'),
    async = require('async'),
    { Pool } = require('pg'),
    cookieParser = require('cookie-parser'),
    app = express(),
    server = require('http').Server(app),
    io = require('socket.io')(server);
```

#### 2. Database Connection Management
```javascript
var pool = new Pool({
  connectionString: 'postgres://postgres:postgres@db/postgres'
});

// Robust connection retry logic
async.retry(
  {times: 1000, interval: 1000},
  function(callback) {
    pool.connect(function(err, client, done) {
      if (err) {
        console.error("Waiting for db");
      }
      callback(err, client);
    });
  },
  function(err, client) {
    if (err) {
      return console.error("Giving up");
    }
    console.log("Connected to db");
    getVotes(client);
  }
);
```

#### 3. Real-time Vote Aggregation
```javascript
function getVotes(client) {
  client.query('SELECT vote, COUNT(id) AS count FROM votes GROUP BY vote', [], function(err, result) {
    if (err) {
      console.error("Error performing query: " + err);
    } else {
      var votes = collectVotesFromResult(result);
      io.sockets.emit("scores", JSON.stringify(votes));
    }
    
    setTimeout(function() {getVotes(client) }, 1000);
  });
}
```

### Frontend Architecture

#### AngularJS Application (`views/app.js`)
```javascript
var app = angular.module('catsvsdogs', []);
var socket = io.connect();

app.controller('statsCtrl', function($scope){
  $scope.aPercent = 50;
  $scope.bPercent = 50;

  var updateScores = function(){
    socket.on('scores', function (json) {
       data = JSON.parse(json);
       var a = parseInt(data.a || 0);
       var b = parseInt(data.b || 0);

       var percentages = getPercentages(a, b);
       
       // Update visual elements
       bg1.style.width = percentages.a + "%";
       bg2.style.width = percentages.b + "%";

       // Update scope
       $scope.$apply(function () {
         $scope.aPercent = percentages.a;
         $scope.bPercent = percentages.b;
         $scope.total = a + b;
       });
    });
  };
});
```

## Coding Standards

### JavaScript Style Guide

#### ESLint Configuration (`.eslintrc.js`)
```javascript
module.exports = {
  env: {
    node: true,
    es2021: true,
    browser: true
  },
  extends: ['eslint:recommended'],
  parserOptions: {
    ecmaVersion: 12,
    sourceType: 'module'
  },
  rules: {
    'indent': ['error', 2],
    'quotes': ['error', 'single'],
    'semi': ['error', 'always'],
    'no-unused-vars': 'warn',
    'no-console': 'off'
  }
};
```

#### Prettier Configuration (`.prettierrc`)
```json
{
  "semi": true,
  "trailingComma": "es5",
  "singleQuote": true,
  "printWidth": 80,
  "tabWidth": 2
}
```

### Naming Conventions
- **Variables**: camelCase (`voteCount`, `clientSocket`)
- **Functions**: camelCase (`getVotes`, `collectVotesFromResult`)
- **Constants**: UPPER_SNAKE_CASE (`DEFAULT_PORT`, `RETRY_INTERVAL`)
- **Files**: kebab-case (`vote-handler.js`, `socket-manager.js`)

### Code Organization
```javascript
// 1. External dependencies
const express = require('express');
const { Pool } = require('pg');

// 2. Internal modules
const voteHandler = require('./lib/vote-handler');
const socketManager = require('./lib/socket-manager');

// 3. Configuration
const PORT = process.env.PORT || 4000;
const DB_CONFIG = {
  connectionString: process.env.DATABASE_URL
};

// 4. Application setup
const app = express();
const server = require('http').Server(app);

// 5. Route handlers
app.get('/', handleIndex);
app.use('/api', apiRoutes);

// 6. Server startup
server.listen(PORT, () => {
  console.log(`Server running on port ${PORT}`);
});
```

## Testing

### Test Structure
```
tests/
├── unit/                   # Unit tests
│   ├── vote-aggregation.test.js
│   └── socket-events.test.js
├── integration/            # Integration tests
│   ├── database.test.js
│   └── api-endpoints.test.js
├── e2e/                    # End-to-end tests
│   ├── user-interface.test.js
│   └── real-time-updates.test.js
└── fixtures/               # Test data
    └── sample-votes.json
```

### Unit Testing with Jest

#### Setup Jest (`package.json`)
```json
{
  "scripts": {
    "test": "jest",
    "test:watch": "jest --watch",
    "test:coverage": "jest --coverage"
  },
  "devDependencies": {
    "jest": "^29.0.0",
    "supertest": "^6.0.0",
    "socket.io-client": "^4.7.2"
  },
  "jest": {
    "testEnvironment": "node",
    "collectCoverageFrom": [
      "*.js",
      "!server.js"
    ]
  }
}
```

#### Example Unit Test
```javascript
// tests/unit/vote-aggregation.test.js
const { collectVotesFromResult } = require('../../lib/vote-utils');

describe('Vote Aggregation', () => {
  test('should aggregate votes correctly', () => {
    const mockResult = {
      rows: [
        { vote: 'a', count: '45' },
        { vote: 'b', count: '32' }
      ]
    };

    const result = collectVotesFromResult(mockResult);
    
    expect(result).toEqual({
      a: 45,
      b: 32
    });
  });

  test('should handle empty results', () => {
    const mockResult = { rows: [] };
    const result = collectVotesFromResult(mockResult);
    
    expect(result).toEqual({ a: 0, b: 0 });
  });
});
```

### Integration Testing
```javascript
// tests/integration/api-endpoints.test.js
const request = require('supertest');
const app = require('../../server');

describe('API Endpoints', () => {
  test('GET / should return HTML page', async () => {
    const response = await request(app)
      .get('/')
      .expect(200)
      .expect('Content-Type', /html/);
    
    expect(response.text).toContain('Cats vs Dogs');
  });
});
```

### End-to-End Testing with Playwright
```javascript
// tests/e2e/user-interface.test.js
const { test, expect } = require('@playwright/test');

test('should display real-time vote updates', async ({ page }) => {
  await page.goto('http://localhost:4000');
  
  // Wait for initial load
  await expect(page.locator('.choice.cats .stat')).toBeVisible();
  
  // Check for real-time updates
  const initialCatsPercent = await page.locator('.choice.cats .stat').textContent();
  
  // Simulate vote change (via API or database)
  // ...
  
  // Verify UI updated
  await expect(page.locator('.choice.cats .stat')).not.toHaveText(initialCatsPercent);
});
```

## Performance Optimization

### Database Optimization
```javascript
// Connection pooling configuration
const pool = new Pool({
  connectionString: process.env.DATABASE_URL,
  max: 20,                    // Maximum connections
  idleTimeoutMillis: 30000,   // Close idle connections after 30s
  connectionTimeoutMillis: 2000, // Return error after 2s if no connection
});

// Prepared statements for repeated queries
const getVotesQuery = {
  text: 'SELECT vote, COUNT(id) AS count FROM votes GROUP BY vote',
  values: []
};
```

### Caching Implementation
```javascript
const redis = require('redis');
const client = redis.createClient();

async function getCachedVotes() {
  try {
    const cached = await client.get('vote_counts');
    return cached ? JSON.parse(cached) : null;
  } catch (error) {
    console.error('Cache error:', error);
    return null;
  }
}

async function setCachedVotes(votes, ttl = 5) {
  try {
    await client.setEx('vote_counts', ttl, JSON.stringify(votes));
  } catch (error) {
    console.error('Cache set error:', error);
  }
}
```

### WebSocket Optimization
```javascript
// Rate limiting for WebSocket events
const rateLimitMap = new Map();

io.on('connection', (socket) => {
  const clientId = socket.handshake.address;
  
  // Implement rate limiting
  if (rateLimitMap.has(clientId)) {
    const lastConnection = rateLimitMap.get(clientId);
    if (Date.now() - lastConnection < 1000) {
      socket.disconnect();
      return;
    }
  }
  
  rateLimitMap.set(clientId, Date.now());
  
  // Room-based broadcasting for scaling
  socket.join('results');
  io.to('results').emit('scores', voteData);
});
```

## Debugging

### Debug Configuration
```javascript
// Add to server.js
if (process.env.NODE_ENV === 'development') {
  app.use((req, res, next) => {
    console.log(`${new Date().toISOString()} ${req.method} ${req.url}`);
    next();
  });
}

// Database query debugging
client.query(query, values, (err, result) => {
  if (process.env.DEBUG_SQL) {
    console.log('Query:', query);
    console.log('Values:', values);
    console.log('Result:', result?.rows?.length, 'rows');
  }
  // ... rest of handler
});
```

### Common Debug Scenarios

#### Database Connection Issues
```bash
# Enable PostgreSQL logging
export DEBUG=pg:*
node server.js

# Check connection string
node -e "console.log(process.env.DATABASE_URL)"
```

#### WebSocket Connection Problems
```javascript
// Client-side debugging
socket.on('connect', () => console.log('Connected'));
socket.on('disconnect', (reason) => console.log('Disconnected:', reason));
socket.on('connect_error', (error) => console.log('Connection error:', error));

// Server-side debugging
io.on('connection', (socket) => {
  console.log('Client connected:', socket.id);
  
  socket.on('disconnect', (reason) => {
    console.log('Client disconnected:', socket.id, reason);
  });
});
```

## Contributing Guidelines

### Git Workflow
```bash
# Create feature branch
git checkout -b feature/vote-caching

# Make changes and commit
git add .
git commit -m "feat: add Redis caching for vote aggregation"

# Push and create pull request
git push origin feature/vote-caching
```

### Commit Message Format
```
type(scope): description

[optional body]

[optional footer]
```

Types: `feat`, `fix`, `docs`, `style`, `refactor`, `test`, `chore`

### Pull Request Checklist
- [ ] Code follows style guidelines
- [ ] Tests pass (`npm test`)
- [ ] Documentation updated
- [ ] Performance impact considered
- [ ] Security implications reviewed

## Security Considerations

### Input Validation
```javascript
// Sanitize database queries
const sanitizeInput = (input) => {
  return input.replace(/[<>]/g, '');
};

// Validate environment variables
const validateConfig = () => {
  const required = ['DB_HOST', 'DB_USER', 'DB_PASSWORD'];
  const missing = required.filter(key => !process.env[key]);
  
  if (missing.length > 0) {
    throw new Error(`Missing required environment variables: ${missing.join(', ')}`);
  }
};
```

### Rate Limiting
```javascript
const rateLimit = require('express-rate-limit');

const limiter = rateLimit({
  windowMs: 15 * 60 * 1000, // 15 minutes
  max: 100, // Limit each IP to 100 requests per windowMs
  message: 'Too many requests from this IP'
});

app.use('/', limiter);
```

## Monitoring and Observability

### Application Metrics
```javascript
const prometheus = require('prom-client');

// Custom metrics
const voteCounter = new prometheus.Counter({
  name: 'votes_processed_total',
  help: 'Total number of votes processed'
});

const responseTime = new prometheus.Histogram({
  name: 'http_request_duration_seconds',
  help: 'Duration of HTTP requests in seconds',
  labelNames: ['method', 'route']
});

// Middleware to collect metrics
app.use((req, res, next) => {
  const start = Date.now();
  
  res.on('finish', () => {
    const duration = (Date.now() - start) / 1000;
    responseTime.labels(req.method, req.route?.path || req.url).observe(duration);
  });
  
  next();
});
```

### Health Check Endpoint
```javascript
app.get('/health', async (req, res) => {
  try {
    // Check database connectivity
    await pool.query('SELECT 1');
    
    res.status(200).json({
      status: 'healthy',
      timestamp: new Date().toISOString(),
      version: process.env.npm_package_version
    });
  } catch (error) {
    res.status(503).json({
      status: 'unhealthy',
      error: error.message,
      timestamp: new Date().toISOString()
    });
  }
});
```