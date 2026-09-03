# Lab Report: Monitoring, Logging & Incident Detection

---

**Course:** IKB42603 Cloud Computing Security Essentials  

**Student Name:** MUHAMMAD ALIFF AQLAN BINMUHAMMAD ARIFF 

**Lecturer:** MADAM ADANI 

---

## Lab Learning Outcomes

At the end of this lab, I was able to:

1. ✅ Collect and **centralise logs** from multiple services (cloud telemetry)
2. ✅ Distinguish **logs from events** and query logs for security-relevant activity
3. ✅ Build a **tamper-evident (hash-chained)** log and detect alteration
4. ✅ **Detect an incident** by correlating events (brute-force followed by suspicious action)
5. ✅ Execute **incident-response** steps: detect, contain, collect evidence, and document a timeline

---

# Session A (Week 9) - Logging & Centralisation

## Setup & Configuration

### Starting LocalStack

```
# Start LocalStack container
docker run -d --name localstack -p 4566:4566 localstack/localstack
```
```
# Set the AWS CLI endpoint and create a log group and log stream.
EP='--endpoint-url=http://localhost:4566'
aws $EP logs create-log-group --log-group-name /ccse/app
aws $EP logs create-log-stream --log-group-name /ccse/app --log-stream-name auth
```

---

## Task 1: Generate Application Logs

### Objective
Initialize a simulated local authentication event log containing standard user entries, successive failed login attempts to mimic a threat actor probing the system, and a subsequent data export operation.

### Commands Executed


Create authentication log file
```
cat > auth.log << 'EOF'
2025-03-01T09:00:01 LOGIN_OK user=ahmad ip=10.0.0.5
2025-03-01T09:01:10 LOGIN_FAIL user=admin ip=203.0.113.9
2025-03-01T09:01:12 LOGIN_FAIL user=admin ip=203.0.113.9
2025-03-01T09:01:15 LOGIN_FAIL user=admin ip=203.0.113.9
2025-03-01T09:01:18 LOGIN_FAIL user=admin ip=203.0.113.9
2025-03-01T09:01:22 LOGIN_OK user=admin ip=203.0.113.9
2025-03-01T09:01:40 EXPORT_DATA user=admin ip=203.0.113.9 size=500MB
EOF
```


Verify the log file
```
cat auth.log
```

Evidence  : <img width="638" height="327" alt="Image" src="https://github.com/user-attachments/assets/3f173510-ced8-413d-b55c-cf9eef893bc0" />

## Task 2: Centralise Logs Ship to CloudWatch

Transmit the generated local log events sequentially to a centralized CloudWatch log stream via LocalStack to ensure system-wide audit visibility and prevent local log manipulation.

Initialize timestamp variable and push local log events to CloudWatch.

```
TS=$(date +%s000)
while IFS= read -r line; do
  aws --endpoint-url=http://localhost:4566 logs put-log-events \
    --log-group-name /ccse/app \
    --log-stream-name auth \
    --log-events timestamp=$TS,message="$line" >/dev/null
  TS=$((TS+1000))
done < auth.log
```

Retrieve the centralized logs back from CloudWatch for verification

```
aws --endpoint-url=http://localhost:4566 logs get-log-events \
  --log-group-name /ccse/app \
  --log-stream-name auth \
  --query 'events[].message' \
  --output text
```

Evidence :  <img width="910" height="191" alt="Image" src="https://github.com/user-attachments/assets/d98c0adc-958e-4a46-9731-5b928ad139e0" />

## Task 3: Query Security Relevant Activity

Extract and filter security-relevant indicators from the log files, specifically targeting failed login counts grouped by originating IP addresses to identify potential brute-force behavior.

Filter failed logins, extract IP address fields, sort, and count occurrences.

```
grep LOGIN_FAIL auth.log | awk '{print $4, $5}' | sort | uniq -c
```

Evidence : <img width="603" height="52" alt="Image" src="https://github.com/user-attachments/assets/e1b5972a-d680-45db-8d08-18d4e7c96c3b" />


## Task 4: Tamper-Proof / Hash-Chained Logs

Implement a simple integrity verification mechanism using hash chaining on the log files to detect any unauthorized alterations or local tampering.

Create a script or sequence to generate hash-chained logs (example verification step)
```
sha256sum auth.log
```

Output

```
e3b0c44298fc1c149afbf4c8996fb92427ae41e4649b934ca495991b7852b855  auth.log
```

## Implement a cryptographic hash-chaining mechanism across sequential log records to guarantee data immutability and immediately detect any unauthorized modifications or local tampering.

Generate hash-chained log for the original file
```
PREV=0
while IFS= read -r line; do
  PREV=$(printf '%s%s' "$PREV" "$line" | sha256sum | cut -d' ' -f1)
  printf '%s | %s\n' "$line" "$PREV"
done < auth.log > auth.chain
```

Simulate tampering by modifying an entry in the log file
```
sed 's/500MB/5MB/' auth.log > auth.tampered
```

Generate hash chain for the tampered file
```
PREV=0
while IFS= read -r line; do
  PREV=$(printf '%s%s' "$PREV" "$line" | sha256sum | cut -d' ' -f1)
  printf '%s | %s\n' "$line" "$PREV"
done < auth.tampered > auth.tampered.chain
```

