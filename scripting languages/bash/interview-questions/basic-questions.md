# Bash — Basic Questions

---

## 1. Write a shell script to alert when CPU utilisation reaches 80%.

> **Also asked as:** "Write a script to alert when CPU utilization exceeds 80%"

```bash
#!/bin/bash
# cpu-alert.sh — checks CPU utilisation and sends an alert if above threshold

THRESHOLD=80
LOG_FILE="/var/log/cpu-alert.log"
ALERT_EMAIL="devops-team@company.com"

# Get current CPU usage (idle% from top, subtract from 100)
CPU_IDLE=$(top -bn1 | grep "Cpu(s)" | awk '{print $8}' | cut -d'%' -f1)
CPU_USAGE=$(echo "100 - $CPU_IDLE" | bc)

# Round to integer for comparison
CPU_INT=${CPU_USAGE%.*}

TIMESTAMP=$(date '+%Y-%m-%d %H:%M:%S')
HOSTNAME=$(hostname)

echo "[$TIMESTAMP] CPU Usage: ${CPU_USAGE}% on $HOSTNAME" >> "$LOG_FILE"

if [ "$CPU_INT" -ge "$THRESHOLD" ]; then
    MESSAGE="ALERT: CPU usage is ${CPU_USAGE}% on $HOSTNAME (threshold: ${THRESHOLD}%)"
    echo "[$TIMESTAMP] $MESSAGE" >> "$LOG_FILE"

    # Option 1: Send email alert
    echo "$MESSAGE" | mail -s "CPU Alert - $HOSTNAME" "$ALERT_EMAIL"

    # Option 2: Send Slack alert via webhook
    curl -s -X POST "$SLACK_WEBHOOK_URL" \
      -H 'Content-type: application/json' \
      -d "{\"text\": \"$MESSAGE\"}"

    # Option 3: Log only (for testing)
    echo "$MESSAGE"
fi
```

**Alternative using `/proc/stat` (more accurate, no dependency on `top`):**

```bash
#!/bin/bash
# More accurate CPU calculation using /proc/stat

THRESHOLD=80

get_cpu_usage() {
    # Read two snapshots 1 second apart, calculate the difference
    read -r cpu user nice system idle iowait irq softirq steal < /proc/stat

    total1=$((user + nice + system + idle + iowait + irq + softirq + steal))
    idle1=$idle

    sleep 1

    read -r cpu user nice system idle iowait irq softirq steal < /proc/stat

    total2=$((user + nice + system + idle + iowait + irq + softirq + steal))
    idle2=$idle

    total_diff=$((total2 - total1))
    idle_diff=$((idle2 - idle1))

    cpu_usage=$(( (total_diff - idle_diff) * 100 / total_diff ))
    echo $cpu_usage
}

CPU=$(get_cpu_usage)
HOSTNAME=$(hostname)
TIMESTAMP=$(date '+%Y-%m-%d %H:%M:%S')

if [ "$CPU" -ge "$THRESHOLD" ]; then
    echo "[$TIMESTAMP] ALERT: CPU at ${CPU}% on $HOSTNAME"
    # Add your notification logic here (email, Slack, SNS, etc.)
fi
```

**Schedule with cron to run every 5 minutes:**

```bash
# Make the script executable
chmod +x /opt/scripts/cpu-alert.sh

# Add to crontab
crontab -e
*/5 * * * * /opt/scripts/cpu-alert.sh
```

**Send alert via AWS SNS (common in cloud environments):**

```bash
#!/bin/bash
THRESHOLD=80
SNS_TOPIC_ARN="arn:aws:sns:ap-south-1:123456789012:cpu-alerts"

CPU_IDLE=$(top -bn1 | grep "Cpu(s)" | awk '{print $8}' | cut -d'%' -f1)
CPU_USAGE=$(echo "100 - $CPU_IDLE" | bc)
CPU_INT=${CPU_USAGE%.*}

if [ "$CPU_INT" -ge "$THRESHOLD" ]; then
    aws sns publish \
      --topic-arn "$SNS_TOPIC_ARN" \
      --message "CPU ALERT: $(hostname) is at ${CPU_USAGE}% CPU utilisation" \
      --subject "CPU Alert - $(hostname)"
fi
```
