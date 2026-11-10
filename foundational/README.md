# Task 2: Log Analysis & SIEM Mini Dashboard

A small SIEM lab built on the **ELK Stack** (Elasticsearch, Kibana, Filebeat) running on a Kali Linux VM. It collects Linux authentication logs, ingests them into Elasticsearch, and raises alerts on suspicious activity such as SSH brute-force attempts.

## Objective

Build a basic SIEM setup to analyze logs and detect suspicious activity:

- Collect and preprocess sample system logs from Linux or Windows
- Ingest log data into ELK Stack or Graylog
- Configure at least 3 alert rules (failed logins, IP anomalies, port scans)
- Design a mini dashboard to visualize log insights and security events
- Demonstrate real-time detection of abnormal patterns

## Status

| Item | Status |
|------|--------|
| ELK stack installed and running | Done |
| Linux logs collected (Filebeat `system` module) | Done |
| Logs ingested and searchable in Kibana Discover | Done |
| Alert rule 1: SSH brute force | Configured, see [rules/ssh-brute-force.md](rules/ssh-brute-force.md) |
| Alert rule 2: port scan | Pending |
| Alert rule 3: invalid user / root login | Pending |
| Mini dashboard | Pending |
| Real-time detection demo | Pending |

## Environment

| Component | Details |
|-----------|---------|
| Host OS | Kali Linux VM (Oracle VirtualBox) |
| VM resources | 4 GB RAM and 2 CPUs recommended (Elasticsearch and Kibana struggle on 2 GB) |
| Elasticsearch / Kibana / Filebeat | 8.19.x (Elastic 8.x apt repository) |
| Log sources | `/var/log/auth.log`, `/var/log/syslog` |
| Attack tool (testing) | Hydra, run against the lab VM's own SSH service |

## Architecture

```
/var/log/auth.log, /var/log/syslog
              |
        Filebeat (system module)
              |
        Elasticsearch  (https://localhost:9200)
              |
   Kibana: Discover, Rules (alerting), Dashboards
```

## Setup steps

### 1. Prerequisites

```bash
sudo apt update
sudo apt install -y rsyslog openssh-server ufw curl gnupg apt-transport-https
sudo systemctl enable --now rsyslog ssh
```

`rsyslog` is needed so SSH events are written to `/var/log/auth.log`.

### 2. Install Elasticsearch, Kibana and Filebeat

```bash
curl -fsSL https://artifacts.elastic.co/GPG-KEY-elasticsearch | sudo gpg --dearmor -o /usr/share/keyrings/elastic.gpg
echo "deb [signed-by=/usr/share/keyrings/elastic.gpg] https://artifacts.elastic.co/packages/8.x/apt stable main" | sudo tee /etc/apt/sources.list.d/elastic-8.x.list
sudo apt update
sudo apt install -y elasticsearch kibana filebeat
```

The Elasticsearch install prints a password for the `elastic` user once. Store it somewhere safe, and never commit it.

### 3. Start Elasticsearch and verify

```bash
sudo systemctl enable --now elasticsearch
sudo curl --cacert /etc/elasticsearch/certs/http_ca.crt -u elastic https://localhost:9200
```

A JSON response with `cluster_name` and the version confirms it works. If the password is lost:

```bash
sudo /usr/share/elasticsearch/bin/elasticsearch-reset-password -u elastic -f
```

### 4. Start Kibana and enroll it

```bash
sudo systemctl enable --now kibana
sudo /usr/share/elasticsearch/bin/elasticsearch-create-enrollment-token -s kibana
```

Open `http://localhost:5601`, paste the enrollment token, and log in as `elastic`.

### 5. Configure Filebeat

Edit `/etc/filebeat/filebeat.yml` (full sample in [configs/filebeat.yml.sample](configs/filebeat.yml.sample)):

```yaml
output.elasticsearch:
  hosts: ["https://localhost:9200"]
  preset: balanced
  username: "elastic"
  password: "YOUR_PASSWORD"
  ssl.certificate_authorities: ["/etc/elasticsearch/certs/http_ca.crt"]
```

Keep `output.logstash` commented out, since only one output can be active. For anything beyond a lab, use a dedicated Filebeat user or API key instead of the `elastic` superuser.

### 6. Enable the system module and start Filebeat

