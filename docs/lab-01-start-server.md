# Lab 01 - Start the Web Server

## Objective

Get the ResumeCrunch application up and running using Docker Compose.

## Background

The ResumeCrunch application is a web-based resume parsing service that uses:
- **Flask** - Python web framework
- **Nginx** - Web server and reverse proxy
- **Redis** - Caching layer
- **Docker** - Containerization

## Steps

### 1. Clone the Repository (if not already done)

```bash
git clone <repository-url>
cd ResumeCrunch-Lab
```

### 2. Start the Application

Use Docker Compose to start all services:

```bash
docker-compose up -d
```

This command will:
- Build the application containers
- Start Nginx, Flask app, and Redis
- Run everything in the background (`-d` flag)

### 3. Verify Services are Running

Check that all containers are up:

```bash
docker-compose ps
```

You should see three containers running:
- `resumecrunch-nginx`
- `resumecrunch-app`
- `resumecrunch-redis`

### 4. Access the Application

Open your browser and navigate to:

```
http://localhost:8080
```

You should see the ResumeCrunch home page.

### 5. Test Basic Functionality

Try uploading a sample resume file to ensure the application is working correctly.

## Verification

- [ ] All Docker containers are running
- [ ] Application is accessible at http://localhost:8080
- [ ] You can view the home page
- [ ] Basic functionality works (file upload)

## Troubleshooting

### Containers Won't Start

```bash
# Check logs
docker-compose logs

# Restart services
docker-compose down
docker-compose up -d
```

### Port Already in Use

If port 8080 is already in use, you can modify the `docker-compose.yml` file to use a different port.

## Next Steps

Once the application is running, proceed to [Lab 02 - Interrogate the Web Server](lab-02-interrogate-server.md) to begin reconnaissance.

---

[← Back to Lab Overview](lab-overview.md) | [Next: Interrogate Server →](lab-02-interrogate-server.md)
