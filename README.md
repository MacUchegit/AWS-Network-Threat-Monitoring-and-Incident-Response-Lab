# AWS Network Threat Monitoring and Incident Response

**A hands-on cloud security portfolio project by Onyekwere McDonald**

This project documents how I designed, built, tested and investigated a network threat-monitoring environment for **Northstar Payments**, a fictional payment company. The lab separates a public security operations host from a private production workload, records traffic that reaches the production server, detects suspicious network behavior and supports a practical incident-response process.

The project was built in `eu-west-2` using simulated data and infrastructure that I owned. No real customer information, malware or third-party systems were used.

<img width="2600" height="1500" alt="image" src="https://github.com/user-attachments/assets/57ef9b14-a060-4f9b-ac88-391fc946482d" />

## Project snapshot

| Item | Details |
|---|---|
| Business problem | The production workload had preventive controls, but the security team lacked centralized evidence for accepted traffic, rejected traffic, reconnaissance, unusual transfers and security-group changes. |
| My role | Junior Cloud Security Engineer responsible for design, implementation, testing, investigation, containment, recovery and documentation. |
| Production asset | Private Amazon EC2 instance running a simulated internal payment service on TCP 8080. |
| Administrative path | EC2 Instance Connect to the public analyst host, followed by a temporary-key SSH connection to the production private IP across VPC peering. |
| Primary telemetry | VPC Flow Logs enabled on the production network interface and delivered to Amazon CloudWatch Logs. |
| Detection path | CloudWatch Logs Insights, a rejected-packet metric filter, a CloudWatch alarm, a dashboard and Amazon SNS email notification. |
| Change evidence | AWS CloudTrail Event History for security-group API activity. |
| Tested scenarios | Rejected connection spike, controlled port sweep, unusually large internal transfer, unsafe security-group rule and routing failure. |

## Summary

I built a two-VPC AWS security monitoring lab that keeps the production server private while giving an analyst a controlled administration path. I applied least-privilege security-group rules, used IAM roles and short-lived EC2 Instance Connect keys instead of stored access keys, centralized production ENI Flow Logs in CloudWatch, created queries and an alert for rejected traffic, built a monitoring dashboard and investigated five controlled security scenarios. I also diagnosed a real IAM authorization failure, remediated it with a resource-scoped policy and documented the full recovery process.

## What I implemented

- A dedicated **Security Operations VPC** with a public analyst subnet, internet gateway and route table.
- A separate **Production VPC** with a private application subnet, no internet gateway and no public IPv4 address on the workload.
- **VPC peering** with explicit bidirectional routes for private communication.
- **Security-group referencing across the active same-Region peering connection**, limiting production access to the analyst security group.
- Two Amazon Linux 2023 EC2 instances: `ec2-secops-analyst` and `ec2-prod-payment-app`.
- Browser-based EC2 Instance Connect for the analyst host and `SendSSHPublicKey` for short-lived access to the production host.
- An EC2 IAM role that permits the analyst host to push a temporary SSH key only to the production instance for the `ec2-user` account.
- A production ENI Flow Log that sends `ACCEPT` and `REJECT` metadata to CloudWatch Logs.
- CloudWatch Logs Insights queries for baseline traffic, rejected sources, targeted ports, reconnaissance and large transfers.
- A metric filter that converts rejected packet values into the custom `RejectedPackets` metric.
- A CloudWatch alarm that notifies an SNS email subscription when rejected packets cross the test threshold.
- A CloudWatch dashboard for traffic, ports, sources, transfer volume, alarm state and instance health.
- CloudTrail Event History analysis for `AuthorizeSecurityGroupIngress` activity.
- Controlled simulations followed by triage, containment, recovery and lessons learned.

## Architecture walk-through

