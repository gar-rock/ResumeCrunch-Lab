# Lab 05 - Test for Known Vulnerabilities with pip-audit

## Objective

Use `pip-audit` (or `pip-audit-extra`) to scan the `requirements.txt` file for known vulnerabilities in Python dependencies.

## Background

`pip-audit` is a tool that scans Python packages for known security vulnerabilities by:
- Checking packages against the PyPI Advisory Database
- Identifying CVEs (Common Vulnerabilities and Exposures)
- Providing severity ratings
- Suggesting fixes and upgrades

This automated scanning can quickly identify low-hanging fruit for exploitation.

## Prerequisites

### Install pip-audit

```bash
# Install pip-audit
pip install pip-audit

# Or install the enhanced version
pip install pip-audit-extra
```

Alternatively, use pipx for isolated installation:

```bash
pipx install pip-audit
```

## Steps

### 1. Navigate to Your Analysis Directory

```bash
cd resumecrunch-analysis
# Ensure requirements.txt is present
ls -la requirements.txt
```

### 2. Run Basic pip-audit Scan

Scan the requirements.txt file:

```bash
pip-audit -r requirements.txt
```

This will output:
- Package names
- Installed versions
- CVE identifiers
- Vulnerability descriptions
- Fixed versions

### 3. Run Detailed Scan with JSON Output

For more detailed analysis, output to JSON:

```bash
pip-audit -r requirements.txt -f json -o vulnerabilities.json

# View the JSON output
cat vulnerabilities.json | python3 -m json.tool
```

### 4. Generate a Report

Create a formatted vulnerability report:

```bash
pip-audit -r requirements.txt -f markdown -o vulnerability-report.md

# View the report
cat vulnerability-report.md
```

### 5. Analyze Vulnerability Severity

The output will show severity levels:
- **CRITICAL** - Immediate action required
- **HIGH** - Should be fixed soon
- **MEDIUM** - Fix when possible
- **LOW** - Minimal risk

Focus on CRITICAL and HIGH severity vulnerabilities.

### 6. Research Specific CVEs

For each CVE found, research the details:

```bash
# Example: If CVE-2023-12345 is found
# Search for it online
open "https://nvd.nist.gov/vuln/detail/CVE-2023-12345"
```

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
