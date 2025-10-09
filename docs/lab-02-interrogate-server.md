# Lab 02 - Interrogate the Web Server

## Objective

Use `dirb` and `curl` to gather information about the web server, discover directories, analyze HTTP headers and cookies, and identify the technology stack.

## Background

Reconnaissance is the first phase of any penetration test. By interrogating the server with automated tools and manual requests, we can identify:
- Available directories and endpoints
- Server software and versions
- Response headers and security configurations
- Session management mechanisms
- Potential misconfigurations

We'll use two key tools:
- **dirb**: Directory and file brute-forcing tool
- **curl**: Command-line HTTP client for detailed request/response analysis

## Prerequisites

### Install dirb (if needed)

**macOS:**
```bash
brew install dirb
```

**Linux:**
```bash
sudo apt install dirb
```

**Windows:**
This lab has not been tested on Windows machines. You'll need to find a Windows version of DIRB, or alternatively, you can use OWASP ZAP (Zed Attack Proxy) which works well on Windows.

**Documentation:**
For more information about DIRB, see the official documentation: https://www.kali.org/tools/dirb/

## Steps

### 1. Initial Directory Discovery with dirb

Start by running `dirb` with the common wordlist to discover available directories:

**Note:** You might want to cd to the lab-files directory first, for easy access to the wordlists there.

```bash
dirb http://localhost:8080 common.txt
```

**What to look for:**
- Watch for HTTP 200 and 301 responses
- Note any directories discovered

**Expected discoveries:**
```
==> DIRECTORY: http://localhost:8080/static/
==> DIRECTORY: http://localhost:8080/static/css/                                                                  
==> DIRECTORY: http://localhost:8080/static/images/                                                               
==> DIRECTORY: http://localhost:8080/static/js/ 
```

### 2. Examine HTTP Response Headers with curl

Now use `curl` to examine the server's HTTP response headers in detail:

```bash
# Get headers only (-I flag, include only response headers in output)
curl -I http://localhost:8080
```


**Look for key information:**

**Server identification:**
```
Server: nginx/1.19.10
```
- Note the server type (Nginx)
- Note the specific version number
- This could help identify version-specific vulnerabilities

**Additional headers to observe:**

*Note: These headers are optional and may not appear on every endpoint. Their presence (or absence) reveals important security information.*

- `X-Powered-By`: Reveals the application framework or language (e.g., Express, PHP, ASP.NET). Its presence is an information disclosure risk.
- `Content-Type`: Specifies the MIME type of the response content (e.g., text/html, application/json).
- `Content-Length`: Indicates the size of the response body in bytes.
- Security headers (often missing, but should be present):
  - `X-Frame-Options`: Prevents clickjacking by controlling whether the page can be embedded in frames.
  - `X-Content-Type-Options`: Prevents MIME-sniffing attacks by forcing browsers to respect the declared Content-Type.
  - `Strict-Transport-Security` (HSTS): Forces browsers to use HTTPS for all future requests to the domain.
  - `Content-Security-Policy` (CSP): Defines approved sources of content, mitigating XSS and injection attacks.

### 3. Analyze Cookies Headers Being Set

Make a request and examine cookies being set:

```bash
# View full response with headers (-v flag)
curl -I http://localhost:8080
```

**Look for:**
- **Session cookies**: Names like `session`, `SESSIONID`, `PHPSESSID`, `connect.sid`
- **Cookie attributes**: 
  - `HttpOnly`: Prevents JavaScript access
  - `Secure`: Only sent over HTTPS
  - `SameSite`: CSRF protection
  - `Domain`: Cookie scope
  - `Path`: Cookie path restrictions
  - `Expires/Max-Age`: Cookie lifetime

**Notice our web server is setting our cookie like this:**
```
Set-Cookie: session_id=d6da6196-334a-4e0b-b795-635488abc8af; Expires=Wed, 08 Oct 2025 05:48:53 GMT; Max-Age=3600; Path=/
```

### 4. Test for Cookie Manipulation

Since we've identified a session cookie, let's test if we can manipulate it:

```bash
# Make a request with a modified session cookie
curl -b "session_id=my_modified_cookie" http://localhost:8080/
```

**Questions to consider:**
- Does the application validate session cookies properly?
- Can you create arbitrary session values?
- Are session tokens predictable?
- What happens with an invalid session cookie?

**Note:** A couple other things you can try for reconnissance: Test different HTTP methods, and look for common information disclosure files.


## Key Observations

### Nginx Web Server
You should have identified:
- **Server**: Nginx (from the `Server:` header)
- **Version**: Specific version number disclosed
- **Why this matters**: Version disclosure helps attackers find version-specific vulnerabilities

### Session Cookie Analysis
The session cookie is critical:
- **Created by the server**: Shows session-based authentication is used
- **Can you issue new ones?**: This is a key question for later exploitation
- **Format and encoding**: Understanding this helps with session manipulation attacks

### Directory Structure
The `/static/` directory is particularly interesting:
- Contains static assets (CSS, JavaScript, images)
- Often has different configuration than other paths
- May be vulnerable to path traversal (test in next lab!)

## Information to Document

| Item | Finding | Potential Impact |
|------|---------|------------------|
| Web Server | nginx/1.19.10 | Version-specific vulnerabilities |
| Directories | /static/, /upload/, /results/ | Attack surface mapping |
| Session Management | Cookie-based sessions | Session manipulation potential |
| Security Headers | Missing X-Frame-Options, CSP, etc. | Clickjacking, XSS risks |
| Cookie Security | No Secure flag | Session hijacking over HTTP |

## Verification

- [ ] Successfully ran dirb with common.txt wordlist
- [ ] Discovered the /static/ directory (and others)
- [ ] Used curl to examine HTTP response headers
- [ ] Identified Nginx as the web server
- [ ] Found the server version number
- [ ] Analyzed session cookies being created
- [ ] Checked for security headers (we are missing all of them)

## Questions to Answer

1. **What web server is being used?** (Nginx)
2. **Is the server version disclosed?** (Yes, check the `Server:` header)
3. **What directories did dirb discover?** (/static/, and some various other endpoints )
4. **What is the session cookie name?** (Typically `session`, in our case `session_id`)
5. **Can you issue new session cookies?** (To be tested)
6. **What security headers are missing?** (Most of them!)
7. **Is there anything interesting about the /static/ directory?** (Keep this in mind for Lab 03!)

## Next Steps

You've completed reconnaissance and identified:
- The web server (Nginx)
- Key directories (especially `/static/`)
- Session management mechanism
- Missing security controls

Now proceed to [Lab 03 - Path Traversal Discovery](lab-03-path-traversal.md) to test for Nginx misconfigurations, specifically the off-by-one slash vulnerability on the `/static/` directory.

---

[← Previous: Start Server](lab-01-start-server.md) | [Next: Path Traversal Testing →](lab-03-path-traversal.md)
