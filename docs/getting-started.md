# Getting Started

Welcome to ResumeCrunch-Lab! This guide will help you set up your development environment and get the application running locally.

## Prerequisites

Before you begin, ensure you have the following software installed on your system:

### Required Software

- **Docker Desktop** (recommended) or Docker Engine
  - [Download Docker Desktop](https://www.docker.com/products/docker-desktop/)
  - Docker Desktop includes both Docker Engine and Docker Compose
  - Minimum version: Docker 20.10 or higher

- **Docker Compose**
  - Included with Docker Desktop
  - If using Docker Engine standalone, install Docker Compose separately
  - Minimum version: Docker Compose 2.0 or higher

### Verify Installation

After installing Docker Desktop, verify your installation by running:

```bash
docker --version
docker-compose --version
```

You should see version information for both commands.

## Quick Start

### 1. Clone the Repository

```bash
git clone https://github.com/gar-rock/ResumeCrunch-Lab.git
cd ResumeCrunch-Lab
```

### 2. Start the Application

Use Docker Compose to build and start all services:

```bash
docker-compose up -d
```

The `-d` flag runs the containers in detached mode (in the background).

### 3. Access the Application

Once the containers are running, you can access the application at:

- **Application**: http://localhost:8080 

### 4. View Logs

To view the application logs:

```bash
docker-compose logs -f
```

Press `Ctrl+C` to stop following the logs.

## Common Commands

### Stop the Application

```bash
docker-compose down
```

### Rebuild Containers

If you've made changes to the Dockerfile or dependencies:

```bash
docker-compose up -d --build
```

### View Running Containers

```bash
docker-compose ps
```

### Execute Commands in a Container

```bash
docker-compose exec <service-name> <command>
```

### Remove All Containers and Volumes

```bash
docker-compose down -v
```

**Warning**: This will delete all data stored in Docker volumes.

## Troubleshooting

### Docker Desktop Not Running

If you see connection errors, ensure Docker Desktop is running:
- On macOS: Check the Docker icon in the menu bar
- On Windows: Check the Docker icon in the system tray
- On Linux: Ensure the Docker daemon is running with `sudo systemctl status docker`

### Port Already in Use

If you encounter a "port already in use" error:

1. Check what's using the port:
   ```bash
   lsof -i :<port-number>
   ```

2. Either stop the conflicting service or modify the port mapping in `docker-compose.yml`

### Permission Denied Errors

On Linux, you may need to add your user to the docker group:

```bash
sudo usermod -aG docker $USER
```

Log out and back in for the changes to take effect.

### Container Fails to Start

Check the logs for specific error messages:

```bash
docker-compose logs <service-name>
```

## Next Steps

Now that you have the application running:

- Review the [Learning Objectives](learning-objectives.md) to understand the project goals
- Check out [Security Best Practices](security-best-practices.md) to learn about secure development
- Explore the [Vulnerabilities](vulnerabilities.md) documentation to understand security testing

## Getting Help

If you encounter any issues not covered in this guide:

1. Check existing [GitHub Issues](https://github.com/gar-rock/ResumeCrunch-Lab/issues)
2. Review the Docker logs for error messages
3. Create a new issue with detailed information about your problem

Happy coding! 🚀
