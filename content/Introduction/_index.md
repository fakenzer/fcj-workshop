---
title: "Workshop Introduction"
weight: 1
chapter: false
pre: " <b> 1. </b> "
---

In the journey of exploring and working with AWS, surely all of us have experienced those "shocking" moments when receiving service bills that far exceed expectations, simply because we forgot to turn off unused resources. Perhaps after a rushed lab session, you forgot to clean up EC2 instances, or when switching regions, you didn't notice those EC2, RDS, S3 buckets still quietly "consuming" costs, or simply turned on services for a few days but then left them running for months. These small mistakes can lead to significant financial consequences. To solve this problem, I want to share the idea of this lab about an automated system that helps find and delete unused AWS resources based on tags, helping save costs and providing peace of mind when starting to learn cloud.

### Architecture Overview

The following diagram provides a visual representation of the services used in this simple lab and how they are connected. This application uses **IAM, Lambda, SNS, AWS Step Functions, Amazon EventBridge, AWS Systems Manager** to delete lab resources such as **VPC, EC2, S3, RDS**.

As you go through the workshop, you will learn details about these services and find resources to help you grasp them quickly.

![Architecture Application Schema](/images/1.Introduce/001-architectdiagram.png)

### What You Will Achieve

A system that helps you find and optimize costs at a very cheap price or free if you configure it correctly.

Better understanding of AWS services such as:

- **IAM**: User management and access permissions. Proper configuration helps limit risks and control resources effectively.
- **Lambda**: Run code without servers (serverless), only charged when executed – very cost-effective.
- **AWS EventBridge**: Automatically process AWS events to trigger cost optimization workflows.
- **CloudWatch**: Monitor performance, logs, and alerts – helps monitor costs and unusual activities.
- **...**

### Requirements

- **AWS Account:** with administrator privileges.

- **If you are using AWS FreeTier account, that's great**

{{% notice note%}}
Accounts created within the last 24 hours may not yet have access to the services required for this lab
{{% /notice%}}

### Main Content

1. [Introduction](Introduction)
2. [IAM Policy and Role Configuration](IAM-configuration)
3. [Lambda Configuration](Lambda-configuration)
4. [Create Step Functions](Set-up-StepFunction)
5. [EventBridge Setup](Create-EventBridge)
6. [CloudWatch Setup](Set-up-CloudWatch)
7. [Testing](Test-system)
