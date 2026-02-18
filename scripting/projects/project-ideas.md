# Scripting Project Ideas

> Hands-on projects to build real Bash and Python automation skills.

## Beginner

- **System Info Script** — Write a Bash script that prints hostname, OS, CPU, memory, disk usage, and uptime in a clean report.
- **File Organiser** — Write a Python script that moves files in a directory into sub-folders by extension (e.g., `.jpg` → `images/`).

## Intermediate

<!-- Add intermediate project ideas here -->

- **Automated Backup** — Bash script that compresses a directory, names it with a timestamp, and syncs it to an S3 bucket using the AWS CLI.
- **Log Parser** — Python script that reads an Nginx/Apache access log, counts requests per status code, and outputs a summary.
- **AWS Resource Lister** — Python + boto3 script that lists EC2 instances, RDS databases, and S3 buckets across all regions.

## Advanced

<!-- Add advanced project ideas here -->

- **Infrastructure Health Check** — Python script that checks URLs, pings hosts, verifies SSL certificate expiry, and sends a Slack alert on failure.
- **Kubernetes Cleanup Bot** — Python script that finds and deletes completed Jobs and evicted Pods in a Kubernetes cluster using the K8s API.
- **CLI Tool with Click** — Build a polished CLI tool (e.g., a deployment helper) using Python + Click with subcommands, flags, and help text.

## Project Template

For each project, document:
1. **Goal** — What the script automates
2. **Language** — Bash / Python / both
3. **Dependencies** — External tools or libraries
4. **Usage** — `./script.sh --help` or `python script.py --help` output
5. **Code** — Full script with comments
