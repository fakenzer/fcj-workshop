---
title: "IAM Configuration"
weight: 2
chapter: false
pre: " <b> 2. </b> "
---

This chapter provides detailed guidance on creating and configuring custom IAM Policies and Roles in AWS to manage access permissions, optimize costs, and ensure compliance with tagging rules.

## Objectives

- Create IAM Policy with specific permissions for AWS resource management
- Create IAM Policy requiring mandatory **"Environment"** tag for resources
- Create IAM Role that can be used by AWS services
- Establish secure and efficient permission system for automation

## Required Knowledge

- Basic understanding of AWS IAM
- Experience working with AWS Console
- Understanding of JSON format
- Knowledge of AWS tagging best practices

## Expected Results

After completing this chapter, you will have:

- **ManageResourcePolicy**: IAM policy with permissions for resource, tag, and cost management
- **RequireEnvironmentTagPolicy**: Policy requiring mandatory environment tag
- **ResourceManagerRole**: Comprehensive role that can be used for automation and resource management

## Important Notes

- Always carefully review permissions before creating policies
- Follow the **"least privilege"** principle when granting permissions
- Regularly review and update policies according to actual needs
- Backup policy and role configurations before making changes
- Ensure all resources have appropriate **Environment** tags
