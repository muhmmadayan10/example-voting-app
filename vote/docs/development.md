# Development Guide

This guide covers setting up the Vote Service for local development and contributing to the project.

## Development Environment Setup

### Prerequisites

- Python 3.9 or higher
- Redis server
- Git
- Docker (optional but recommended)
- Your favorite code editor (VS Code recommended)

### Initial Setup

1. **Clone the repository**
   ```bash
   git clone https://github.com/muhmmadayan10/example-voting-app.git
   cd example-voting-app/vote
   ```

2. **Create virtual environment**
   ```bash
   python -m venv venv
   source venv/bin/activate  # On Windows: venv\Scripts\activate
   ```

3. **Install dependencies**
   ```bash
   pip install -r requirements.txt
   ```

4. **Install development dependencies**
   ```bash
   pip install -r requirements-dev.txt  # If available
   # Or install manually:
   pip install pytest flask-testing black flake8
   ```

## Local Development

### Running with Docker Compose

The easiest way to run the full stack locally:

```bash
# From the root directory
docker-compose up -d redis
cd vote
python app.py
```

### Running Standalone

1. **Start Redis**
   ```bash
   # Using Docker
   docker run -d -p 6379:6379 redis:alpine
   
   # Or using local Redis installation
   redis-server
   ```

2. **Set environment variables**
   ```bash
   export REDIS_HOST=localhost
   export REDIS_PORT=6379
   export FLASK_ENV=development
   export FLASK_DEBUG=1
   ```

3. **Run the application**
   ```bash
   python app.py
   ```

The application will be available at `http://localhost:5000`

### Development Configuration

For development, you can customize the voting options:

```bash
export OPTION_A="Coffee"
export OPTION_B="Tea"
```

### Hot Reloading

When running in development mode with `FLASK_DEBUG=1`, the application will automatically reload when you make changes to the code.

## Code Structure

```
vote/
├── app.py              # Main Flask application
├── static/             # Static assets (CSS, JS, images)
│   ├── stylesheets/
│   └── images/
├── templates/          # Jinja2 templates
│   └── index.html
├── requirements.txt    # Python dependencies
├── Dockerfile         # Container build instructions
├── docs/              # Documentation (TechDocs)
└── tests/             # Test files (if any)
```

### Key Files

#### `app.py`
The main Flask application containing:
- Route handlers
- Redis connection logic
- Vote processing logic
- Health check implementation

#### `templates/index.html`
The main voting interface template with:
- Voting form
- Real-time updates
- Responsive design

#### `static/`
Static assets including:
- CSS stylesheets
- JavaScript files
- Images and icons

## Development Workflow

### Making Changes

1. **Create a feature branch**
   ```bash
   git checkout -b feature/your-feature-name
   ```

2. **Make your changes**
   - Edit code files
   - Update tests
   - Update documentation

3. **Test your changes**
   ```bash
   # Run the application
   python app.py
   
   # Test manually
   curl http://localhost:5000/
   curl -X POST http://localhost:5000/ -d "vote=a"
   ```

4. **Run tests** (if available)
   ```bash
   python -m pytest
   ```

5. **Code formatting**
   ```bash
   black app.py
   flake8 app.py
   ```

### Testing

#### Manual Testing

1. **Basic functionality**
   - Visit `http://localhost:5000/`
   - Cast votes for both options
   - Verify votes are recorded

2. **Error scenarios**
   - Stop Redis and test error handling
   - Submit invalid form data
   - Test with Redis restart

3. **Performance testing**
   ```bash
   # Simple load test
   for i in {1..100}; do
     curl -X POST http://localhost:5000/ -d "vote=a" &
   done
   wait
   ```

#### Automated Testing

If tests are available:

```bash
# Unit tests
python -m pytest tests/unit/

# Integration tests  
python -m pytest tests/integration/

# All tests
python -m pytest
```

## Debugging

### Common Issues

1. **Redis connection failed**
   - Check if Redis is running: `redis-cli ping`
   - Verify Redis host/port configuration
   - Check network connectivity

2. **Port already in use**
   ```bash
   # Find process using port 5000
   lsof -i :5000
   # Kill the process
   kill -9 <PID>
   ```

3. **Template not found**
   - Verify `templates/` directory exists
   - Check file permissions
   - Ensure Flask can find template files

### Debug Mode

Enable debug mode for detailed error messages:

```bash
export FLASK_DEBUG=1
python app.py
```

### Logging

Add debug logging to your code:

```python
import logging
logging.basicConfig(level=logging.DEBUG)
app.logger.debug("Debug message")
```

## Code Quality

### Style Guidelines

- Follow PEP 8 for Python code
- Use meaningful variable names
- Add docstrings to functions
- Keep functions small and focused

### Code Formatting

```bash
# Format code
black app.py

# Check style
flake8 app.py

# Sort imports
isort app.py
```

### Pre-commit Hooks

Consider setting up pre-commit hooks:

```bash
pip install pre-commit
# Create .pre-commit-config.yaml
pre-commit install
```

## Contributing

### Pull Request Process

1. Fork the repository
2. Create a feature branch
3. Make your changes
4. Write/update tests
5. Update documentation
6. Submit a pull request

### Commit Messages

Use descriptive commit messages:
```
feat: add health check endpoint
fix: handle Redis connection timeout
docs: update API documentation
```

### Code Review

- All changes require code review
- Tests must pass
- Documentation must be updated
- Follow the existing code style

## Docker Development

### Building the Image

```bash
docker build -t vote-service .
```

### Running with Docker

```bash
docker run -d -p 5000:80 \
  -e REDIS_HOST=redis \
  --name vote-service \
  vote-service
```

### Multi-stage Builds

The Dockerfile uses multi-stage builds for optimization:
- Development stage: Includes dev tools
- Production stage: Minimal runtime image

## Environment Variables

### Development

```bash
export FLASK_ENV=development
export FLASK_DEBUG=1
export REDIS_HOST=localhost
export REDIS_PORT=6379
export OPTION_A="Cats"
export OPTION_B="Dogs"
```

### Production

```bash
export FLASK_ENV=production
export REDIS_HOST=redis-service
export REDIS_PORT=6379
# Don't set FLASK_DEBUG in production
```

## Performance Considerations

### Optimization Tips

1. **Redis connection pooling**
   - Use connection pools for better performance
   - Configure appropriate timeouts

2. **Static file serving**
   - Use a web server (nginx) for static files in production
   - Enable gzip compression

3. **Caching**
   - Cache template compilation
   - Use Redis for session caching

### Monitoring

Monitor these metrics during development:
- Response time
- Memory usage
- Redis connection count
- Error rates

## Troubleshooting

### Common Development Issues

1. **Redis connection timeout**
   ```python
   # Add connection timeout
   redis_client = redis.Redis(
       host=redis_host, 
       port=redis_port,
       socket_connect_timeout=5,
       socket_timeout=5
   )
   ```

2. **Template rendering issues**
   ```python
   # Debug template variables
   print(f"Template variables: {locals()}")
   ```

3. **Static files not loading**
   - Check static file paths
   - Verify Flask static folder configuration
   - Test with absolute URLs

## Getting Help

- **Documentation**: This TechDocs site
- **Issues**: [GitHub Issues](https://github.com/muhmmadayan10/example-voting-app/issues)
- **Team**: Contact the Frontend Team
- **Slack**: #voting-tech-dev channel