---
title: "Creating EventBridge Rule"
weight: 5
chapter: false
pre: " <b> 5. </b> "
---

In this section, we will create an EventBridge Rule to automatically trigger the **ResourceManagementWorkflow** Step Functions on a scheduled basis.

## EventBridge Rule: ResourceManagementSchedule

#### Purpose

This EventBridge Rule will:

- Automatically trigger ResourceManagementWorkflow on schedule
- Run daily compliance checks at 5:00 PM UTC
- Send appropriate input parameters to Step Functions
- Use IAM role automatically created by AWS to invoke Step Functions
- Ensure stable workflow execution with monitoring capabilities

#### Schedule

The rule will trigger the workflow:

1. **Daily at 5:00 PM UTC** (12:00 AM Vietnam time)
2. **Can be customized** according to organizational needs

#### Steps to Create EventBridge Rule

##### Step 1: Access EventBridge Console

1. Log into AWS Console
2. Search for **"EventBridge"** in the search bar
3. Select Amazon EventBridge service
   ![Event Bridge](/images/5.EventBridge/001-eventbridge.png)

##### Step 2: Create New Rule

1. In the sidebar, select **"Rules"**
2. Click the **"Create rule"** button
3. Ensure you're in the **"default"** Event bus

##### Step 3: Configure Rule Definition

**Rule detail**:

- **Name**: `ResourceManagementSchedule`
- **Description**: `Daily trigger for resource management compliance check`
- **Event bus**: `default`
- **Rule type**: `Schedule`
  ![Create rule](/images/5.EventBridge/002-createrule.png)

##### Step 4: Configure Schedule Pattern

**Schedule pattern**:

1. Choose "A schedule that runs at a regular rate, such as every 10 minutes"
2. Or choose "A fine-grained schedule that runs at a specific time"

**Option 1: Cron Expression (Recommended)**

```
0 17 * * ? *
```

- Runs daily at 5:00 PM UTC
  ![Configuration cron](/images/5.EventBridge/003-configurationcron.png)
  **Option 2: Rate Expression**

```
rate(1 day)
```

- Runs every 24 hours from the first creation

##### Step 5: Configure Target

**Target configuration**:

1. Click **"Add target"**
2. In the **"Select a target"** dropdown, choose **"Step Functions state machine"**

**Step Functions configuration**:

- **State machine**: Select `ResourceManagementWorkflow`
- **Execution role**: Choose **"Create a new role for this specific resource"**

**AWS will automatically create a role with format**: `Amazon_EventBridge_Invoke_Step_Functions_Role_[random_id]`
![Select target](/images/5.EventBridge/004-selecttarget.png)

##### Step 6: Configure Additional Settings

**Retry policy**:

- **Maximum age of event**: 24 hours
- **Retry attempts**: 3

**Dead letter queue**:

- Choose "None" or create SQS queue to handle failed events

##### Step 7: Review and Create

1. Review all configuration
2. Click **"Create rule"**
3. Verify the rule was created successfully

#### Monitoring and Troubleshooting

##### CloudWatch Metrics

EventBridge provides metrics:

- **InvocationsCount**: Number of times rule is triggered
- **SuccessfulInvocations**: Number of successful invocations
- **FailedInvocations**: Number of failed invocations
- **TargetErrors**: Errors when invoking target

##### CloudWatch Logs

**EventBridge doesn't automatically log**, but can be enabled:

##### Step Functions Execution History

After EventBridge triggers:

1. Go to Step Functions Console
2. Select `ResourceManagementWorkflow`
3. View "Executions" tab to track status

#### Customization Options

##### Multiple Schedules

Can create multiple rules for different use cases:

```json
{
  "WeeklyDeepScan": "0 17 ? * MON *",
  "DailyLightScan": "0 17 * * ? *",
  "HourlyTestScan": "0 * * * ? *"
}
```

##### Dynamic Input

Use input transformer for dynamic values:

```json
{
  "scan_type": "daily_compliance_check",
  "timestamp": "<aws.events.event.ingestion-time>",
  "region": "<aws.events.event.region>"
}
```

##### Conditional Execution

Add input conditions:

```json
{
  "scan_type": "daily_compliance_check",
  "conditions": {
    "only_business_hours": true,
    "skip_weekends": false,
    "environment_filter": ["prod", "staging"]
  }
}
```

This EventBridge Rule ensures the resource management workflow runs automatically, consistently, and can be monitored effectively.