1. The analyst opens an EC2 Instance Connect browser session to `ec2-secops-analyst`. Inbound SSH is restricted to the AWS-managed EC2 Instance Connect prefix list for `eu-west-2`.
2. `NorthstarSecOpsAnalystRole` supplies temporary AWS credentials to the analyst instance. No IAM user access key is stored on the server.
3. The analyst generates a temporary SSH key and calls `ec2-instance-connect:SendSSHPublicKey` for `ec2-prod-payment-app`. The key is available for the short connection window, then SSH travels to `10.10.1.123` through VPC peering.
4. The production security group accepts SSH 22, application traffic 8080 and ICMP only from `secops-analyst-sg`.
5. The production instance's primary ENI sends VPC Flow Log metadata to `/aws/vpc/prod-payments/flowlogs` in CloudWatch Logs.
6. Logs Insights supports investigations. A metric filter extracts rejected packet counts, the alarm evaluates the metric, SNS sends the email alert and the dashboard provides a shared monitoring view.
7. CloudTrail Event History identifies the actor, source IP, time, API action and rule details for a production security-group change.

## AWS resources used

| Layer | Implemented resources |
|---|---|
| Network | 2 VPCs, 2 subnets, 2 custom route tables, 1 internet gateway and 1 VPC peering connection |
| Network controls | `secops-analyst-sg` and `prod-payment-app-sg` |
| Compute | 2 Amazon EC2 instances using Amazon Linux 2023 and 8 GiB gp3 root volumes |
| Secure access | EC2 Instance Connect and private SSH over VPC peering |
| Identity | `NorthstarSecOpsAnalystRole`, `NorthstarSecOpsEICPolicy`, `NorthstarVPCFlowLogsRole` and `NorthstarVPCFlowLogsPolicy` |
| Network telemetry | VPC Flow Logs on the production ENI |
| Monitoring | CloudWatch Logs, Logs Insights, metric filter, custom metric, alarm and dashboard |
| Notification | SNS standard topic and confirmed email subscription |
| Audit evidence | CloudTrail Event History |

## Addressing and trust boundaries

| Component | Name | Address or CIDR | Security purpose |
|---|---|---|---|
| SecOps VPC | `vpc-secops` | `10.20.0.0/16` | Controlled administration and test source |
| Public subnet | `subnet-secops-public-a` | `10.20.1.0/24` | Hosts the analyst instance and routes to the internet gateway |
| Analyst EC2 | `ec2-secops-analyst` | `10.20.1.195` private IP plus public IPv4 | Approved administration host and controlled traffic generator |
| Production VPC | `vpc-prod-payments` | `10.10.0.0/16` | Isolated workload boundary |
| Private subnet | `subnet-prod-app-private-a` | `10.10.1.0/24` | Hosts the private payment application |
| Production EC2 | `ec2-prod-payment-app` | `10.10.1.123` private IP only | Simulated internal service on TCP 8080 |

## Security design decisions

### Private production workload

The production VPC has no internet gateway, and the production instance has no public IPv4 address. This removes direct internet administration and makes the SecOps path explicit.

### Least-privilege network rules

The production security group references `secops-analyst-sg` across the active same-Region peering connection. It does not accept a permanent `0.0.0.0/0` SSH rule.

| Direction | Protocol or service | Port | Source or destination | Reason |
|---|---|---:|---|---|
| Analyst inbound | SSH | 22 | Regional EC2 Instance Connect prefix list | Browser-based IAM-authorized access |
| Production inbound | SSH | 22 | `secops-analyst-sg` | Administration from the approved host |
| Production inbound | Custom TCP | 8080 | `secops-analyst-sg` | Internal test application |
| Production inbound | ICMP IPv4 | All | `secops-analyst-sg` | Controlled reachability tests |

### Temporary credentials instead of stored secrets

The analyst EC2 role supplies temporary credentials through the instance metadata service. The role can send an SSH public key only to the production instance, and the policy restricts the operating-system user to `ec2-user`.

### Focused telemetry

Flow logging is scoped to the production ENI rather than the entire VPC. This captures the evidence needed for the investigation while limiting log volume. The log group has a three-day retention period, and Logs Insights queries use narrow time windows.

## Threat model and validation plan

| Threat or failure | Expected evidence | Response tested |
|---|---|---|
| Repeated blocked connection attempts | `REJECT` Flow Log records and rejected-packet alarm | Identify source, destination, targeted port and packet volume |
| Port reconnaissance | One source contacting multiple destination ports | Confirm lab authorization and stop the test after evidence is collected |
| Unusually large internal transfer | Accepted flow with bytes well above baseline | Compare direction and volume; do not claim payload visibility |
| Unsafe production security-group rule | CloudTrail `AuthorizeSecurityGroupIngress` event | Attribute the change and remove the rule immediately |
| Missing VPC peering route | Failed connectivity and possibly no record at the production ENI | Inspect routing, restore the route and validate recovery |

