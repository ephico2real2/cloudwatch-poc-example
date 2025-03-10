# Setting Up the CloudWatch Agent on Ubuntu 22.04
## Monitoring Systemd Services and System Metrics

This guide details how to install, configure, and troubleshoot the AWS CloudWatch Agent on an Ubuntu 22.04 EC2 instance to monitor both system metrics and systemd service logs.

---

## Prerequisites

- **Ubuntu 22.04 LTS EC2 instance** – Ensure your instance is up to date
- **Internet connectivity** – The instance must have outbound access to AWS CloudWatch endpoints
- **AWS CLI** installed (optional but recommended for troubleshooting)
- **Proper IAM permissions** – The instance must have an IAM role that includes the CloudWatchAgentServerPolicy

---

## Part 1: Configure IAM Permissions

### Option A: Create a New IAM Role

1. Sign in to the AWS Management Console and open the IAM console

2. Navigate to Roles and click **Create role**

3. Select **AWS service** as the trusted entity, and choose **EC2**

4. In the permissions step, search for and select **CloudWatchAgentServerPolicy**

5. (Optional) Add tags to help identify the role

6. Name the role (e.g., `EC2CloudWatchRole`) and click **Create role**

7. Attach the role to your EC2 instance:
   - Open the EC2 console
   - Select your instance
   - Click **Actions > Security > Modify IAM role**
   - Choose the newly created role and click **Save**

### Option B: Use an Existing IAM Role

1. Open the IAM console and navigate to **Roles**

2. Find and select the role currently attached to your EC2 instance

3. Click **Add permissions > Attach policies**

4. Search for and select **CloudWatchAgentServerPolicy**

5. Click **Add permissions**

> **Tip:** After attaching or updating the role, restart your EC2 instance (or the agent) to ensure that new permissions are applied.

---

## Part 2: Install the CloudWatch Agent

Connect to your EC2 instance via SSH, then run:

```bash

CW_NAMESPACE="dotmobi-tools-api/CloudWatchAgentServer"
CW_CONFIG="/opt/aws/amazon-cloudwatch-agent/etc/amazon-cloudwatch-agent.json"

# Update the apt package index
apt-get -y update

# Install collectd
apt-get -y install collectd

# Update package lists
sudo apt-get update

# Download the CloudWatch Agent package
wget https://s3.amazonaws.com/amazoncloudwatch-agent/ubuntu/amd64/latest/amazon-cloudwatch-agent.deb

# Install the package
sudo dpkg -i -E ./amazon-cloudwatch-agent.deb

# Clean up the installer file
rm ./amazon-cloudwatch-agent.deb
```

---

## Part 3: Configure the CloudWatch Agent

### 1. Create the Configuration Directory and File

```bash
# Create directory if it doesn't exist
sudo mkdir -p /opt/aws/amazon-cloudwatch-agent/etc/

# Open the configuration file for editing
sudo vi /opt/aws/amazon-cloudwatch-agent/etc/amazon-cloudwatch-agent.json
```

### 2. Paste the Following JSON Configuration

This sample configuration collects both system logs (such as /var/log/syslog) and logs from selected systemd services. It also gathers key system metrics (CPU, disk, memory, network, and swap):

```json
{
  "agent": {
    "metrics_collection_interval": 60,
    "run_as_user": "root",
    "timezone": "Local"
  },
  "logs": {
    "logs_collected": {
      "files": {
        "collect_list": [
          {
            "file_path": "/var/log/*.log",
            "log_group_name": "system-logs",
            "log_stream_name": "{instance_id}-system",
            "retention_in_days": 14
          },
          {
            "file_path": "/usr/local/api/logs/zendesk_it_glue_*.log",
            "log_group_name": "zendesk-it-glue-logs",
            "log_stream_name": "{instance_id}-zendesk-it-glue",
            "retention_in_days": 14
          },
          {
            "file_path": "/usr/local/api/logs/it_glue_export_*.log",
            "log_group_name": "it-glue-backup-logs",
            "log_stream_name": "{instance_id}-it-glue-backup",
            "retention_in_days": 14
          },
          {
            "file_path": "/usr/local/api/logs/jamf_pro_*.log",
            "log_group_name": "jamf-pro-logs",
            "log_stream_name": "{instance_id}-jamf-pro",
            "retention_in_days": 14
          },
          {
            "file_path": "/usr/local/api/logs/zendesk_statusio_*.log",
            "log_group_name": "zendesk-statusio-logs", 
            "log_stream_name": "{instance_id}-zendesk-statusio",
            "retention_in_days": 14
          }
        ]
      }
    }
  },
  "metrics": {
    "namespace": "dotmobi-tools-api/CloudWatchAgentServer",
    "aggregation_dimensions": [
      [
        "InstanceId"
      ]
    ],
    "metrics_collected": {
      "collectd": {
        "metrics_aggregation_interval": 60
      },
      "cpu": {
        "measurement": [
          "cpu_usage_idle",
          "cpu_usage_iowait",
          "cpu_usage_user",
          "cpu_usage_system"
        ],
        "metrics_collection_interval": 60,
        "totalcpu": false
      },
      "disk": {
        "measurement": [
          "used_percent",
          "inodes_free"
        ],
        "metrics_collection_interval": 60,
        "resources": [
          "*"
        ]
      },
      "diskio": {
        "measurement": [
          "io_time"
        ],
        "metrics_collection_interval": 60,
        "resources": [
          "*"
        ]
      },
      "mem": {
        "measurement": [
          "mem_used_percent"
        ],
        "metrics_collection_interval": 60
      },
      "swap": {
        "measurement": [
          "swap_used_percent"
        ],
        "metrics_collection_interval": 60
      }
    }
  }
}
```

