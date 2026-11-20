# AWS Threat Detection & Automated Security Alerting

## Project Overview

This project demonstrates how to build an AWS security monitoring and automated alerting solution using Amazon CloudTrail, Amazon GuardDuty, Amazon CloudWatch, Amazon EventBridge, and Amazon SNS.

The solution monitors AWS account activity, detects suspicious behavior and potentially unauthorized API activity, and automatically sends email notifications when security events or configured alarm thresholds are triggered.

The project was implemented using the AWS Management Console and AWS CLI through Windows PowerShell.

## Objectives

- Enable AWS CloudTrail to record AWS account activity.
- Configure Amazon GuardDuty to detect potential security threats.
- Integrate CloudTrail with Amazon CloudWatch Logs.
- Create CloudWatch metric filters to identify specific security-related events.
- Configure CloudWatch alarms for security-related events.
- Set up Amazon SNS for automated email notifications.
- Configure Amazon EventBridge to forward GuardDuty findings to SNS.
- Validate the end-to-end security monitoring and alerting workflows.

## AWS Services Used

| AWS Service | Purpose |
|---|---|
| AWS CloudTrail | Records AWS account API activity and management events |
| Amazon S3 | Stores CloudTrail log files |
| Amazon GuardDuty | Detects potential security threats and suspicious activity |
| Amazon CloudWatch Logs | Centralizes and stores CloudTrail logs |
| Amazon CloudWatch Metric Filters | Identifies specific security-related events |
| Amazon CloudWatch Alarms | Monitors security metrics and triggers alerts |
| Amazon EventBridge | Routes GuardDuty findings to SNS |
| Amazon SNS | Delivers automated security notifications through email |
| AWS IAM | Manages permissions and service access |

## Network Architecture

The solution consists of two independent security monitoring workflows that converge on a central Amazon SNS topic for email notifications.

### Architecture Diagram

```mermaid
flowchart TB
    subgraph W1["Workflow 1: AWS Account Activity Monitoring"]
        CT["AWS CloudTrail<br/>Multi-Region Trail"]
        S3[("Amazon S3<br/>CloudTrail Log Storage")]
        CWL["Amazon CloudWatch Logs<br/>/aws/cloudtrail/threat-detection"]
        MF["CloudWatch Metric Filters<br/>ConsoleLoginFailures<br/>UnauthorizedApiCalls<br/>IamPolicyChanges"]
        AL["CloudWatch Alarms<br/>3 Security Alarms"]

        CT -->|"Log files"| S3
        CT -->|"Management events"| CWL
        CWL --> MF
        MF -->|"Metric data"| AL
    end

    subgraph W2["Workflow 2: GuardDuty Threat Detection"]
        GD["Amazon GuardDuty<br/>Threat Detection"]
        EB["Amazon EventBridge<br/>GuardDutyFindingsRule"]

        GD -->|"GuardDuty Finding"| EB
    end

    subgraph W3["Centralized Alerting"]
        SNS["Amazon SNS<br/>ThreatDetectionAlerts"]
        EMAIL["Email Notifications"]
    end

    AL -->|"Alarm actions"| SNS
    EB -->|"Event target"| SNS
    SNS -->|"Email subscription"| EMAIL

    classDef monitoring fill:#EAF3FF,stroke:#2874A6,color:#12344D
    classDef alert fill:#FFF0D9,stroke:#C47A16,color:#593600
    classDef storage fill:#E9F7EF,stroke:#29945B,color:#174A2E

    class CT,CWL,MF,GD,EB monitoring
    class AL,SNS,EMAIL alert
    class S3 storage
```

### Workflow 1 – CloudTrail and CloudWatch Monitoring

1. AWS CloudTrail records AWS account management events.
2. CloudTrail delivers log files to Amazon S3 and management event records to CloudWatch Logs.
3. CloudWatch Logs stores the CloudTrail events.
4. Metric filters identify failed console logins, unauthorized API calls, and IAM policy changes.
5. CloudWatch alarms monitor the resulting metric data.
6. When an alarm threshold is breached, CloudWatch publishes a notification to the SNS topic.
7. Amazon SNS delivers the alert through email.

### Workflow 2 – GuardDuty Threat Detection