Extract final chain values and compare integrity
```
ORIGINAL_FINAL=$(tail -1 auth.chain | awk -F ' \\| ' '{print $2}')
TAMPERED_FINAL=$(tail -1 auth.tampered.chain | awk -F ' \\| ' '{print $2}')

if [ "$ORIGINAL_FINAL" = "$TAMPERED_FINAL" ]; then
  echo "[INTEGRITY CHECK] LOGS INTACT: No tampering detected."
else
  echo "[INTEGRITY CHECK] TAMPERING DETECTED: Log chain mismatch!"
fi
```

Evidence : <img width="891" height="122" alt="Image" src="https://github.com/user-attachments/assets/ad938e93-e23f-42c1-bf9b-f51e687b772e" />

## Task 5: Detect the Incident (Correlation)

Correlate multiple sequential security events from a specific source IP address to detect a compound attack pattern consisting of repeated login failures, a successful authentication, and a subsequent data export.

```
IP=203.0.113.9
FAILS=$(grep -c "LOGIN_FAIL.*$IP" auth.log)
SUCCESS=$(grep -c "LOGIN_OK.*$IP" auth.log)
EXPORT=$(grep -c "EXPORT_DATA.*$IP" auth.log)
echo "IP=$IP fails=$FAILS success=$SUCCESS export=$EXPORT"

if [ "$FAILS" -ge 3 ] && [ "$SUCCESS" -ge 1 ] && [ "$EXPORT" -ge 1 ]; then
  echo 'ALERT: probable brute-force -> compromise -> data exfiltration'
fi
```

Evidence : <img width="672" height="207" alt="Image" src="https://github.com/user-attachments/assets/79a5f6c5-5049-422c-a15d-0fb36b5fcffe" />

## Task 6: Incident Response

Execute the incident response workflow by containing the malicious source IP address via firewall rules, collecting an immutable timestamped evidence copy with its cryptographic hash, and documenting the incident.

CONTAIN: block the attacker IP using iptables inside a temporary container network
```
docker run --rm --cap-add=NET_ADMIN alpine sh -c \
'apk add -q iptables; iptables -A INPUT -s 203.0.113.9 -j DROP; iptables -L INPUT -n | tail -2'
```

COLLECT: make an immutable, timestamped evidence copy with its hash
```
cp auth.log evidence_$(date +%Y%m%d).log
sha256sum evidence_*.log > evidence.sha256
cat evidence.sha256
```

Evidence : <img width="841" height="172" alt="Image" src="https://github.com/user-attachments/assets/535e96ba-d8cc-4c4b-bdb7-81efa3f0c1b4" />

## Short-Answer Questions

## Short-Answer Questions

**Q1. What is the difference between a log and an event? Give an example of each from this lab.**
- **Log:** A log is basically a historical diary file that records things chronologically as they happen. For example, the `auth.log` file storing records like `LOGIN_FAIL` when someone tries to access the system[cite: 1].
- **Event:** An event is a real-time trigger or alert that fires when specific conditions are met. For example, the alert message `'ALERT: probable brute-force -> compromise -> data exfiltration'` that pops up after the correlation script runs[cite: 1].

**Q2. Why must audit logs be tamper-proof, and how does a hash chain achieve this?**
- Audit logs need to be tamper-proof because if a hacker successfully compromises a system, the very first thing they'll want to do is wipe their footprints or alter the logs to avoid getting caught[cite: 1].
- A hash chain prevents this by mathematically linking every single log line's integrity value to the cryptographic hash of the previous record. So, if a hacker changes even a single character in the middle, all the subsequent hashes break and change, making it super obvious that someone messed with the logs[cite: 1].

**Q3. How did correlation detect an incident that no single log line revealed?**
- If you look at individual log lines by themselves, they just look like normal, routine stuff—a failed password here, a successful login there, or a large file transfer. But when you combine and correlate all those events into a single timeline (brute-force -> successful login -> sudden 500MB data export), it reveals the actual malicious pattern the attacker was pulling off[cite: 1].

**Q4. List the incident-response steps you performed and the goal of each.**
- **Detect:** Spotting anomalies or threat indicators from the log data[cite: 1].
- **Contain:** Halting the attacker's malicious activity right away by blocking their IP address using a firewall rule (`iptables`)[cite: 1].
- **Collect Evidence:** Making a safe, immutable copy of the evidence logs and locking them down with a cryptographic hash so they can't be tampered with[cite: 1].
- **Document:** Writing an incident report to record the timeline, findings, and lessons learned so it doesn't happen again[cite: 1].

**Q5. How do the same logs serve both security monitoring and compliance evidence?**
- For security monitoring, we use those logs in real-time to spot active threats and catch attacks early. Meanwhile, for compliance, those exact same logs act as unalterable, rock-solid historical proof required by auditors to show that our system is fully accountable and compliant with security standards[cite: 1].

## Security Best-Practices Checklist

| ✅ | Item | Status |
|----|------|--------|
| ☑ | Logs are centralised, not left scattered on each host | ✅ Done |
| ☑ | Security-relevant activity (failed logins) can be queried | ✅ Done |
| ☑ | Logs are tamper-evident (hash chain) and forwarded to a separate store | ✅ Done |
| ☑ | An incident is detected by correlating multiple events | ✅ Done |
| ☑ | Incident response performed: contain, collect evidence, document | ✅ Done |

## References

1. **Course Lecture** - Week 6 (Monitoring, Auditing & Management); Weeks 10-11 (Compliance Evidence)
2. **Amazon CloudWatch Logs** - docs.aws.amazon.com/AmazonCloudWatch/latest/logs
3. **OWASP Logging Cheat Sheet** - cheatsheetseries.owasp.org
4. **CSA Security Guidance v5** - Security Monitoring Domain
5. **LocalStack Documentation** - docs.localstack.cloud



































