> **Note:** Adjust the JSON as needed for your specific services or if you have additional log files/metrics to capture. Use a JSON validator to ensure proper formatting before saving.

### 3. Save and Exit

In nano, press Ctrl+O to save, then Enter, and Ctrl+X to exit.

---

## Part 4: Start and Enable the CloudWatch Agent

### 1. Start the Agent with Your Configuration

```bash
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl -a fetch-config -m ec2 -s -c file:/opt/aws/amazon-cloudwatch-agent/etc/amazon-cloudwatch-agent.json
```

### 2. Verify the Agent's Status

```bash
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl -m ec2 -a status
```

### 3. Enable the Agent to Start on Boot

```bash
sudo systemctl enable amazon-cloudwatch-agent
```

---

## Part 5: Verify the CloudWatch Agent Installation

### 1. Check the Service Status

```bash
sudo systemctl status amazon-cloudwatch-agent
```

### 2. Examine the Agent Logs

```bash
sudo tail -f /var/log/amazon/amazon-cloudwatch-agent/amazon-cloudwatch-agent.log
```

> **Tip:** If you suspect configuration issues, run:
> ```bash
> sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl -a verify-config -m ec2 -c file:/opt/aws/amazon-cloudwatch-agent/etc/amazon-cloudwatch-agent.json
> ```

---

## Part 6: Verify Metrics and Logs in the CloudWatch Console

### 1. Metrics:
- In the AWS CloudWatch console, navigate to **Metrics > All metrics**
- Look under the **CWAgent** namespace for your system and custom metrics

### 2. Logs:
- Go to **Logs > Log groups**
- Confirm the presence of log groups such as:
  - `system-logs`

---

## Part 7: Create a CloudWatch Dashboard (Optional)

1. Open the CloudWatch console and navigate to **Dashboards > Create dashboard**

2. Name your dashboard (e.g., `UbuntuServerMetrics`) and click **Create dashboard**

3. Add widgets to display metrics like CPU, memory, and disk usage:
   - Namespace: `CWAgent`
   - Metric: Select the desired metric (e.g., `cpu_usage_user`)
   - Statistic: `Average`
   - Period: `1 minute`

4. Save your dashboard.

---

## Part 8: Set Up CloudWatch Alarms (Optional)

1. In the CloudWatch console, go to **Alarms > Create alarm**

2. Click **Select metric** and navigate to the CWAgent namespace

3. Choose a relevant metric (for example, `memory_used_percent`)

4. Define conditions (e.g., trigger an alarm if usage is ≥80% for 5 consecutive minutes)

5. Configure your notification actions (such as an SNS topic or email)

6. Provide a name and description, then create the alarm

---

## Troubleshooting

### Agent Not Starting or Configuration Errors

**Verify JSON Configuration:**
Use the verify command to catch syntax errors:
```bash
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl -a verify-config -m ec2 -c file:/opt/aws/amazon-cloudwatch-agent/etc/amazon-cloudwatch-agent.json

# Start CloudWatch agent
/opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl -a fetch-config -m ec2 -s -c file:$CW_CONFIG
```

**Check Service Status:**
```bash
sudo systemctl status amazon-cloudwatch-agent
```

And view logs with:
```bash
journalctl -u amazon-cloudwatch-agent
```

### No Metrics or Logs Appearing in CloudWatch

**1. IAM Role Permissions:**
Ensure your instance role includes CloudWatchAgentServerPolicy. Check the IAM role attached by running:
```bash
curl http://169.254.169.254/latest/meta-data/iam/security-credentials/
```

**2. Network Connectivity:**
Confirm your instance can reach CloudWatch endpoints:
```bash
curl -v https://monitoring.<your-region>.amazonaws.com
curl -v https://logs.<your-region>.amazonaws.com
```
Replace `<your-region>` with your AWS region (e.g., us-east-1).

**3. Log File Permissions:**
Make sure the CloudWatch agent has read permissions for the log files it's trying to collect.

### Restarting the Agent After Making Changes

After updating the configuration file, restart the agent:
```bash
sudo systemctl restart amazon-cloudwatch-agent
```

---

## Understanding Collected Data

### Service Logs
- **What's Collected:** Start/stop events, script outputs (stdout/stderr), and any service-specific errors or warnings
- **Usage:** Use these logs for troubleshooting service issues

### System Metrics
**Key Metrics:**
- **CPU:** High usage (>80%) may indicate processing bottlenecks
- **Memory:** High utilization (>90%) can lead to swapping and degraded performance
- **Disk:** Monitor for high usage (>85%) to avoid running out of space
- **Network:** Watch for unexpected spikes that could signal issues

---

## CloudWatch Costs

- **Metrics:** Charges apply after the free tier
- **Logs:** Costs are incurred based on log ingestion and storage
- **Dashboards:** Monitor usage as additional widgets may incur costs

> **Cost Tip:** Adjust log retention periods and consider filtering out less critical metrics to control costs.