## Implementation and evidence

All screenshot blocks below are deliberate placeholders. Replace each block with the matching redacted image before publishing. Recommended image size is at least 1600 px wide.

### Phase 1: Build the network foundation

I created the two VPCs, placed the subnets in the same Availability Zone, attached an internet gateway only to the SecOps VPC and configured separate route tables.

> **Screenshot 01 placeholder | Security Operations VPC**  
> File: `evidence/screenshots/01-secops-vpc.png`  
> Show: VPC name, VPC ID, `10.20.0.0/16` CIDR and DNS settings.

> **Screenshot 02 placeholder | Production VPC**  
> File: `evidence/screenshots/02-production-vpc.png`  
> Show: VPC name, VPC ID, `10.10.0.0/16` CIDR and DNS settings.

> **Screenshot 03 placeholder | SecOps public subnet**  
> File: `evidence/screenshots/03-secops-public-subnet.png`  
> Show: subnet name, VPC, Availability Zone, `10.20.1.0/24` and public IPv4 mapping enabled.

> **Screenshot 04 placeholder | Production private subnet**  
> File: `evidence/screenshots/04-production-private-subnet.png`  
> Show: subnet name, VPC, Availability Zone, `10.10.1.0/24` and public IPv4 mapping disabled.

> **Screenshot 05 placeholder | SecOps internet gateway**  
> File: `evidence/screenshots/05-secops-internet-gateway.png`  
> Show: `igw-secops` attached to `vpc-secops`.

> **Screenshot 06 placeholder | SecOps public route table**  
> File: `evidence/screenshots/06-secops-route-table.png`  
> Show: subnet association, local route and `0.0.0.0/0` route to `igw-secops`.

> **Screenshot 07 placeholder | Production route table before peering**  
> File: `evidence/screenshots/07-production-route-table.png`  
> Show: production subnet association and local route only.

### Phase 2: Configure VPC peering and private routing

I created `pcx-secops-to-prod`, accepted the request and added routes for the peer CIDR to each route table. VPC peering is non-transitive, so only the two directly connected VPCs are covered.

> **Screenshot 08 placeholder | Active VPC peering connection**  
> File: `evidence/screenshots/08-vpc-peering-active.png`  
> Show: connection name, requester, accepter and `Active` state.

> **Screenshot 09 placeholder | SecOps route to production**  
> File: `evidence/screenshots/09-secops-peering-route.png`  
> Show: `10.10.0.0/16` routed to the peering connection.

> **Screenshot 10 placeholder | Production route to SecOps**  
> File: `evidence/screenshots/10-production-peering-route.png`  
> Show: `10.20.0.0/16` routed to the same peering connection.

### Phase 3: Enforce least-privilege security groups

The analyst host accepts SSH only from the regional EC2 Instance Connect prefix list. The production host accepts SSH, TCP 8080 and ICMP only from the analyst security group.

> **Screenshot 11 placeholder | Least-privilege security groups**  
> File: `evidence/screenshots/11-least-privilege-security-groups.png`  
> Show: the EIC prefix-list rule and production rules sourced from `secops-analyst-sg`.

### Phase 4: Launch EC2 and implement temporary access

Both instances use Amazon Linux 2023 and an 8 GiB gp3 root volume. The analyst instance is public; the production instance is private. The production user data creates a small HTTP service without downloading extra packages.

```bash
#!/bin/bash
set -e

mkdir -p /opt/northstar-app

cat > /opt/northstar-app/index.html <<'EOF'
<!doctype html>
<html>
  <head><title>Northstar Payments</title></head>
  <body>
    <h1>Northstar Payments Internal Service</h1>
    <p>Status: healthy</p>
    <p>This server contains simulated portfolio data only.</p>
  </body>
</html>
EOF

cat > /etc/systemd/system/northstar-app.service <<'EOF'
[Unit]
Description=Northstar Payments Python HTTP Service
After=network.target

[Service]
Type=simple
WorkingDirectory=/opt/northstar-app
ExecStart=/usr/bin/python3 -m http.server 8080 --bind 0.0.0.0
Restart=on-failure

[Install]
WantedBy=multi-user.target
EOF

systemctl daemon-reload
systemctl enable --now northstar-app.service
```

