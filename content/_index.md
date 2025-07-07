---
title: "Workshop Introduction"
weight: 1
chapter: false
pre: " <b> 1. </b> "
---

# Workshop Introduction

In our journey of exploring and working with AWS, we've all probably experienced those "heart-stopping" moments when receiving service bills that far exceed our expectations, simply because we forgot to turn off unused resources. Perhaps after a hurried lab session, you forgot to clean up EC2 instances, or when switching regions, you didn't notice EC2, RDS, S3 buckets still silently "consuming" costs, or simply turned on services for a few days but left them running for months. These small oversights can lead to significant financial consequences. To solve this problem, I want to share the idea of this lab on an automated system that helps find and remove unused AWS resources based on tags, helping save costs and providing peace of mind when starting to learn cloud.

### Architecture Diagram

The following diagram provides a visual representation of the services used in this simple lab and how they are connected. This application uses **IAM, Lambda, SNS, AWS Step Functions, Amazon EventBridge, AWS Systems Manager** to delete lab resources such as **VPC, EC2, S3, RDS.**

As you go through the workshop, you will learn in detail about these services and find resources that help you master them quickly.

![Architecture Application Schema](/images/1.Introduce/001-architectdiagram.png)

### What You Will Achieve

A system that helps you find and optimize costs at a very low price or free if you configure it correctly.

Better understanding of AWS services such as:

- **IAM**: User management and access control. Proper configuration helps limit risks and control resources effectively.
- **Lambda**: Run code without servers (serverless), only charged when executed – very cost-effective.
- **AWS EventBridge**: Automatically process AWS events to trigger cost optimization workflows.
- **CloudWatch**: Monitor performance, logs and alerts – helps monitor costs and unusual activities.
- **...**

### Requirements

- **AWS Account:** with administrator privileges.

- **If you are using an AWS FreeTier account, that's great**

{{% notice note%}}
Accounts created within the last 24 hours may not yet have access to the services required for this lab
{{% /notice%}}