1. Amazon GuardDuty analyzes supported AWS data sources and generates security findings.
2. Amazon EventBridge captures findings matching the configured event pattern.
3. EventBridge forwards matching findings to the SNS topic.
4. Amazon SNS delivers the GuardDuty finding notification through email.

### Centralized Alerting

Both workflows use the same SNS topic:

`ThreatDetectionAlerts`

This provides a centralized email notification destination for CloudWatch alarms and GuardDuty findings.

## Implementation Details

### Exercise 1 – Configure AWS CloudTrail

- Created a multi-region CloudTrail trail named `aws-threat-detection-trail`.
- Configured an Amazon S3 bucket to store CloudTrail log files.
- Enabled management event logging for read and write operations.
- Enabled log file validation.
- Integrated CloudTrail with Amazon CloudWatch Logs.
- Verified that CloudTrail was actively logging and delivering events.

**Resources:**

- Trail: `aws-threat-detection-trail`
- S3 Bucket: `bryan-cloudtrail-logs-2026`
- CloudWatch Log Group: `/aws/cloudtrail/threat-detection`

**Screenshots:**

![CloudTrail trail configuration](screenshots/01-cloudtrail/cloudtrail-trail-configuration.png)

![CloudTrail logging status](screenshots/01-cloudtrail/cloudtrail-logging-status.png)

### Exercise 2 – Enable Amazon GuardDuty

- Enabled Amazon GuardDuty in the Singapore region.
- Generated sample security findings for testing.
- Retrieved and reviewed GuardDuty findings.
- Verified that GuardDuty was generating security findings successfully.

**Screenshots:**

![GuardDuty detector status](screenshots/02-guardduty/guardduty-detector-status.png)

![GuardDuty sample finding details](screenshots/02-guardduty/sample-finding-details.png)

> **Note:** The sample finding shown in the screenshot is simulated and was generated for testing purposes. It does not represent an actual security incident.

### Exercise 3 – Configure Amazon CloudWatch Logs

- Integrated CloudTrail with the CloudWatch Logs service.
- Verified the existence of the CloudTrail log group and log streams.
- Retrieved CloudTrail events using CloudWatch Logs.
- Confirmed that AWS management events were successfully delivered to CloudWatch Logs.

**Screenshots:**

![CloudWatch log group](screenshots/03-cloudwatch-logs/cloudwatch-log-group.png)

![CloudWatch log streams](screenshots/03-cloudwatch-logs/cloudwatch-log-streams.png)

![Retrieved CloudTrail log events](screenshots/03-cloudwatch-logs/cloudtrail-log-events.png)

### Exercise 4 – Configure CloudWatch Metric Filters and Alarms

Created three CloudWatch metric filters under the custom namespace `ThreatDetectionMetrics`.

| Metric Filter | Detection Purpose |
|---|---|
| `ConsoleLoginFailures` | Detects failed AWS Management Console login attempts |
| `UnauthorizedApiCalls` | Detects unauthorized or access-denied API requests |
| `IamPolicyChanges` | Detects IAM policy creation, modification, attachment, and deletion activities |

**Metric Filters Configuration:**

![All three CloudWatch metric filters](screenshots/04-metric-filters-alarms/metric-filters.png)

Configured corresponding CloudWatch alarms:

| Alarm Name | Metric | Evaluation Threshold |
|---|---|---|
| `Alarm-ConsoleLoginFailures` | ConsoleLoginFailures | Greater than or equal to 1 |
| `Alarm-UnauthorizedApiCalls` | UnauthorizedApiCalls | Greater than or equal to 1 |
| `Alarm-IamPolicyChanges` | IamPolicyChanges | Greater than or equal to 1 |

**Alarm Configuration:**

- Statistic: Sum
- Period: 300 seconds (5 minutes)
- Evaluation Periods: 1
- Missing Data Treatment: Not Breaching

**CloudWatch Alarms:**

![CloudWatch alarms overview](screenshots/04-metric-filters-alarms/cloudwatch-alarms.png)

**SNS Alarm Actions:**

All three CloudWatch alarms were configured to publish notifications to the `ThreatDetectionAlerts` SNS topic.

![CloudWatch alarm actions showing SNS topic ARN](screenshots/04-metric-filters-alarms/alarm-actions.png)

