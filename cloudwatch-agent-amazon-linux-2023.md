# Configure CloudWatch Agent on Amazon Linux 2023

This guide configures an EC2 instance to send application log files to Amazon CloudWatch Logs using the Amazon CloudWatch Agent.

> **Important:** Replace `/opt/myapp/logs/application.log` with the actual path of your application's log file.

## 1. Create and attach the IAM role

In the AWS Console:

1. Open **IAM → Roles → Create role**.
2. Select **AWS service → EC2**.
3. Attach the AWS-managed policy `CloudWatchAgentServerPolicy`.
4. Create the role, for example, `EC2-CloudWatchAgent-Role`.
5. Open **EC2 → Instances**, select your instance, then choose **Actions → Security → Modify IAM role**.
6. Attach the role to the instance.

If the instance already has an IAM role, add the policy to that role instead of replacing it.

## 2. Connect to EC2 and install the agent

Connect to the instance using SSH or AWS Systems Manager Session Manager.

Check the OS and architecture:

```bash
cat /etc/os-release
uname -m
```

Install the agent:

```bash
sudo dnf install -y amazon-cloudwatch-agent
```

Verify installation:

```bash
rpm -q amazon-cloudwatch-agent
```

## 3. Configure the application log file

Check that your application log exists:

```bash
sudo ls -l /opt/myapp/logs/application.log
```

Create the CloudWatch Agent configuration file:

```bash
sudo tee /opt/aws/amazon-cloudwatch-agent/etc/amazon-cloudwatch-agent.json > /dev/null <<'EOF'
{
  "agent": {
    "logfile": "/opt/aws/amazon-cloudwatch-agent/logs/amazon-cloudwatch-agent.log"
  },
  "logs": {
    "logs_collected": {
      "files": {
        "collect_list": [
          {
            "file_path": "/opt/myapp/logs/application.log",
            "log_group_name": "/aws/ec2/myapp/application",
            "log_stream_name": "{instance_id}",
            "timezone": "UTC"
          }
        ]
      }
    }
  }
}
EOF
```

Configuration details:

| Setting | Value |
|---|---|
| EC2 log file | `/opt/myapp/logs/application.log` |
| CloudWatch log group | `/aws/ec2/myapp/application` |
| Log stream | EC2 instance ID |
| Timestamp timezone | UTC |

The log group can be created automatically by the agent if its IAM permissions allow it.

### Check file permissions

The agent commonly runs as the `cwagent` user. Check whether it can read the log file:

```bash
sudo -u cwagent test -r /opt/myapp/logs/application.log \
  && echo "Readable by cwagent" \
  || echo "Permission issue"
```

If access is denied, grant the agent read access to the file and execute access to its parent directories. Avoid making application logs world-readable.

## 4. Start the CloudWatch Agent

Load the configuration and start the agent:

```bash
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl \
  -a fetch-config \
  -m ec2 \
  -c file:/opt/aws/amazon-cloudwatch-agent/etc/amazon-cloudwatch-agent.json \
  -s
```

Check the agent status:

```bash
sudo /opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl \
  -a status
```

Check the system service:

```bash
sudo systemctl status amazon-cloudwatch-agent
```

Enable startup at boot:

```bash
sudo systemctl enable amazon-cloudwatch-agent
```

## 5. Generate a test log entry

Append a test message to the configured application log:

```bash
echo "$(date -u '+%Y-%m-%dT%H:%M:%SZ') INFO CloudWatch test message from EC2" \
  | sudo tee -a /opt/myapp/logs/application.log
```

Confirm the entry exists locally:

```bash
sudo tail -5 /opt/myapp/logs/application.log
```

Wait briefly for the agent to publish the event.

## 6. Verify logs in CloudWatch

1. Open the AWS Console.
2. Go to **CloudWatch → Logs → Log groups**.
3. Find `/aws/ec2/myapp/application`.
4. Open the log group and select the log stream named with your EC2 instance ID.
5. Confirm that you can see `CloudWatch test message from EC2`.

The log group may take a short time to appear after the first successful upload.

## 7. Troubleshooting

### Check the agent's own log

```bash
sudo tail -100 /opt/aws/amazon-cloudwatch-agent/logs/amazon-cloudwatch-agent.log
```

### Check IAM credentials

If AWS CLI is installed, run:

```bash
aws sts get-caller-identity
```

This should return the active AWS identity. Note that the CLI may use a different credential configuration, so this check alone does not prove that the agent can authenticate.

### Check network access

The instance needs outbound HTTPS access (TCP 443) to the CloudWatch Logs endpoint in its AWS Region, either through internet/NAT connectivity or an appropriately configured VPC interface endpoint.

### Check the configuration

```bash
sudo cat /opt/aws/amazon-cloudwatch-agent/etc/amazon-cloudwatch-agent.json
```

### Check service status

```bash
sudo systemctl status amazon-cloudwatch-agent
```

## Final checklist

- [ ] EC2 IAM role has `CloudWatchAgentServerPolicy`.
- [ ] CloudWatch Agent is installed.
- [ ] Application log path is correct and readable.
- [ ] Agent status reports running.
- [ ] Test message appears in the CloudWatch log stream.

## Official AWS documentation

- [Install the CloudWatch Agent on EC2](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/install-CloudWatch-Agent-on-EC2-Instance.html)
- [Troubleshoot the CloudWatch Agent](https://docs.aws.amazon.com/AmazonCloudWatch/latest/monitoring/troubleshooting-CloudWatch-Agent.html)