```bash
sudo filebeat modules enable system
sudo sed -i 's/enabled: false/enabled: true/' /etc/filebeat/modules.d/system.yml
sudo filebeat test config
sudo filebeat test output
sudo filebeat setup --index-management
sudo systemctl enable --now filebeat
```

`test output` should end with `talk to server... OK`. Enabling both the `syslog` and `auth` filesets is what produces SSH login events.

### 7. Create the data view

In Kibana: **Stack Management > Data Views > Create data view**

- Name: `filebeat-*`
- Index pattern: `filebeat-*`
- Timestamp field: `@timestamp`

### 8. Verify ingestion

Generate a few failed SSH logins and look for them in **Discover**:

```bash
ssh fakeuser@localhost     # enter a wrong password several times
```

KQL search:

```
event.dataset : "system.auth" and event.outcome : "failure"
```

### 9. Enable alerting (encryption keys)

Kibana alerting requires encryption keys. Generate your own and add them to `/etc/kibana/kibana.yml`, then restart Kibana:

```bash
sudo /usr/share/kibana/bin/kibana-encryption-keys generate -q | sudo tee -a /etc/kibana/kibana.yml
sudo systemctl restart kibana
```

Do not commit `kibana.yml` with these keys in it.

## Alert rule 1: SSH brute force

| Setting | Value |
|---------|-------|
| Rule type | Elasticsearch query (KQL) |
| Data view | `filebeat-*` |
| Query | `event.dataset : "system.auth" and event.outcome : "failure"` |
| Condition | count is above 5 |
| Time window | last 5 minutes (can be tightened to 1 minute for a faster demo) |
| Check interval | every 1 minute |
| Action | Server log, level `warn` |

Details and test notes: [rules/ssh-brute-force.md](rules/ssh-brute-force.md).

## Testing

A burst of failed SSH logins is generated against the lab VM's own SSH service. This is for testing in an isolated lab only.

```bash
printf '%s\n' pass1 pass2 pass3 pass4 pass5 pass6 pass7 pass8 > ~/pw.txt
hydra -l fakeuser -P ~/pw.txt ssh://127.0.0.1 -t 4
```

Hydra completed 15 login tries with no valid password found, which produces a spike of `system.auth` failure events in Kibana.

## Issues encountered and fixes

| Problem | Cause | Fix |
|---------|-------|-----|
| `Unable to locate package elasticsearch` | Typos in the apt repository line (`https//:` and a missing `/` in the keyring path) | Rewrote `/etc/apt/sources.list.d/elastic-8.x.list` correctly |
| Cluster health RED, password reset failed with exit code 75 | Root disk was about 95% full, so Elasticsearch stopped allocating shards | Freed disk space (`apt clean`, `journalctl --vacuum-size`), then reset the password |
| Filebeat `x509: certificate signed by unknown authority` | `ssl.certificate_authorities` was misspelled, so the CA certificate was never loaded | Corrected the setting name |
| `yaml: line 178: could not find expected ':'` | Broken indentation and quoting in `filebeat.yml` | Restored the 2-space indentation and closing quote |
| `Directory '/etc/filebeat/module.ds' does not exist` | Path typo | The folder is `modules.d` |
| "Additional setup required" in Kibana Rules | No alerting encryption key configured | Generated keys with `kibana-encryption-keys` and restarted Kibana |
| No data views in the rule form | Discover's "All logs" view is temporary and is not selectable in rules | Created the `filebeat-*` data view manually |

## Screenshots

Stored in [screenshots/](screenshots/). Blur or crop passwords and keys before committing.

1. Elasticsearch responding at `https://localhost:9200`
2. `filebeat test output` with all checks OK
3. Discover showing `system.auth` failure events
4. Rule 1 configuration page
5. Hydra output
6. Rule 1 showing Active or an alert fired

## Security notes

- Never commit the `elastic` password, enrollment tokens, or `xpack.*.encryptionKey` values.
- Use the `.sample` config files in this repo with placeholders. Keep real credentials out of version control.
- If a secret is ever committed, rotate it. Deleting the file does not remove it from git history.
- Hydra was run only against this lab VM.

## Next steps

- Alert rule 2: port scan detection (needs firewall logging enabled)
- Alert rule 3: invalid user or root login attempts
- Mini dashboard: failed logins over time, top source IPs, targeted usernames, recent security events
- Real-time detection demo with the dashboard auto-refreshing
