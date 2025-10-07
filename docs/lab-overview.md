# ResumeCrunch Lab Overview

Welcome to the ResumeCrunch Security Testing Lab! This hands-on lab will guide you through discovering and exploiting vulnerabilities in a vulnerable web application.

## Lab Structure

This lab is divided into several phases:

1. **[Start the Web Server](lab-01-start-server.md)** - Get the application up and running
2. **[Interrogate the Web Server](lab-02-interrogate-server.md)** - Gather information about the target
3. **[Test Path Traversal with dirb](lab-03-path-traversal.md)** - Use dirb to discover hidden paths
4. **[Locate Source Code and Requirements](lab-04-locate-source.md)** - Find the application source code
5. **[Test for Known Vulnerabilities](lab-05-test-vulns.md)** - Use pip-audit to find vulnerable dependencies
6. **[Poison Redis Cache](lab-06-redis-exploit.md)** - Deploy a reverse shell via cache poisoning

## Prerequisites

Before starting this lab, ensure you have:

- Docker and Docker Compose installed
- Basic knowledge of web application security
- Familiarity with command-line tools
- The following tools installed:
  - `curl`
  - `dirb` or `dirbuster`
  - `pip-audit` (can be installed during the lab)
  - `netcat` or `nc`

## Learning Outcomes

By completing this lab, you will:

- Understand how to enumerate web applications
- Learn directory brute-forcing techniques
- Identify vulnerable dependencies in Python projects
- Exploit cache poisoning vulnerabilities
- Gain experience with reverse shell deployment

## Lab Environment

This lab uses a deliberately vulnerable application for educational purposes. **Never** use these techniques against systems you don't own or have explicit permission to test.

## Ready to Begin?

Start with [Lab 01 - Start the Web Server](lab-01-start-server.md)
