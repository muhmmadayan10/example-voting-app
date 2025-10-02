# Vote Service

The Vote Service is the frontend interface where users can cast their votes in the VotingTech voting application.

## Overview

This service provides a simple web interface built with Python Flask that allows users to:
- Cast votes for option A or B
- View current voting status  
- Interact with the voting system in real-time

## Architecture

The Vote service is a Python Flask application that:
- Serves HTML pages for the voting interface
- Connects to Redis for session management
- Publishes vote data to message queue for processing
- Provides a clean, responsive web interface

## Technology Stack

- **Language**: Python 3.9+
- **Framework**: Flask
- **Template Engine**: Jinja2
- **Session Store**: Redis
- **Frontend**: HTML5, CSS3, JavaScript
- **Containerization**: Docker

## Quick Start

### Prerequisites
- Python 3.9 or higher
- Redis server running
- Docker (optional but recommended)

### Local Development

1. **Clone the repository**
   ```bash
   git clone https://github.com/muhmmadayan10/example-voting-app.git
   cd example-voting-app/vote
   ```

2. **Install dependencies**
   ```bash
   pip install -r requirements.txt
   ```

3. **Set environment variables**
   ```bash
   export REDIS_HOST=localhost
   export REDIS_PORT=6379
   ```

4. **Run the application**
   ```bash
   python app.py
   ```

The service will be available at `http://localhost:5000`

### Docker Development

```bash
docker build -t vote-service .
docker run -d -p 5000:80 \
  -e REDIS_HOST=redis \
  vote-service
```

## Configuration

### Environment Variables

| Variable | Description | Default | Required |
|----------|-------------|---------|----------|
| `REDIS_HOST` | Redis server hostname | localhost | Yes |
| `REDIS_PORT` | Redis server port | 6379 | No |
| `OPTION_A` | First voting option | Cats | No |
| `OPTION_B` | Second voting option | Dogs | No |

### Application Configuration

The application serves static files from the `static/` directory and uses Jinja2 templates from the `templates/` directory.

## Features

- **Real-time voting**: Immediate feedback when votes are cast
- **Session management**: Prevents duplicate voting per session
- **Responsive design**: Works on desktop and mobile devices
- **Error handling**: Graceful handling of Redis connection issues
- **Health checks**: Built-in health check endpoint

## Related Services

This service is part of the VotingTech microservices ecosystem:

- **[Worker Service](../worker)**: Processes votes asynchronously
- **[Result Service](../result)**: Displays real-time voting results  
- **[Seed Data Service](../seed-data)**: Initializes the database
- **Redis**: Session storage and message queue
- **PostgreSQL**: Persistent vote storage

## Getting Help

- **Issues**: [GitHub Issues](https://github.com/muhmmadayan10/example-voting-app/issues)
- **Documentation**: Available in Backstage TechDocs
- **Team**: Contact the Frontend Team via Slack