### Exercise 5 – Configure Amazon SNS Email Notifications

- Created an SNS topic named `ThreatDetectionAlerts`.
- Configured an email subscription.
- Confirmed the email subscription.
- Connected all three CloudWatch alarms to the SNS topic.
- Published a test notification to verify email delivery.

**SNS Topic:** `ThreatDetectionAlerts`

**Result:** Successfully received the SNS test notification.

**Screenshots:**

![SNS topic configuration](screenshots/05-sns/sns-topic-configuration.png)

![Confirmed SNS email subscription](screenshots/05-sns/sns-email-subscription.png)

![SNS test email notification](screenshots/05-sns/sns-test-notification.png)

### Exercise 6 – Configure Amazon EventBridge for GuardDuty

- Created an EventBridge rule named `GuardDutyFindingsRule`.
- Configured the rule to capture GuardDuty findings.
- Used the following event pattern:

```json
{
  "source": ["aws.guardduty"],
  "detail-type": ["GuardDuty Finding"]
}
```

- Configured the existing SNS topic `ThreatDetectionAlerts` as the EventBridge target.
- Enabled the EventBridge rule.

**Screenshots:**

![EventBridge rule configuration](screenshots/06-eventbridge/eventbridge-rule.png)

![EventBridge GuardDuty event pattern](screenshots/06-eventbridge/eventbridge-event-pattern.png)

![EventBridge SNS target configuration](screenshots/06-eventbridge/eventbridge-sns-target.png)

### Exercise 7 – Security Testing and Validation

Performed validation to confirm that both monitoring and alerting workflows were functioning.

**Test 1: SNS Email Notification**

- Published a test message to the SNS topic.
- Successfully received the test email.

**Test 2: GuardDuty Finding Notification**

- Generated a GuardDuty sample finding.
- Used the finding type `Backdoor:EC2/DenialOfService.Tcp`.
- Verified that EventBridge captured the GuardDuty finding.
- Successfully received the finding notification through SNS.

**Test 3: CloudWatch Alarm Notification**

- Observed the `Alarm-IamPolicyChanges` alarm transition from `OK` to `ALARM`.
- The alarm reported a metric datapoint of `2.0`, exceeding the configured threshold of `1.0`.
- Successfully received the CloudWatch alarm notification through SNS.

**Validation Result:** Both the GuardDuty EventBridge notification workflow and CloudWatch alarm notification workflow successfully delivered email notifications.

**Screenshots:**

![GuardDuty sample finding](screenshots/07-validation/guardduty-sample-finding.png)

![GuardDuty email notification](screenshots/07-validation/guardduty-email-notification.png)

![CloudWatch alarm email notification](screenshots/07-validation/cloudwatch-alarm-email.png)

## Security Monitoring Workflows

**Workflow 1 – AWS Account Activity Monitoring**

AWS CloudTrail → CloudWatch Logs → Metric Filters → CloudWatch Alarms → SNS → Email Notification

**Workflow 2 – GuardDuty Threat Detection**

Amazon GuardDuty → Amazon EventBridge → Amazon SNS → Email Notification

## Key Learning Outcomes

Through this project, I gained practical experience in:

- Implementing AWS account activity monitoring using CloudTrail.
- Understanding threat detection capabilities using Amazon GuardDuty.
- Centralizing and analyzing AWS logs using CloudWatch Logs.
- Creating CloudWatch metric filters for security monitoring.
- Configuring CloudWatch alarms for automated security detection.
- Implementing SNS-based email notification workflows.
- Integrating GuardDuty with EventBridge for automated security alerting.
- Testing and validating AWS security monitoring and notification workflows.

## Cost Considerations

AWS services used in this project may incur charges depending on usage, configuration, and applicable free-tier eligibility.

Review AWS pricing before deploying the solution in a production environment. Monitor CloudTrail storage, CloudWatch Logs ingestion and retention, GuardDuty usage, and other enabled services.

## Disclaimer

This project was developed as a hands-on AWS cybersecurity lab for educational and portfolio purposes.

GuardDuty sample findings are simulated security events and do not represent actual security incidents or compromised AWS resources.

The solution demonstrates foundational security monitoring and alerting capabilities and is not intended to replace a complete enterprise Security Information and Event Management (SIEM) solution.
