---
title: "Creating RDS Database"
weight: 3
chapter: false
pre: " <b> 7.3 </b> "
---

## Objective

Test the ability to detect and handle RDS databases with and without tags to evaluate the effectiveness of the compliance monitoring system for database resources.

## Implementation Steps

### 1. Open RDS Console

- Log into AWS Management Console
- Search and select the **RDS** service
- Ensure you have permissions to create and manage RDS databases

### 2. Create RDS Database WITHOUT tags (Test case 1)

#### 2.1 Create Database

- Click the **Create database** button
- Select **Standard create** for more configuration options

#### 2.2 Engine Options

- Choose **MySQL** or **PostgreSQL** (free tier)
- Select the latest **Version**
- Choose **Templates**: **Free tier** to save costs

![Choose free tier](/images/7.Test/009-choosefreetie.png)

#### 2.3 Settings

- **DB instance identifier**: `test-db-no-tag`
- **Master username**: `admin`
- **Master password**: Create a strong password (at least 8 characters)
- **Confirm password**: Re-enter the password

![Setting](/images/7.Test/010-settingaccount.png)

#### 2.4 Instance Configuration

- **DB instance class**: Select **db.t3.micro** (Free tier eligible)
- **Storage type**: **General Purpose SSD (gp2)**
- **Allocated storage**: 20 GB (minimum)

![Setting](/images/7.Test/012-setting.png)

#### 2.5 Availability & Durability

- **Multi-AZ deployment**: Select **No** to save costs
- Suitable for testing environment

#### 2.6 Connectivity

- **Virtual private cloud (VPC)**: Select Default VPC
- **Subnet group**: Select default
- **Public access**: **Yes** (to enable connection testing)
- **VPC security group**: Select default or create a new security group

#### 2.7 Database Authentication

- Keep default **Password authentication**

#### 2.8 **IMPORTANT: Do NOT add tags**

- In the **Additional configuration** section, expand **Tags**
- **DO NOT add any tags**
- Skip this section completely
- Purpose: test detection of RDS database non-compliance with tagging policy

![No Tag](/images/7.Test/013-notag.png)

#### 2.9 Create Database

- Review all settings
- Click **Create database**
- Wait for database creation (approximately 10-15 minutes)

### 3. Create RDS Database WITH tags (Test case 2)

#### 3.1 Create second Database

- Click **Create database** again
- Select **Standard create**

#### 3.2 Engine Options

- Choose the same engine (MySQL/PostgreSQL)
- Select the same version
- Choose **Templates**: **Free tier**

#### 3.3 Settings

- **DB instance identifier**: `test-db-with-tag`
- **Master username**: `admin`
- **Master password**: Use the same password
- **Confirm password**: Re-enter the password

#### 3.4 Similar Configuration

- Same Instance Configuration
- Same Availability & Durability settings
- Same Connectivity settings
- Same Database Authentication

#### 3.5 **IMPORTANT: Add required tags**

- In the **Additional configuration** section, expand **Tags**
- Click **Add tag**
- Add the following tags:

| Key           | Value | Description       |
| ------------- | ----- | ----------------- |
| `Environment` | `lab` | Usage environment |

![Tag](/images/7.Test/011-tag.png)

#### 3.6 Create Database

- Review all settings
- Click **Create database**
- Wait for successful database creation

### 4. Important Notes

#### 4.1 Cost Considerations

- RDS Free tier has a limit of 750 hours/month
- Monitor usage to avoid incurring charges
- Consider deleting databases after testing

#### 4.2 Security

- Only enable public access in testing environment
- Use strong passwords
- Configure security groups carefully

#### 4.3 Monitoring

- Databases will be checked by the compliance monitoring system
- Expected results:
  - `test-db-no-tag`: Trigger compliance violation
  - `test-db-with-tag`: Pass compliance check
