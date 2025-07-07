---
title: "Creating IAM User"
weight: 3
chapter: false
pre: " <b> 2.3 </b> "
---

In this section, we will create an IAM User and assign the **RequireEnvironmentTagPolicy** to ensure this user can only create resources when the mandatory Environment tag is present.

## Purpose

Create an IAM User with the following characteristics:

- Must have the **"Environment"** tag when creating resources
- Only allow tag values: `prod`, `dev`, `test`, `lab`
- Prevent creation of EC2, S3, RDS resources without the tag
- Ensure governance and compliance adherence

## Steps to Create IAM User

### Step 1: Access IAM Console

1. Log in to the AWS Console
2. Search for **"IAM"** in the search bar
3. Select the IAM service

### Step 2: Create New User

1. In the IAM interface, select **"Users"** from the left menu
2. Click **"Create user"**

### Step 3: Configure User

1. **User name**: `TagEnforcementUser`
2. Select **"Provide user access to the AWS Management Console"**
3. Choose **"I want to create an IAM user"**
4. **Console password**: Select **"Custom password"** and enter a strong password
5. Uncheck **"Users must create a new password at next sign-in"** (optional)
6. Click **"Next"**

### Step 4: Assign Permissions

1. Select **"Attach policies directly"**
2. Search for **"RequireEnvironmentTagPolicy"**
3. Check the **RequireEnvironmentTagPolicy** policy
4. Add other basic policies if needed:
   - **IAMReadOnlyAccess** (so the user can view IAM information)
   - **ReadOnlyAccess** (so the user can view existing resources)

### Step 5: Review and Create

1. Review all information
2. Click **"Create user"**

## Results

After completing this section, you will have:

1. **TagEnforcementUser**: User with permissions restricted by tag policy
2. **Tag Enforcement**: All new resources must have the Environment tag
3. **Compliance**: Ensure adherence to tagging regulations
4. **Testing Environment**: Environment to test policy functionality

This user will not be able to create EC2, S3, or RDS resources without the Environment tag with valid values, ensuring effective governance and cost management.