The analyst role uses the following resource-scoped permission. Replace every angle-bracket value before deployment.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "SendTemporaryKeyToProductionInstance",
      "Effect": "Allow",
      "Action": "ec2-instance-connect:SendSSHPublicKey",
      "Resource": "arn:aws:ec2:eu-west-2:026651348779:instance/i-08151edd74b70b199",
      "Condition": {
        "StringEquals": {
          "ec2:osuser": "ec2-user"
        }
      }
    },
    {
      "Sid": "DescribeConnectionContext",
      "Effect": "Allow",
      "Action": [
        "ec2:DescribeInstances",
        "ec2:DescribeAvailabilityZones"
      ],
      "Resource": "*"
    }
  ]
}
```

From the EC2 Instance Connect browser session on the analyst host:

```bash
export AWS_REGION=eu-west-2
export PROD_INSTANCE_ID=i-08151edd74b70b199
export PROD_AZ=eu-west-2a
export PROD_IP=10.10.1.123

mkdir -p "$HOME/.ssh"
chmod 700 "$HOME/.ssh"
KEY="$HOME/.ssh/northstar-prod-eic"

ssh-keygen -t ed25519 -f "$KEY" -N ""

aws ec2-instance-connect send-ssh-public-key \
  --region "$AWS_REGION" \
  --instance-id "$PROD_INSTANCE_ID" \
  --instance-os-user ec2-user \
  --availability-zone "$PROD_AZ" \
  --ssh-public-key "file://${KEY}.pub"

ssh -o StrictHostKeyChecking=accept-new -i "$KEY" ec2-user@"$PROD_IP"
```

> **Screenshot 12 placeholder | Running EC2 instances**  
> File: `evidence/screenshots/12-running-ec2-instances.png`  
> Show: both names, states, instance types, subnets and Availability Zone.

> **Screenshot 13 placeholder | Analyst IAM role attached**  
> File: `evidence/screenshots/13-analyst-iam-role.png`  
> Show: `NorthstarSecOpsAnalystRole` attached to `ec2-secops-analyst`.

> **Screenshot 14 placeholder | Production instance has no public IPv4**  
> File: `evidence/screenshots/14-production-no-public-ip.png`  
> Show: the private IP and an empty public IPv4 field.

> **Screenshot 15 placeholder | EC2 Instance Connect access denied**  
> File: `evidence/screenshots/15-eic-access-denied.png`  
> Show: the `AccessDeniedException` for `ec2-instance-connect:SendSSHPublicKey`.

> **Screenshot 16 placeholder | IAM policy remediation**  
> File: `evidence/screenshots/16-eic-policy-remediation.png`  
> Show: the resource-scoped EIC permission and `ec2:osuser` condition.

> **Screenshot 17 placeholder | Private connectivity restored**  
> File: `evidence/screenshots/17-private-connectivity-restored.png`  
> Show: assumed-role identity, temporary-key SSH, ping replies and HTTP response by private IP.

### Phase 5: Centralize production VPC Flow Logs

I created `/aws/vpc/prod-payments/flowlogs`, set a three-day retention period and created a service role trusted by `vpc-flow-logs.amazonaws.com`. The flow log captures all traffic for the production ENI with a one-minute maximum aggregation interval and the AWS default record format.

```text
version account-id interface-id srcaddr dstaddr srcport dstport protocol packets bytes start end action log-status
```

The Flow Logs delivery policy uses a log-group-scoped ARN:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "WriteToNorthstarFlowLogGroup",
      "Effect": "Allow",
      "Action": [
        "logs:CreateLogStream",
        "logs:PutLogEvents"
      ],
      "Resource": "arn:aws:ec2:eu-west-2:026651348779:instance/i-08151edd74b70b199:*"
    },
    {
      "Sid": "DescribeCloudWatchLogs",
      "Effect": "Allow",
      "Action": [
        "logs:DescribeLogGroups",
        "logs:DescribeLogStreams"
      ],
      "Resource": "*"
    }
  ]
}
```

