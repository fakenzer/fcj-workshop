---
title: "Lambda Setup"
weight: 3
chapter: false
pre: " <b> 3. </b> "
---

This chapter provides detailed guidance on creating and configuring Lambda Functions in AWS to automate resource management, monitor costs, and send notifications when tagging rule violations are detected.

## Objectives

- Set up a complete Resource Management System
- Configure automatic alerts for resource management
- Create ResourceScanner Lambda Function to scan and detect non-compliant resources
- Create ResourceDeleter Lambda Function to delete resources that violate rules
- Create NotificationSender Lambda Function to send notifications via email and Slack

## Prerequisites

- Basic understanding of AWS Lambda
- Experience working with Python
- Knowledge of AWS SDK (boto3 for Python)
- Understanding of AWS CloudWatch Events/EventBridge
- Experience with AWS SNS and SES
- Knowledge of JSON format and IAM configuration

## Expected Outcomes

After completing this chapter, you will have:

- **ResourceScanner Function**: Lambda that scans resources and detects violations
- **ResourceDeleter Function**: Lambda that deletes non-compliant resources
- **NotificationSender Function**: Lambda that sends notifications (email)

## Architecture Overview

The Lambda system consists of 3 main components:

1. **ResourceScanner**: Scans AWS resources and checks compliance
2. **ResourceDeleter**: Handles deletion of resources that violate rules
3. **NotificationSender**: Sends notifications via email

## Important Notes

- Always test Lambda functions in development environment first
- Configure appropriate timeout and memory settings for each function
- Ensure IAM permissions are configured correctly
- Regularly backup code and Lambda function configurations
