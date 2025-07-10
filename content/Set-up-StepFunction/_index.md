---
title: "Creating Step Functions"
weight: 4
chapter: false
pre: " <b> 4. </b> "
---

In this section, we will create Step Functions **ResourceManagementWorkflow** to orchestrate the entire automated AWS resource management process.

## Step Functions: ResourceManagementWorkflow

### Purpose

This Step Functions will coordinate the entire resource management workflow:

- Start the resource scanning process via ResourceScanner
- Check and process the list of discovered resources
- Classify and perform appropriate actions for each resource
- Send alert notifications or delete resources according to policy
- Process multiple resources in parallel to optimize performance

### Operation Flow

The workflow executes the following steps:

1. **ScanResources**: Trigger ResourceScanner to find violating resources
2. **CheckResourcesExist**: Check if there are resources to process
3. **CheckResourcesNotEmpty**: Verify the resource list is not empty
4. **ProcessEachResource**: Process each resource in parallel
5. **DecideAction**: Classify the action to be performed
6. **Execute Actions**: Perform delete, warning, or schedule delete

### Action Types

- **DELETE_IMMEDIATE**: Delete immediately (resources with Environment tag as `lab`)
- **SEND_WARNING**: Send warning (resources missing Environment tag)
- **NO_ACTION**: Wait 48h then delete (resources with Environment tag but not `lab`)

### Steps to Create Step Functions

#### Step 1: Access Step Functions Console

1. Log into AWS Console
2. Search for **"Step Functions"** in the search bar
3. Select AWS Step Functions service
   ![Step Functions](/images/4.StepFunctions/001-stepfunction.png)

#### Step 2: Create New State Machine

1. Click the **"Create state machine"** button
2. Choose **"Write your workflow in code"**
3. Select **"Standard"** state machine type

#### Step 3: Configure State Machine

**State machine configuration**:

- **Name**: `ResourceManagementWorkflow`
- **Type**: `Standard`
  ![Create Step Functions](/images/4.StepFunctions/002-createstepfunction.png)

#### Step 4: Definition Code

Paste the following JSON definition into the editor:

```json
{
  {
  "Comment": "Resource Management Workflow for scanning, notifying, and deleting multiple resources",
  "StartAt": "ScanResources",
  "States": {
    "ScanResources": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:your-region:your-aws-id:function:ResourceScanner",
      "ResultPath": "$.scan_result",
      "Next": "CheckResourcesExist",
      "Catch": [
        {
          "ErrorEquals": [
            "States.ALL"
          ],
          "ResultPath": "$.error",
          "Next": "HandleScanError"
        }
      ]
    },
    "CheckResourcesExist": {
      "Type": "Choice",
      "Choices": [
        {
          "Variable": "$.scan_result.resources",
          "IsPresent": true,
          "Next": "CheckResourcesNotEmpty"
        }
      ],
      "Default": "NoResourcesFound"
    },
    "CheckResourcesNotEmpty": {
      "Type": "Choice",
      "Choices": [
        {
          "Variable": "$.scan_result.resources[0]",
          "IsPresent": true,
          "Next": "ProcessResources"
        }
      ],
      "Default": "NoResourcesFound"
    },
    "NoResourcesFound": {
      "Type": "Pass",
      "Parameters": {
        "message": "No resources found to process",
        "status": "completed"
      },
      "End": true
    },
    "HandleScanError": {
      "Type": "Pass",
      "Parameters": {
        "message": "Scan failed",
        "status": "failed",
        "error.$": "$.error"
      },
      "End": true
    },
    "ProcessResources": {
      "Type": "Map",
      "ItemsPath": "$.scan_result.resources",
      "MaxConcurrency": 10,
      "Parameters": {
        "resource_arn.$": "$$.Map.Item.Value.resource_arn",
        "action.$": "$$.Map.Item.Value.action",
        "tags.$": "$$.Map.Item.Value.tags",
        "details.$": "$$.Map.Item.Value.details",
        "service.$": "$$.Map.Item.Value.service",
        "region.$": "$$.Map.Item.Value.region",
        "extra_info.$": "$$.Map.Item.Value.extra_info",
        "timestamp.$": "$$.Map.Item.Value.timestamp"
      },
      "Iterator": {
        "StartAt": "DecideAction",
        "States": {
          "DecideAction": {
            "Type": "Choice",
            "Choices": [
              {
                "Variable": "$.action",
                "StringEquals": "DELETE_IMMEDIATE",
                "Next": "DeleteNow"
              },
              {
                "Variable": "$.action",
                "StringEquals": "SEND_WARNING",
                "Next": "SendWarning"
              },
              {
                "Variable": "$.action",
                "StringEquals": "NO_ACTION",
                "Next": "WaitThenDelete"
              }
            ],
            "Default": "NoAction"
          },
          "DeleteNow": {
            "Type": "Task",
            "Resource": "arn:aws:lambda:your-region:your-aws-id:function:ResourceDeleter",
            "End": true,
            "Catch": [
              {
                "ErrorEquals": [
                  "States.ALL"
                ],
                "ResultPath": "$.error",
                "Next": "HandleDeleteError"
              }
            ]
          },
          "SendWarning": {
            "Type": "Task",
            "Resource": "arn:aws:lambda:your-region:your-aws-id:function:NotificationSender",
            "Parameters": {
              "resources.$": "States.Array($)"
            },
            "End": true,
            "Catch": [
              {
                "ErrorEquals": [
                  "States.ALL"
                ],
                "ResultPath": "$.error",
                "Next": "HandleNotificationError"
              }
            ]
          },
          "WaitThenDelete": {
            "Type": "Wait",
            "Seconds": 172800,
            "Next": "DeleteScheduled"
          },
          "DeleteScheduled": {
            "Type": "Task",
            "Resource": "arn:aws:lambda:your-region:your-aws-id:function:ResourceDeleter",
            "End": true,
            "Catch": [
              {
                "ErrorEquals": [
                  "States.ALL"
                ],
                "ResultPath": "$.error",
                "Next": "HandleScheduledDeleteError"
              }
            ]
          },
          "HandleDeleteError": {
            "Type": "Pass",
            "Parameters": {
              "status": "failed",
              "error.$": "$.error"
            },
            "End": true
          },
          "HandleNotificationError": {
            "Type": "Pass",
            "Parameters": {
              "status": "failed",
              "error.$": "$.error"
            },
            "End": true
          },
          "HandleScheduledDeleteError": {
            "Type": "Pass",
            "Parameters": {
              "status": "failed",
              "error.$": "$.error"
            },
            "End": true
          },
          "NoAction": {
            "Type": "Pass",
            "Parameters": {
              "status": "skipped",
              "message": "No action taken for resource"
            },
            "End": true
          }
        }
      },
      "ResultPath": "$.action_results",
      "End": true
    }
  }
}
```

![Workflow](/images/4.StepFunctions/003-workflow.png)
{{%notice note%}}
Replace `your-aws-id` with your actual Account ID and replace `your-region` with the region you're using.
{{%/notice%}}

#### Step 7: Review and Deploy

1. Review the entire configuration
2. Click **"Create state machine"**
3. Verify the state machine was created successfully

### Performance Configuration

- **MaxConcurrency**: 10 (process maximum 10 resources in parallel)
- **Timeout**: 1 hour (sufficient for including the 48h wait of scheduled delete)
- **Retry Policy**: 3 times with exponential backoff

This workflow ensures efficient, automated, and scalable AWS resource management according to organizational needs.