> **Screenshot 18 placeholder | CloudWatch log group**  
> File: `evidence/screenshots/18-cloudwatch-log-group.png`  
> Show: log-group name, Region, Standard log class and three-day retention.

> **Screenshot 19 placeholder | Flow Logs IAM policy**  
> File: `evidence/screenshots/19-flow-logs-iam-policy.png`  
> Show: log-write permissions and the scoped log-group ARN.

> **Screenshot 20 placeholder | Flow Logs role trust policy**  
> File: `evidence/screenshots/20-flow-logs-trust-policy.png`  
> Show: `NorthstarVPCFlowLogsRole`, attached policy and trusted service principal.

> **Screenshot 21 placeholder | Production ENI Flow Log**  
> File: `evidence/screenshots/21-production-eni-flow-log.png`  
> Show: ENI scope, ALL traffic, one-minute interval, destination log group and service role.

> **Screenshot 22 placeholder | First Flow Log events**  
> File: `evidence/screenshots/22-first-flow-log-events.png`  
> Show: at least one `ACCEPT` and one `REJECT` record for the production ENI.

### Phase 6: Establish a normal traffic baseline

I generated approved SSH, HTTP and ICMP traffic, then recorded the normal sources, destinations, ports, packet counts and byte ranges.

```sql
fields @timestamp, srcAddr, dstAddr, srcPort, dstPort,
       protocol, packets, bytes, action
| filter srcAddr = "10.20.1.195" or dstAddr = "10.20.1.195"
| sort @timestamp desc
| limit 100
```

> **Screenshot 23 placeholder | Normal traffic baseline**  
> File: `evidence/screenshots/23-normal-traffic-baseline.png`  
> Show: approved source, destination, ports, action and typical transfer volume.

### Phase 7: Alert on rejected traffic

I created the SNS topic `secops-network-alerts`, confirmed the email subscription and configured the metric filter below.

```text
[version, account, interfaceId, srcAddr, dstAddr, srcPort, dstPort,
 protocol, packets, bytes, startTime, endTime, action="REJECT", logStatus]
```

| Alarm setting | Value |
|---|---|
| Metric | `Northstar/Security / RejectedPackets` |
| Metric value | `$packets` |
| Statistic | Sum |
| Period | 5 minutes |
| Threshold | Greater than or equal to 20 |
| Datapoints | 1 of 1 |
| Missing data | Treat as not breaching |
| Action | Notify `secops-network-alerts` |

> **Screenshot 24 placeholder | SNS topic**  
> File: `evidence/screenshots/24-sns-topic.png`  
> Show: topic name and type.

> **Screenshot 25 placeholder | Confirmed SNS subscription**  
> File: `evidence/screenshots/25-sns-subscription.png`  
> Show: confirmed status. Redact the email address and ARN details.

> **Screenshot 26 placeholder | Rejected-packet metric filter**  
> File: `evidence/screenshots/26-rejected-packets-metric-filter.png`  
> Show: complete filter pattern, namespace, metric name and `$packets` value.

> **Screenshot 27 placeholder | CloudWatch alarm conditions**  
> File: `evidence/screenshots/27-cloudwatch-alarm-conditions.png`  
> Show: statistic, period, threshold, datapoints and missing-data behavior.

> **Screenshot 28 placeholder | CloudWatch alarm action**  
> File: `evidence/screenshots/28-cloudwatch-alarm-actions.png`  
> Show: alarm name, description and SNS notification action.

### Phase 8: Build the monitoring dashboard

The `Northstar-Production-Network-Security` dashboard answers four operational questions: Is rejected traffic increasing? Which source is responsible? Which ports are targeted? Did an unusual transfer or configuration change occur near the same time?

```sql
# Rejected packets over time
filter action = "REJECT"
| stats sum(packets) as rejectedPackets by bin(5m)

# Top rejected sources
filter action = "REJECT"
| stats sum(packets) as rejectedPackets by srcAddr
| sort rejectedPackets desc
| limit 10

# Destination ports targeted
filter action = "REJECT"
| stats sum(packets) as rejectedPackets by dstPort
| sort rejectedPackets desc
| limit 10

# Largest accepted transfers
filter action = "ACCEPT"
| stats sum(bytes) as bytesTransferred,
        sum(packets) as packetsTransferred
  by srcAddr, dstAddr, dstPort
| sort bytesTransferred desc
| limit 20
```

