### 1. An organization receives multiple security findings from a CSPM tool. How would you decide which issues to fix first?
---
In this project, I used Prowler Local Server to assess my AWS environment and identify security misconfigurations.

I intentionally created two security issues: an EC2 Security Group allowing SSH access from the Internet and an S3 bucket with Block Public Access disabled.

Prowler generated four failed checks related to these two underlying issues. I investigated the affected resources and prioritized the SSH exposure because it increased the external attack surface.

I removed the unrestricted SSH rule, re-enabled S3 Block Public Access, and ran another assessment. All four targeted checks passed after remediation.

This project taught me to investigate the root cause of security findings rather than treating every finding as a separate problem.

### 2. Your security team needs to run a tool that depends on several services. Why would you use Docker Compose instead of starting each container manually?
---
I used Docker Compose to run and manage Prowler Local Server because it requires multiple containers to work together.

These containers provide different functions, such as the web interface, API, database, and background workers. Docker Compose allows me to define these services in YAML configuration files and start them together with a single command.

For example, I used `docker compose up -d` to start the services and `docker compose ps` to check their status.

I also created a Docker Compose override file to mount my AWS configuration into the required containers without modifying the original Prowler configuration. This allowed Prowler to access the credentials needed to connect to AWS securely.

Using Docker Compose made the setup easier to manage, restart, and maintain.

### 3. 