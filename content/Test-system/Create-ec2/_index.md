---
title: "Creating EC2 Instances"
weight: 1
chapter: false
pre: " <b> 7.1 </b> "
---

## Objective

Test the ability to detect and handle EC2 instances with and without tags to evaluate the effectiveness of the compliance monitoring system.

## Implementation Steps

### 1. Open EC2 Console

- Log in to AWS Management Console
- Search for and select the **EC2** service
- Choose the appropriate region (e.g., us-east-1, ap-southeast-1)
- Ensure you are in the correct region that the compliance system is monitoring

### 2. Create EC2 Instance without tags (Test case 1)

#### 2.1 Launch Instance

- Click the orange **Launch Instance** button
- Name the instance: `test-instance-no-tag`

![Name](/images/7.Test/001-nameec2notag.png)

#### 2.2 Choose AMI (Amazon Machine Image)

- Select **Amazon Linux 2023** (Free tier eligible)
- Or choose **Ubuntu Server 22.04 LTS** (Free tier eligible)

#### 2.3 Choose Instance Type

- Select **t2.micro** (Free tier eligible)
- This instance has 1 vCPU and 1 GiB Memory

![Choose system](/images/7.Test/002-choosesystem.png)

#### 2.4 Configure Key Pair

- Choose an existing key pair or create a new one
- If creating new: name it `test-keypair` and download the .pem file

![Key pair](/images/7.Test/003-keypair.png)

#### 2.5 Network Settings

- Keep default VPC and subnet
- Create a new security group or select an existing one
- Allow SSH (port 22) from your IP address

#### 2.6 **IMPORTANT: Do not add tags**

- In the **Advanced details** section, expand **Tags**
- **DO NOT add any tags** - skip this section completely
- Purpose: test detection of resources that don't comply with tagging policy

#### 2.7 Launch Instance

- Review the configuration
- Click **Launch instance**
- Wait for the instance to change to **Running** state

### 3. Create EC2 Instance with tags (Test case 2)

#### 3.1 Launch second instance

- Click **Launch Instance** again
- Name the instance: `test-instance-with-tag`

#### 3.2 Configure similarly

- Choose the same AMI (Amazon Linux 2023 or Ubuntu)
- Select instance type **t2.micro**
- Use the same key pair created earlier
- Configure network settings similarly

#### 3.3 **IMPORTANT: Add required tags**

- In the **Advanced details** section, expand **Tags**
- Click **Add tag** and add the following tags:

| Key           | Value | Description           |
| ------------- | ----- | --------------------- |
| `Environment` | `lab` | Environment for usage |

![name with tag](/images/7.Test/004-namewithtag.png)

## Important Notes

- Only run tests in development/staging environments
- Ensure to terminate instances after testing to avoid charges
- Check IAM permissions before implementation
- Record Instance ID and creation time to track results

![All ec2](/images/7.Test/005-ec2-instance.png)