> **Screenshot 29 placeholder | Security monitoring dashboard**  
> File: `evidence/screenshots/29-security-dashboard.png`  
> Show: traffic, source, port, transfer, alarm-state and EC2 health widgets.

### Phase 9: Simulate and investigate security events

> [!CAUTION]
> Run these tests only against private IP addresses and AWS resources that you own. The objective is detection validation, not exploitation.

#### Scenario 1: Rejected connection spike

```bash
export PROD_IP=10.10.1.123

for i in $(seq 1 50); do
  timeout 1 bash -c "echo >/dev/tcp/$PROD_IP/3389" 2>/dev/null || true
done
```

```sql
fields @timestamp, srcAddr, dstAddr, srcPort, dstPort,
       protocol, packets, bytes, action
| filter action = "REJECT"
| sort @timestamp desc
| limit 100
```

> **Screenshot 30 placeholder | Rejected-traffic simulation**  
> File: `evidence/screenshots/30-rejected-traffic-simulation.png`  
> Show: the controlled loop and production private IP.

> **Screenshot 31 placeholder | Rejected-traffic evidence**  
> File: `evidence/screenshots/31-rejected-traffic-logs.png`  
> Show: source, destination, port 3389, packet count, `REJECT` action and test time window.

#### Scenario 2: Controlled port sweep

```bash
export PROD_IP=10.10.1.123

for port in 21 22 23 25 80 443 3306 3389 5432 8080; do
  timeout 1 bash -c "echo >/dev/tcp/$PROD_IP/$port" 2>/dev/null || true
done
```

```sql
filter action = "REJECT"
| stats count_distinct(dstPort) as uniquePorts,
        sum(packets) as rejectedPackets,
        count(*) as flowRecords
  by srcAddr, dstAddr
| filter uniquePorts >= 8
| sort uniquePorts desc
```

> **Screenshot 32 placeholder | Port-sweep detection**  
> File: `evidence/screenshots/32-port-sweep-detection.png`  
> Show: source, destination, unique-port count, rejected packets and query time range.

#### Scenario 3: Abnormally large internal transfer

I compared a harmless 1 MiB object with a 20 MiB object served by the private application. The larger flow should stand out from the normal baseline.

```bash
sudo dd if=/dev/zero of=/opt/northstar-app/baseline-1mb.bin \
  bs=1M count=1 status=none

sudo dd if=/dev/zero of=/opt/northstar-app/unusual-20mb.bin \
  bs=1M count=20 status=none
```

```bash
curl -s -o /dev/null -w "baseline: %{size_download} bytes\n" \
  "http://10.10.1.123:8080/baseline-1mb.bin"

curl -s -o /dev/null -w "unusual: %{size_download} bytes\n" \
  "http://10.10.1.123:8080/unusual-20mb.bin"
```

> **Screenshot 33 placeholder | Abnormal-transfer simulation**  
> File: `evidence/screenshots/33-abnormal-transfer-simulation.png`  
> Show: commands and the reported 1 MiB and 20 MiB download sizes.

> **Screenshot 34 placeholder | Abnormal-transfer evidence**  
> File: `evidence/screenshots/34-abnormal-transfer-logs.png`  
> Show: source, destination, port and the larger accepted byte total.

#### Scenario 4: Unsafe security-group change

I temporarily added an inbound SSH rule from `0.0.0.0/0` to the production security group, captured the event in CloudTrail Event History and removed the rule immediately. The production VPC still had no internet gateway, so this test generated control-plane evidence without creating an internet path to the private instance.

> **Screenshot 35 placeholder | Temporary unsafe security-group rule**  
> File: `evidence/screenshots/35-unsafe-security-group-rule.png`  
> Show: the temporary TCP 22 rule from `0.0.0.0/0` before removal.

> **Screenshot 36 placeholder | CloudTrail change event**  
> File: `evidence/screenshots/36-cloudtrail-change-event.png`  
> Show: event time, identity, source IP, Region, security-group ID, port, CIDR and request elements.

#### Scenario 5: Routing misconfiguration and recovery

