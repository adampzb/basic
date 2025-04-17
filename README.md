# Deploying OpenHands on a Custom Server

This guide provides detailed instructions for deploying OpenHands on your own server infrastructure. It covers both Docker-based deployment (recommended for production) and building from source (for development or customization).

## Table of Contents

- [System Requirements](#system-requirements)
- [Docker-based Deployment](#docker-based-deployment)
  - [Basic Deployment](#basic-deployment)
  - [Hardened Deployment for Public Networks](#hardened-deployment-for-public-networks)
  - [Connecting to Your Local Filesystem](#connecting-to-your-local-filesystem)
  - [Environment Variables](#environment-variables)
- [Building from Source](#building-from-source)
  - [Prerequisites](#prerequisites)
  - [Installation Steps](#installation-steps)
  - [Configuration](#configuration)
  - [Running the Application](#running-the-application)
- [LLM Configuration](#llm-configuration)
- [Advanced Deployment Options](#advanced-deployment-options)
  - [Headless Mode](#headless-mode)
  - [CLI Mode](#cli-mode)
  - [GitHub Action Integration](#github-action-integration)
- [Troubleshooting](#troubleshooting)

## System Requirements

### Minimum Requirements
- CPU: 4 cores
- RAM: 8GB
- Storage: 20GB free space
- Operating System: Linux (Ubuntu 22.04+), macOS, or Windows with WSL2
- Docker installed and running

### Recommended Requirements
- CPU: 8+ cores
- RAM: 16GB+
- Storage: 50GB+ SSD
- Operating System: Linux (Ubuntu 22.04+)
- Docker installed and running
- Fast internet connection for LLM API calls

## Docker-based Deployment

### Basic Deployment

The simplest way to deploy OpenHands is using Docker. This method requires minimal setup and is recommended for most users.

1. Make sure Docker is installed and running on your server:
   ```bash
   docker --version
   ```

2. Create a directory for OpenHands state:
   ```bash
   mkdir -p ~/.openhands-state
   ```

3. Pull and run the OpenHands container:
   ```bash
   docker pull docker.all-hands.dev/all-hands-ai/runtime:0.33-nikolaik

   docker run -it --rm --pull=always \
       -e SANDBOX_RUNTIME_CONTAINER_IMAGE=docker.all-hands.dev/all-hands-ai/runtime:0.33-nikolaik \
       -e LOG_ALL_EVENTS=true \
       -v /var/run/docker.sock:/var/run/docker.sock \
       -v ~/.openhands-state:/.openhands-state \
       -p 3000:3000 \
       --add-host host.docker.internal:host-gateway \
       --name openhands-app \
       docker.all-hands.dev/all-hands-ai/openhands:0.33
   ```

4. Access OpenHands at `http://your-server-ip:3000`

5. When you open the application, you'll be asked to choose an LLM provider and add an API key. [Anthropic's Claude 3.5 Sonnet](https://www.anthropic.com/api) (`anthropic/claude-3-5-sonnet-20241022`) is recommended, but there are [many options](https://docs.all-hands.dev/modules/usage/llms).

### Hardened Deployment for Public Networks

If you're deploying on a public network, you should take additional security measures:

```bash
docker run -it --rm --pull=always \
    -e SANDBOX_RUNTIME_CONTAINER_IMAGE=docker.all-hands.dev/all-hands-ai/runtime:0.33-nikolaik \
    -e LOG_ALL_EVENTS=true \
    -v /var/run/docker.sock:/var/run/docker.sock \
    -v ~/.openhands-state:/.openhands-state \
    -p 127.0.0.1:3000:3000 \
    --add-host host.docker.internal:host-gateway \
    --name openhands-app \
    docker.all-hands.dev/all-hands-ai/openhands:0.33
```

This binds the application only to localhost (127.0.0.1), preventing direct access from the internet. You should then set up a reverse proxy (like Nginx or Apache) with proper authentication and HTTPS to securely expose the application.

Example Nginx configuration:

```nginx
server {
    listen 443 ssl;
    server_name openhands.yourdomain.com;

    ssl_certificate /path/to/cert.pem;
    ssl_certificate_key /path/to/key.pem;

    location / {
        proxy_pass http://127.0.0.1:3000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        # WebSocket support
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
    }

    # Basic authentication
    auth_basic "Restricted Access";
    auth_basic_user_file /path/to/.htpasswd;
}
```

### Connecting to Your Local Filesystem

To connect OpenHands to your local filesystem:

```bash
docker run -it --rm --pull=always \
    -e SANDBOX_RUNTIME_CONTAINER_IMAGE=docker.all-hands.dev/all-hands-ai/runtime:0.33-nikolaik \
    -e LOG_ALL_EVENTS=true \
    -e WORKSPACE_MOUNT_PATH=/path/to/your/projects \
    -v /var/run/docker.sock:/var/run/docker.sock \
    -v ~/.openhands-state:/.openhands-state \
    -v /path/to/your/projects:/opt/workspace_base \
    -p 3000:3000 \
    --add-host host.docker.internal:host-gateway \
    --name openhands-app \
    docker.all-hands.dev/all-hands-ai/openhands:0.33
```

Replace `/path/to/your/projects` with the absolute path to your project directory.

### Environment Variables

You can customize the OpenHands deployment using these environment variables:

| Variable | Description | Default |
|----------|-------------|---------|
| `SANDBOX_RUNTIME_CONTAINER_IMAGE` | The Docker image used for the runtime environment | `docker.all-hands.dev/all-hands-ai/runtime:0.33-nikolaik` |
| `LOG_ALL_EVENTS` | Enable detailed event logging | `false` |
| `WORKSPACE_MOUNT_PATH` | Path to mount as the workspace | `/opt/workspace_base` |
| `DEBUG` | Enable debug mode for LLM logging | `0` |
| `SANDBOX_USER_ID` | User ID for the sandbox (advanced) | - |

## Building from Source

Building from source is recommended if you want to customize OpenHands or contribute to its development.

### Prerequisites

- Linux, macOS, or Windows with WSL2 (Ubuntu 22.04+ recommended)
- Python 3.12
- Node.js 22.x or later
- Poetry 1.8 or later
- Docker
- Git

#### Installing Prerequisites on Ubuntu

```bash
# Update package lists
sudo apt update

# Install basic build tools
sudo apt install -y build-essential netcat

# Install Python 3.12
sudo add-apt-repository ppa:deadsnakes/ppa
sudo apt update
sudo apt install -y python3.12 python3.12-venv python3.12-dev

# Install Node.js 22.x
curl -fsSL https://deb.nodesource.com/setup_22.x | sudo -E bash -
sudo apt install -y nodejs

# Install Poetry
curl -sSL https://install.python-poetry.org | python3.12 -

# Add Poetry to PATH (add to your .bashrc or .zshrc)
echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.bashrc
source ~/.bashrc

# Install Docker
sudo apt install -y docker.io
sudo systemctl enable --now docker
sudo usermod -aG docker $USER
# Log out and log back in for group changes to take effect
```

### Installation Steps

1. Clone the repository:
   ```bash
   git clone https://github.com/All-Hands-AI/OpenHands.git
   cd OpenHands
   ```

2. Build the project:
   ```bash
   make build
   ```
   This command will:
   - Check for all required dependencies
   - Install Python dependencies using Poetry
   - Install frontend dependencies using npm
   - Install pre-commit hooks
   - Build the frontend

3. Set up the configuration:
   ```bash
   make setup-config
   ```
   This will prompt you for:
   - Workspace directory (absolute path)
   - LLM model name
   - LLM API key
   - LLM base URL (optional, for local LLMs)

### Configuration

The configuration is stored in `config.toml` in the project root. You can edit this file directly if needed:

```toml
[core]
workspace_base = "/path/to/your/workspace"

[llm]
model = "anthropic/claude-3-5-sonnet-20241022"
api_key = "your-api-key-here"
# base_url = "http://localhost:5001/v1/" # Uncomment for local LLMs
```

### Running the Application

1. Start the full application (backend and frontend):
   ```bash
   make run
   ```

2. Or start the components individually:
   ```bash
   # Start only the backend
   make start-backend

   # Start only the frontend
   make start-frontend
   ```

3. Access OpenHands at `http://your-server-ip:3000`

## LLM Configuration

OpenHands supports various LLM providers through the [litellm](https://docs.litellm.ai) library. Here are some recommended models:

1. **Anthropic Claude 3.5 Sonnet** (Recommended)
   - Model: `anthropic/claude-3-5-sonnet-20241022`
   - [Get API key from Anthropic](https://www.anthropic.com/api)

2. **OpenAI GPT-4o**
   - Model: `gpt-4o`
   - [Get API key from OpenAI](https://platform.openai.com/api-keys)

3. **Google Gemini 1.5 Pro**
   - Model: `gemini-1.5-pro`
   - [Get API key from Google AI Studio](https://makersuite.google.com/app/apikey)

4. **Local Models**
   - You can use local models through services like [Ollama](https://ollama.ai/) or [LM Studio](https://lmstudio.ai/)
   - Set the base_url in config.toml to your local endpoint

## Advanced Deployment Options

### Headless Mode

OpenHands can run in headless mode for scripted operations:

```bash
# Using Docker
docker run -it --rm --pull=always \
    -e SANDBOX_RUNTIME_CONTAINER_IMAGE=docker.all-hands.dev/all-hands-ai/runtime:0.33-nikolaik \
    -e LOG_ALL_EVENTS=true \
    -v /var/run/docker.sock:/var/run/docker.sock \
    -v ~/.openhands-state:/.openhands-state \
    -v /path/to/your/projects:/opt/workspace_base \
    --add-host host.docker.internal:host-gateway \
    --name openhands-app \
    docker.all-hands.dev/all-hands-ai/openhands:0.33 \
    openhands headless "Create a simple Python web server"

# From source
poetry run openhands headless "Create a simple Python web server"
```

### CLI Mode

OpenHands provides a CLI mode for interactive terminal usage:

```bash
# Using Docker
docker run -it --rm --pull=always \
    -e SANDBOX_RUNTIME_CONTAINER_IMAGE=docker.all-hands.dev/all-hands-ai/runtime:0.33-nikolaik \
    -e LOG_ALL_EVENTS=true \
    -v /var/run/docker.sock:/var/run/docker.sock \
    -v ~/.openhands-state:/.openhands-state \
    -v /path/to/your/projects:/opt/workspace_base \
    --add-host host.docker.internal:host-gateway \
    --name openhands-app \
    docker.all-hands.dev/all-hands-ai/openhands:0.33 \
    openhands cli

# From source
poetry run openhands cli
```

### GitHub Action Integration

You can integrate OpenHands with GitHub Actions to automatically process issues. Create a file at `.github/workflows/openhands.yml` in your repository:

```yaml
name: OpenHands AI Assistant

on:
  issues:
    types: [opened, edited, labeled]

jobs:
  openhands:
    if: contains(github.event.issue.labels.*.name, 'openhands')
    runs-on: ubuntu-latest
    permissions:
      contents: write
      issues: write
      pull-requests: write
    steps:
      - uses: actions/checkout@v4
      - name: Run OpenHands
        uses: All-Hands-AI/openhands-action@v1
        with:
          github-token: ${{ secrets.GITHUB_TOKEN }}
          openai-api-key: ${{ secrets.OPENAI_API_KEY }}
          # Or use Anthropic
          # anthropic-api-key: ${{ secrets.ANTHROPIC_API_KEY }}
```

## Troubleshooting

### Common Issues

1. **Docker Socket Permission Denied**
   ```
   Error: Got permission denied while trying to connect to the Docker daemon socket
   ```
   Solution:
   ```bash
   sudo chmod 666 /var/run/docker.sock
   # Or add your user to the docker group
   sudo usermod -aG docker $USER
   # Then log out and log back in
   ```

2. **Port Already in Use**
   ```
   Error: Address already in use
   ```
   Solution:
   ```bash
   # Find the process using port 3000
   sudo lsof -i :3000
   # Kill the process
   sudo kill <PID>
   # Or change the port in the docker run command
   ```

3. **LLM API Key Issues**
   ```
   Error: Authentication error with LLM provider
   ```
   Solution:
   - Verify your API key is correct
   - Check if your account has billing set up with the LLM provider
   - Ensure you have sufficient credits/quota

4. **Memory Issues**
   ```
   Error: Container exited with code 137
   ```
   Solution:
   - Increase Docker memory limit
   - Reduce other memory-intensive processes on the server

### Logs and Debugging

To enable debug mode for LLM logging:

```bash
# Using Docker
docker run -it --rm --pull=always \
    -e SANDBOX_RUNTIME_CONTAINER_IMAGE=docker.all-hands.dev/all-hands-ai/runtime:0.33-nikolaik \
    -e LOG_ALL_EVENTS=true \
    -e DEBUG=1 \
    -v /var/run/docker.sock:/var/run/docker.sock \
    -v ~/.openhands-state:/.openhands-state \
    -p 3000:3000 \
    --add-host host.docker.internal:host-gateway \
    --name openhands-app \
    docker.all-hands.dev/all-hands-ai/openhands:0.33
```

Logs will be stored in the `logs/llm/CURRENT_DATE` directory within the container. You can access them by:

```bash
docker exec -it openhands-app ls -la /app/logs/llm/
```

For more detailed troubleshooting, refer to the [official documentation](https://docs.all-hands.dev/modules/usage/troubleshooting).