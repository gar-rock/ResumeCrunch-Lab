# Lab 02 - Interrogate the Web Server

## Objective

Gather information about the web server, its configuration, and potential attack vectors.

## Background

Reconnaissance is the first phase of any penetration test. By interrogating the server, we can identify:
- Server software and versions
- Enabled HTTP methods
- Response headers
- Technology stack
- Potential misconfigurations

## Steps

### 1. Check Server Headers

Use `curl` to examine the HTTP response headers:

```bash
curl -I http://localhost:8080
```

**Look for:**
- Server version (e.g., `nginx/1.21.0`)
- Additional headers like `X-Powered-By`
- Security headers (or lack thereof)

### 2. Test Different HTTP Methods

Check which HTTP methods are allowed:

```bash
curl -X OPTIONS http://localhost:8080 -i
```

Try potentially dangerous methods:

```bash
# Test PUT
curl -X PUT http://localhost:8080/test.txt -d "test data" -i

# Test DELETE
curl -X DELETE http://localhost:8080/test.txt -i

# Test TRACE
curl -X TRACE http://localhost:8080 -i
```

### 3. Check for Common Files

Look for common configuration and information disclosure files:

```bash
# Robots.txt
curl http://localhost:8080/robots.txt

# Sitemap
curl http://localhost:8080/sitemap.xml

# .git directory
curl http://localhost:8080/.git/

# Common admin panels
curl http://localhost:8080/admin
curl http://localhost:8080/admin/
```

### 4. Analyze the Application

Browse the application manually and note:
- Available endpoints (e.g., `/upload`, `/results`, `/api`)
- Form fields and their validation
- Client-side JavaScript files
- Cookie structure
- URL parameters

### 5. Check for Error Messages

Try to trigger error messages that might reveal information:

```bash
# Non-existent page
curl http://localhost:8080/nonexistent

# Malformed requests
curl http://localhost:8080/%00

# SQL injection attempts (just to see responses)
curl "http://localhost:8080/search?q='"
```

### 6. Examine JavaScript Files

If the application uses JavaScript, download and review the source:

```bash
# Find JavaScript files first
curl -s http://localhost:8080 | grep -o 'src="[^"]*\.js"' | cut -d'"' -f2

# Download and examine
curl http://localhost:8080/static/js/main.js
```

## Information to Document

Create a notes file with your findings:

| Item | Finding | Potential Impact |
|------|---------|------------------|
| Web Server | nginx/x.x.x | Version-specific vulnerabilities |
| Application Server | Flask/Python | Framework vulnerabilities |
| Enabled Methods | GET, POST, OPTIONS | Potential for upload/delete attacks |
| Security Headers | Missing X-Frame-Options | Clickjacking possible |
| Error Messages | Verbose stack traces | Information disclosure |

## Questions to Answer

1. What web server is being used?
2. Is the server version disclosed?
3. Are there any interesting custom headers?
4. What endpoints did you discover?
5. Does the application leak any sensitive information in error messages?

## Verification

- [ ] Identified web server and version
- [ ] Tested HTTP methods
- [ ] Found available endpoints
- [ ] Checked for information disclosure
- [ ] Documented findings

## Next Steps

Now that you've gathered reconnaissance data, move on to [Lab 03 - Test Path Traversal with dirb](lab-03-path-traversal.md) to discover hidden directories and files.

---

[← Previous: Start Server](lab-01-start-server.md) | [Next: Path Traversal Testing →](lab-03-path-traversal.md)
