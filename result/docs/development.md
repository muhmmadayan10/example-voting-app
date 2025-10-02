# Development Guide

## Setting up the Result Service

### Prerequisites

- Node.js 16+ 
- PostgreSQL database
- Environment variables configured

### Installation

```bash
npm install
```

### Configuration

Set the following environment variables:

```bash
POSTGRES_HOST=localhost
POSTGRES_DB=votes
POSTGRES_USER=postgres
POSTGRES_PASSWORD=postgres
```

### Running in Development

```bash
npm start
```

The service will be available at http://localhost:4000

### Development Workflow

1. Make changes to the source code
2. The application will automatically restart
3. Check the logs for any errors
4. Test the voting results display

### Database Schema

The service expects a `votes` table with the following structure:

```sql
CREATE TABLE votes (
    id SERIAL PRIMARY KEY,
    vote VARCHAR(255) NOT NULL,
    timestamp TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```