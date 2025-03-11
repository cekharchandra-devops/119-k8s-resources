# sidecar 

What is the Sidecar Pattern?
The Sidecar Pattern in Kubernetes involves deploying an additional container alongside the main application container within the same Pod. This sidecar container enhances the functionality of the primary application without modifying its code.

Key Benefits:

**Separation of Concerns:** Keeps the main application lightweight while delegating auxiliary tasks to the sidecar.

**Reusability:** The sidecar container can be used across multiple applications.

**Scalability & Observability:** Enhances logging, monitoring, and networking functionalities.

# Real-Time Scenarios Where the Sidecar Pattern is Used

1️⃣ Logging & Monitoring (Fluentd as a Sidecar)

🔹 Scenario: An application generates logs, but you need to forward them to Elasticsearch, CloudWatch, or Splunk. Instead of modifying the app, use a sidecar container with Fluentd to collect and ship logs.


Install Fluentd & CloudWatch plugin:

```
gem install fluent-plugin-cloudwatch-logs

```
Configure Fluentd (fluent.conf)

```
<source>
  @type tail
  path /var/log/app.log
  pos_file /var/log/app.log.pos
  tag myapp.logs
  format none
</source>

<match myapp.logs>
  @type cloudwatch_logs
  log_group_name my-app-log-group
  log_stream_name my-app-log-stream
  region us-east-1
  auto_create_stream true
</match>

```
Ensure Fluentd has IAM permissions

Attach the following IAM policy to the Fluentd instance or container:

```
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "logs:PutLogEvents",
        "logs:DescribeLogStreams",
        "logs:CreateLogStream"
      ],
      "Resource": "arn:aws:logs:us-east-1:YOUR_ACCOUNT_ID:log-group:my-app-log-group:*"
    }
  ]
}

```



```
apiVersion: v1
kind: Pod
metadata:
  name: app-with-logging
spec:
  containers:
    - name: my-app
      image: my-app:latest
      volumeMounts:
        - name: logs
          mountPath: /var/log/app
    - name: fluentd-sidecar
      image: fluent/fluentd:latest
      volumeMounts:
        - name: logs
          mountPath: /var/log/app
  volumes:
    - name: logs
      emptyDir: {}
```

fluent.conf:

```
# Set log level
<system>
  log_level info
</system>

## 🌟 1️⃣ Input: Collect logs from multiple sources
# Example: Collect logs from files (e.g., /var/log/app.log)
<source>
  @type tail
  path /var/log/app.log
  pos_file /var/log/fluentd-app.pos
  tag app.logs
  format json
</source>

# Collect system logs (journald for systemd-based Linux)
<source>
  @type systemd
  tag system.logs
  path /var/log/messages
</source>

## 🌟 2️⃣ Filter: Enrich or modify logs before forwarding
<filter **>
  @type record_transformer
  <record>
    hostname "#{Socket.gethostname}"
    environment production
  </record>
</filter>

## 🌟 3️⃣ Output: Send logs to CloudWatch, ELK, or Splunk

# Send logs to AWS CloudWatch
<match app.logs>
  @type cloudwatch_logs
  log_group_name fluentd-app-logs
  log_stream_name logs-stream
  region us-east-1
</match>

# Send logs to ELK (Elasticsearch)
<match **>
  @type elasticsearch
  host elasticsearch.example.com
  port 9200
  logstash_format true
  user fluentd
  password yourpassword
</match>

# Send logs to Splunk
<match system.logs>
  @type splunk_hec
  hec_host splunk.example.com
  hec_port 8088
  token YOUR_SPLUNK_TOKEN
  index my-index
  sourcetype my-source
</match>

# Optional: Send unmatched logs to a local file for debugging
<match **>
  @type file
  path /var/log/fluentd-debug.log
</match>
```