I temporarily removed the `10.10.0.0/16` peering route from `rtb-secops-public`, repeated the approved ping and HTTP tests, restored the route and verified recovery. Because the packet could not leave the source subnet correctly, it might never reach the production ENI. The absence of a production Flow Log record therefore supported the routing hypothesis.

> **Screenshot 37 placeholder | Peering route removed**  
> File: `evidence/screenshots/37-peering-route-removed.png`  
> Show: the SecOps route table without the production CIDR route.

> **Screenshot 38 placeholder | Connectivity failure**  
> File: `evidence/screenshots/38-connectivity-failure.png`  
> Show: failed ping or HTTP test after route removal.

> **Screenshot 39 placeholder | Peering route restored**  
> File: `evidence/screenshots/39-peering-route-restored.png`  
> Show: `10.10.0.0/16` restored to the peering target.

> **Screenshot 40 placeholder | Connectivity recovered**  
> File: `evidence/screenshots/40-connectivity-recovered.png`  
> Show: successful private-IP ping and HTTP response after restoration.

## Troubleshooting case: EC2 Instance Connect authorization failure

The first attempt to push the temporary SSH key failed with `AccessDeniedException`. SSH later returned `Permission denied (publickey)` because the key had never been delivered.

My troubleshooting separated the network path from the identity problem:

1. The SSH client reached the production host and recorded its host key, which suggested that peering, routes, the production security group and TCP 22 were working.
2. The AWS API response explicitly stated that no identity-based policy allowed `ec2-instance-connect:SendSSHPublicKey`.
3. I added a least-privilege permission to `NorthstarSecOpsAnalystRole`, scoped it to the production instance ARN and restricted `ec2:osuser` to `ec2-user`.
4. I retried the API call and opened SSH immediately within the temporary-key window.
5. I validated the final path with role identity, SSH, ping and HTTP evidence.

This was the most useful failure in the project because it required evidence-based diagnosis rather than random configuration changes.

## Incident-response workflow

| Stage | Work performed |
|---|---|
| Detect | Reviewed the alarm, SNS notification and dashboard for a rejected-traffic spike. |
| Triage | Identified source, destination, action, ports, packets, bytes and the test time window in Logs Insights. |
| Correlate | Compared network activity with CloudTrail Event History when a security-group change was involved. |
| Contain | Stopped the controlled traffic, removed the unsafe SSH rule and restored the approved route. |
| Recover | Re-ran role identity, temporary-key SSH, ICMP and HTTP checks. |
| Document | Recorded findings, limitations, root cause, remediation and prevention recommendations. |

## Results

- The production EC2 instance remained private and had no public IPv4 address.
- Approved administration and application traffic crossed the peering connection using private IP addresses.
- Unapproved traffic generated `REJECT` Flow Log records.
- The rejected-packet metric and alarm converted log evidence into an actionable notification path.
- Logs Insights identified targeted ports, repeated rejected sources and unusually large accepted transfers.
- CloudTrail Event History attributed the unsafe security-group change to an identity and API action.
- The IAM failure and route failure were diagnosed, fixed and retested.
- All test data was simulated, and every unsafe change was temporary and reversed.

## Important limitations

- VPC Flow Logs record metadata, not packet payloads. A high byte count can indicate unusual movement, but it does not prove data exfiltration or reveal file contents.
- Flow Log delivery is best effort and not instantaneous. This lab is not a real-time packet inspection system.
- CloudTrail Event History covers recent management events in the selected Region. Long-term centralized retention would require an additional design that was not implemented here.
- The lab uses one account, one Region and one Availability Zone. It is intended to demonstrate security reasoning, not production high availability.
- The threshold of 20 rejected packets in five minutes was chosen for controlled validation. A production threshold would require a longer baseline and tuning for the workload.

## Cost controls

- Used micro EC2 instance types and 8 GiB gp3 root volumes.
- Scoped Flow Logs to one production ENI.
- Set CloudWatch Logs retention to three days.
- Used narrow Logs Insights time windows and a dashboard refresh interval of at least five minutes.
- Avoided NAT Gateway, AWS Network Firewall and other services that were unnecessary for this lab.
- Removed the environment after evidence collection.

AWS account offers and pricing change over time. Check the current price of every resource in your own Region before reproducing the lab.

