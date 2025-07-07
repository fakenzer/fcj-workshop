---
title: "Creating S3 Buckets"
weight: 2
chapter: false
pre: " <b> 7.2 </b> "
---

## Objective

Test the ability to detect and handle S3 buckets with and without tags to evaluate the effectiveness of the compliance monitoring system for storage resources.

## Implementation Steps

### 1. Open S3 Console

- Log in to AWS Management Console
- Search for and select the **S3** service
- Ensure you have permissions to create and manage S3 buckets

### 2. Create S3 Bucket WITHOUT tags (Test case 1)

#### 2.1 Create Bucket

- Click the orange **Create bucket** button
- Enter bucket name: `test-bucket-no-tag`

  ![Name s3](/images/7.Test/006-nameofs3.png)

#### 2.2 Choose AWS Region

- Select the appropriate region (e.g., us-east-1, ap-southeast-1)
- Ensure it's the same region as your compliance monitoring system

#### 2.3 Object Ownership

- Choose **ACLs disabled (recommended)**
- Bucket owner will own all objects

#### 2.4 Block Public Access settings

- Keep default: **Block all public access** checked
- Ensure bucket security

#### 2.5 Bucket Versioning

- Select **Disable** to simplify testing
- Can enable if you need to test versioning compliance

#### 2.6 Default encryption

- Choose **Amazon S3 managed keys (SSE-S3)**
- Or keep default settings

#### 2.7 **IMPORTANT: Do not add tags**

- In the **Tags** section, **DO NOT add any tags**
- Skip this section completely
- Purpose: test detection of S3 buckets that don't comply with tagging policy

#### 2.8 Create Bucket

- Review all settings
- Click **Create bucket**
- Wait for bucket to be created successfully

### 3. Create S3 Bucket WITH tags (Test case 2)

#### 3.1 Create second bucket

- Click **Create bucket** again
- Enter bucket name: `test-bucket-with-tag`
  ![Name s3](/images/7.Test/007-nameofs3withtag.png)

#### 3.2 Configure similarly

- Choose the same AWS Region
- Same Object Ownership settings
- Same Block Public Access settings
- Same Versioning and encryption settings

#### 3.3 **IMPORTANT: Add required tags**

- In the **Tags** section, click **Add tag**
- Add the following tags:

| Key           | Value | Description           |
| ------------- | ----- | --------------------- |
| `Environment` | `lab` | Environment for usage |

![Tag s3](/images/7.Test/008-tags3.png)

#### 3.4 Create Bucket

- Review all settings
- Click **Create bucket**
- Wait for bucket to be created successfully

### 4. Upload test file (Optional)

#### 4.1 Upload file to bucket without tags

- Click on bucket `test-bucket-no-tag`
- Click **Upload**
- Select a test file or create a simple text file
- Upload the file

#### 4.2 Upload file to bucket with tags

- Do the same with bucket `test-bucket-with-tag`
- Purpose: test compliance check for objects inside buckets

### 5. Verify Results

#### 5.1 View bucket list

- In S3 Console, view the bucket list
- Note the names and creation times of both buckets

#### 5.2 Check tags

- Click on each bucket
- Go to **Properties** tab
- Scroll down to **Tags** section to see details
- Compare the differences between buckets

## Important Notes

- Only run tests in development/staging environments
- Ensure to delete buckets after testing to avoid charges
- Check IAM permissions before implementation
- Record bucket names and creation times to track results
- S3 bucket names must be globally unique across all AWS accounts
