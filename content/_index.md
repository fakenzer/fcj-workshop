---
title: "Introduction"
weight: 1
chapter: false
pre: " <b> 1. </b> "
---

In our journey of exploring and working with AWS, we've all experienced those "heart-stopping" moments when receiving service bills that far exceed our expectations, simply because we forgot to turn off unused resources. Perhaps after a rushed lab session, you forgot to clean up EC2 instances; or when switching regions, you didn't notice EC2s, RDS, or S3 buckets still silently "burning" costs; or you simply enabled a service for a few days but left it running for an entire month. These small oversights can lead to significant financial consequences. To solve this problem, I want to share the idea behind this lab about an automated system that helps find and remove unused AWS resources, helping save costs and providing peace of mind when working in the cloud.

### Architecture Overview

The following diagram provides a visual representation of the services used in this simple lab and how they are connected. This application uses **IAM, Lambda, SNS, AWS Step Functions, Amazon EventBridge, AWS Systems Manager** to delete lab resources such as **VPC, EC2, S3, RDS.**

As you go through the workshop, you'll learn about these services in detail and find resources to help you grasp them quickly.

![Architecture Application Schema]()

### What You'll Achieve

A system that helps you find and optimize costs at a very low price or free if you configure it correctly.

Better understanding of AWS services such as:

- **IAM**: User management and access control. Proper configuration helps limit risks and control resources effectively.
- **Lambda**: Run code without servers (serverless), only charged when executed – very cost-effective.
- **AWS EventBridge**: Automatically process AWS events to trigger cost optimization workflows.
- **CloudWatch**: Monitor performance, logs, and alerts – helps monitor costs and unusual activities.
- **...**

### Requirements

**1 AWS account:** with administrator privileges.

**If you're using an AWS FreeTier account, that's perfect**

{{% notice note%}}
Accounts created within the last 24 hours may not yet have access to the services required for this lab
{{% /notice%}}
