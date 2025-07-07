---
title: "Cleanup Resources"
weight: 8
chapter: false
pre: " <b> 8. </b> "
---

## Delete Monitoring Resources

### CloudWatch

- Go to **Dashboards** / Select and delete all dashboards

### SNS

- Go to **SNS Console** / **Subscriptions** / Delete all subscriptions first
- Go to **Topics** / Select and delete all topics

## Delete Serverless Resources

### Lambda Functions

- Go to **Lambda Console** / **Functions** / Select and delete all functions

### Step Functions

- Go to **Step Functions Console** / **State machines** / Delete all state machines

## Delete Event Resources

### EventBridge

- Go to **EventBridge Console** / **Rules** / Remove targets from rules first
- Delete all custom rules

### Systems Manager

- Go to **Systems Manager Console** / **Parameter Store** / Delete all parameters

## Delete Security Resources

### IAM

- Go to **IAM Console** / **Roles** / Detach policies from roles first
- Delete custom roles
- Go to **Policies** / Delete custom policies

## Final Verification

### Check Each Service

- CloudWatch: No alarms, dashboards, log groups
- SNS: No topics or subscriptions
- Lambda: No functions or layers
- Step Functions: No state machines
- EventBridge: No rules
- Systems Manager: No parameters
- IAM: No custom roles or policies

### Monitor Billing

- Go to **Billing Console** / Check for unexpected charges
- Set up billing alerts for future monitoring

## Important Notes

- Always backup important data before deletion
- Some resources may take time to delete completely
- Delete dependent resources first to avoid errors
- Check billing dashboard after cleanup
