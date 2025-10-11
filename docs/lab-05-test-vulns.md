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

**CVE-2021-33026** affects Flask-Caching versions through 1.10.1, allows remote code execution through cache poisoning by exploiting unsafe pickle deserialization.

## Prerequisites

### Install pip-audit or pip-audit-extra

*If pip is not installed, please install first.*
*at this point you may have to use a python virtual environment*
```bash
python3 -m venv venv
source venv/bin/activate
```

```bash
# Install pip-audit
pip install pip-audit

# Or install the enhanced version with extra features
pip install pip-audit-extra
```

## Steps

### 1. Deep Analysis of app.py - Redis Integration

Open and thoroughly analyze the Flask application:

```bash
cat app.py
```

**Search for Redis-specific code:**

```python
app.config["CACHE_REDIS_URL"] = f"redis://:{rpass}@{rserver}:6379/0"
```

Using the data we retrieved from last lab we have our redis connection URL as:

```
redis://hCQr7gvbyRRN79Ugvc9Lssq6@resumecrunch-redis:6379/0

```

### 2. Identify which endpoints have redis caching

See the main path in our source code for home or /

```python
@app.route("/")
def home():
    # Get or create session ID from cookie
    session_id = request.cookies.get('session_id')
    
    if not session_id:
        # New session - create ID and initialize data
        session_id = str(uuid4())
        session_data = {
            'session_id': session_id,
            'created_at': datetime.datetime.now().isoformat(),
            'visit_count': 1,
            'last_visit': datetime.datetime.now().isoformat()
        }
        cache.set(f'session:{session_id}', session_data, timeout=3600)  # 1 hour timeout
    else:
        # Existing session - retrieve and update data
        session_data = cache.get(f'session:{session_id}')
        print()
        print(session_data ) 
        print()      
        if session_data is None:
            # Session expired or doesn't exist, create new one
            session_data = {
                'session_id': session_id,
                'created_at': datetime.datetime.now().isoformat(),
                'visit_count': 1,
                'last_visit': datetime.datetime.now().isoformat()
            }
        else:
            # Update existing session
            session_data['visit_count'] = session_data.get('visit_count', 0) + 1
            session_data['last_visit'] = datetime.datetime.now().isoformat()
        
        # Save updated session data back to cache
        cache.set(f'session:{session_id}', session_data, timeout=3600)
    
    # Create response and set cookie
    response = make_response(render_template("index.html"))
    response.set_cookie('session_id', session_id, max_age=3600)  # 1 hour cookie
    
    return response
```

It appears that this is the main page that is interacting with the session cache and issuing new session cookies. 

If we run the following command we will see our new session cookie that the web server is telling our client to set. 

```
curl -I http://127.0.0.1:8080
```
```
curl: (3) URL rejected: No host part in the URL
HTTP/1.1 200 OK
Server: nginx/1.19.10
Date: Sat, 11 Oct 2025 00:57:57 GMT
Content-Type: text/html; charset=utf-8
Content-Length: 28796
Connection: keep-alive
Set-Cookie: session_id=8d9d6ba2-fe9e-4782-aff6-ce92f10b865f; Expires=Sat, 11 Oct 2025 01:57:57 GMT; Max-Age=3600; Path=/
```

### 3. Verify Resume Upload Folder Accessibility

Test if you can access the resumes folder and the files you uploaded in Lab 01:

```bash
# Try to list the resumes directory
curl -v http://localhost:8080/static../resumes/
```
We see a 403 forbidden, due to directory listing not being allowed

```
# Try to access a specific resume you uploaded earlier
curl http://localhost:8080/static../resumes/my_resume.pdf
```
We can see that we are able to access a specific file in this directory, and verify that we uploaded our resume earlier.
*we will use this technique later*


### 4. Run pip-audit-extra on requirements.txt

Scan all Python dependencies for known vulnerabilities:

```bash
# Basic scan
pip-audit -r requirements.txt
```
*if you do not have requirements.txt downloaded please run this*
```
curl http://localhost:8080/static../requirements.py -o requirements.txt
```

Expected output from pip-audit
```
pip-audit -r requirements.txt.txt
Found 3 known vulnerabilities in 2 packages
Name          Version ID                  Fix Versions
------------- ------- ------------------- ------------
flask-caching 1.10.1  GHSA-656c-6cxf-hvcv
pypdf2        1.24    PYSEC-2022-194      1.27.5
pypdf2        1.24    GHSA-jrm6-h9cq-8gqw 1.27.9
```
We can see that flask-caching has a known vulnerability that GitHub has an ID for, let's look this up:
[GitHub Security Advisory GHSA-656c-6cxf-hvcv](https://github.com/advisories/GHSA-656c-6cxf-hvcv)

Reading this GitHub alert we see the following explanation:
```
Flask-Cache adds easy cache support to Flask. The Flask-Caching extension through 1.10.1 for Flask 
relies on Pickle for serialization, which may lead to remote code execution or local privilege escalation. 
If an attacker gains access to cache storage (e.g., filesystem, Memcached, Redis, etc.),
 they can construct a crafted payload, poison the cache, and execute Python code.
```

Well, we do have cache access now with our redis credentials, now we just need a way to uplaod a payload.

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

[← Previous: Extract Docker Files](lab-04-extract-docker.md) | [Next: Redis Exploitation →](lab-06-redis-exploit.md)