## Cleanup order

1. Delete the CloudWatch alarm, dashboard and metric filter.
2. Delete the SNS subscription and topic.
3. Delete the production ENI Flow Log, then delete the CloudWatch log group after exporting any approved evidence.
4. Terminate both EC2 instances and confirm that their root volumes are deleted.
5. Detach and delete the project IAM roles and customer-managed policies.
6. Delete the two security groups after the instances are gone.
7. Remove peering routes, then delete the VPC peering connection.
8. Detach and delete `igw-secops`.
9. Delete the custom route tables, subnets and VPCs.
10. Review billing and resource inventory for anything left behind.

## Recommended repository structure

```text
aws-network-threat-monitoring/
├── README.md
├── architecture/
│   ├── aws-network-threat-monitoring-architecture.png
│   └── aws-network-threat-monitoring-architecture.svg
├── docs/
│   └── AWS-Network-Threat-Monitoring-Portfolio.docx
├── evidence/
│   └── screenshots/
│       ├── 01-secops-vpc.png
│       ├── ...
│       └── 40-connectivity-recovered.png
├── policies/
│   ├── analyst-eic-policy.json
│   ├── analyst-role-trust-policy.json
│   ├── flow-logs-policy.json
│   └── flow-logs-role-trust-policy.json
├── queries/
│   └── cloudwatch-logs-insights.md
└── LICENSE
```

## Skills demonstrated

- AWS network architecture and segmentation
- VPC, subnet, route-table, internet-gateway and VPC-peering configuration
- Security-group design and same-Region peer security-group referencing
- Amazon EC2 and Amazon Linux 2023 administration
- EC2 Instance Connect and temporary SSH access
- IAM roles, trust policies, resource scoping and condition keys
- VPC Flow Logs and CloudWatch Logs
- CloudWatch Logs Insights query development
- Metrics, alarms, dashboards and SNS notifications
- CloudTrail management-event analysis
- Threat simulation, incident triage, containment and recovery
- Root-cause analysis, technical documentation, evidence handling and redaction
- Cost awareness and environment cleanup

## Resume project entry

**AWS Network Threat Monitoring and Incident Response Lab**

- Designed a two-VPC AWS security monitoring environment with a public SecOps analyst host and a private production workload connected through VPC peering.
- Enforced least-privilege administration using security-group references, EC2 Instance Connect, temporary SSH keys and resource-scoped IAM permissions.
- Centralized production ENI Flow Logs in CloudWatch, built detection queries, created a rejected-packet alarm and SNS notification path, and investigated five controlled security scenarios.
- Diagnosed an IAM authorization failure and a routing failure, implemented corrective actions and validated recovery with network, identity and application evidence.

## Evidence handling before publication

Redact AWS account IDs, IAM principal IDs, ARNs where unnecessary, public IP addresses, email addresses, billing information and console-session details. Private RFC 1918 lab addresses may remain if they help explain the architecture, but confirm that every screenshot is safe before committing it.

## Technical references

- [Connect to a Linux instance using EC2 Instance Connect](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/ec2-instance-connect-methods.html)
- [EC2 Instance Connect `send-ssh-public-key` command](https://docs.aws.amazon.com/cli/latest/reference/ec2-instance-connect/send-ssh-public-key.html)
- [How VPC peering connections work](https://docs.aws.amazon.com/vpc/latest/peering/vpc-peering-basics.html)
- [Reference security groups across a peering connection](https://docs.aws.amazon.com/vpc/latest/peering/vpc-peering-security-groups.html)
- [VPC Flow Logs](https://docs.aws.amazon.com/vpc/latest/userguide/flow-logs.html)
- [Publish VPC Flow Logs to CloudWatch Logs](https://docs.aws.amazon.com/vpc/latest/userguide/flow-logs-cwl.html)
- [IAM role for publishing Flow Logs to CloudWatch Logs](https://docs.aws.amazon.com/vpc/latest/userguide/flow-logs-iam-role.html)
- [CloudTrail Event History](https://docs.aws.amazon.com/awscloudtrail/latest/userguide/view-cloudtrail-events.html)
- [AWS Architecture Icons](https://aws.amazon.com/architecture/icons/)


