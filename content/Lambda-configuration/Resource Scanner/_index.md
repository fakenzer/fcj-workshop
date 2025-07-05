---
title: "Creating ResourceScanner Lambda Function"
weight: 1
chapter: false
pre: " <b> 3.1. </b> "
---

In this section, we will create the **ResourceScanner** Lambda function to scan and check tags of AWS resources.

## Lambda Function: ResourceScanner

#### Purpose

This function will perform the following critical functions:

- Scan all tagged resources in the AWS account
- Check if resources exist or have been deleted
- Identify resources missing required tags
- Classify resources by Environment for automatic deletion
- Support whitelist to protect critical resources

#### Supported Resource Types

ResourceScanner checks the following resource types:

- **EC2 instances**: Check running/stopped status
- **VPC**: Check available/deleting status
- **S3 buckets**: Check bucket existence
- **RDS instances**: Check DB instance status
- **RDS clusters**: Check cluster status
- **FSx file systems**: Check filesystem lifecycle

#### Steps to Create Lambda Function

##### Step 1: Access Lambda Console

1. Log in to AWS Console
2. Search for **"Lambda"** in the search bar
3. Select Lambda service

   ![Choose lambda](/images/3.Lambda/001-lambda.png)

##### Step 2: Create New Function

1. Click **"Create function"** button
2. Select **"Author from scratch"**

##### Step 3: Configure Function

**Function configuration**:

- **Function name**: `ResourceScanner`
- **Runtime**: `Python 3.10`

**Execution role**:

1. Find and click on **"Change default execution role"**
2. Select **"Use an existing role"**
3. In the dropdown, select the **"ResourceManagerRole"** created earlier
   ![Create Scanner Function](/images/3.Lambda/002-createscanner.png)

##### Step 4: Deploy Code

1. Click **"Create function"** to create the Lambda function
2. Delete the existing sample code in the editor
3. Paste the following Python code into the editor:

```python
import json
import boto3
from datetime import datetime
from botocore.exceptions import ClientError, NoCredentialsError, BotoCoreError

def resource_exists(resource_arn):
    print(f"[DEBUG] Checking resource existence for ARN: {resource_arn}")

    try:
        arn_parts = resource_arn.split(":")
        if len(arn_parts) < 6:
            print(f"[WARN] Invalid ARN format: {resource_arn}")
            return False

        service = arn_parts[2]
        region = arn_parts[3] if len(arn_parts) > 3 and arn_parts[3] else None
        account_id = arn_parts[4]
        resource_part = arn_parts[5]
        print(f"[DEBUG] Parsed ARN - Service: {service}, Region: {region}, Account: {account_id}, Resource Part: {resource_part}")

        if service == 'ec2':
            # Handle both instances and VPCs
            if resource_part.startswith('instance/'):
                resource_id = resource_part.split("/")[-1]
                ec2 = boto3.client('ec2', region_name=region)
                try:
                    response = ec2.describe_instances(InstanceIds=[resource_id])
                    instances = [i for r in response['Reservations'] for i in r['Instances']]
                    if not instances:
                        print(f"[INFO] No EC2 instances found for {resource_id}")
                        return False
                    for instance in instances:
                        if instance['State']['Name'] in ['terminated', 'shutting-down']:
                            print(f"[INFO] EC2 instance in {instance['State']['Name']} state: {resource_id}")
                            return False
                except ClientError as e:
                    if e.response['Error']['Code'] == 'InvalidInstanceID.NotFound':
                        print(f"[INFO] EC2 instance not found: {resource_id}")
                        return False
                    raise

            elif resource_part.startswith('vpc/'):
                vpc_id = resource_part.split("/")[-1]
                ec2 = boto3.client('ec2', region_name=region)
                try:
                    response = ec2.describe_vpcs(VpcIds=[vpc_id])
                    vpcs = response.get('Vpcs', [])
                    if not vpcs:
                        print(f"[INFO] VPC not found: {vpc_id}")
                        return False
                    for vpc in vpcs:
                        if vpc['State'] in ['deleting', 'deleted']:
                            print(f"[INFO] VPC in {vpc['State']} state: {vpc_id}")
                            return False
                except ClientError as e:
                    if e.response['Error']['Code'] == 'InvalidVpcID.NotFound':
                        print(f"[INFO] VPC not found: {vpc_id}")
                        return False
                    raise

        elif service == 's3':
            bucket_name = resource_part
            s3 = boto3.client('s3')
            try:
                location = s3.get_bucket_location(Bucket=bucket_name)['LocationConstraint']
                if location:
                    s3 = boto3.client('s3', region_name=location)
                s3.head_bucket(Bucket=bucket_name)
            except ClientError as e:
                if e.response['Error']['Code'] == 'NoSuchBucket':
                    print(f"[INFO] S3 bucket not found: {bucket_name}")
                    return False
                raise

        elif service == 'rds':
            resource_type, resource_id = resource_part.split(":", 1) if ":" in resource_part else (resource_part, "")
            if resource_type not in ['db', 'cluster']:
                print(f"[WARN] Invalid RDS ARN resource type: {resource_type}")
                return False
            rds = boto3.client('rds', region_name=region)
            try:
                if resource_type == 'db':
                    response = rds.describe_db_instances(DBInstanceIdentifier=resource_id)
                    db_instances = response.get('DBInstances', [])
                    if not db_instances:
                        print(f"[INFO] RDS DB instance not found: {resource_id}")
                        return False
                    for db in db_instances:
                        if db['DBInstanceStatus'] in ['deleting', 'deleted']:
                            print(f"[INFO] RDS DB in {db['DBInstanceStatus']} state: {resource_id}")
                            return False
                elif resource_type == 'cluster':
                    response = rds.describe_db_clusters(DBClusterIdentifier=resource_id)
                    db_clusters = response.get('DBClusters', [])
                    if not db_clusters:
                        print(f"[INFO] RDS DB cluster not found: {resource_id}")
                        return False
                    for cluster in db_clusters:
                        if cluster['Status'] in ['deleting', 'deleted']:
                            print(f"[INFO] RDS cluster in {cluster['Status']} state: {resource_id}")
                            return False
            except ClientError as e:
                if e.response['Error']['Code'] in ['DBInstanceNotFound', 'DBClusterNotFound']:
                    print(f"[INFO] RDS {resource_type} not found: {resource_id}")
                    return False
                raise

        elif service == 'fsx':
            # Handle FSx file systems
            resource_id = resource_part.split("/")[-1]
            fsx = boto3.client('fsx', region_name=region)
            try:
                response = fsx.describe_file_systems(FileSystemIds=[resource_id])
                file_systems = response.get('FileSystems', [])
                if not file_systems:
                    print(f"[INFO] FSx file system not found: {resource_id}")
                    return False
                for fs in file_systems:
                    if fs['Lifecycle'] in ['DELETING', 'DELETED', 'FAILED']:
                        print(f"[INFO] FSx file system in {fs['Lifecycle']} state: {resource_id}")
                        return False
            except ClientError as e:
                if e.response['Error']['Code'] == 'FileSystemNotFound':
                    print(f"[INFO] FSx file system not found: {resource_id}")
                    return False
                raise

        else:
            print(f"[WARN] Unsupported service in ARN: {service}")
            return False

        return True

    except ClientError as e:
        error_code = e.response['Error']['Code']
        print(f"[ERROR] ClientError for {resource_arn} | Code: {error_code}")
        return False

    except (NoCredentialsError, BotoCoreError) as e:
        print(f"[ERROR] AWS client error for {resource_arn} | Error: {str(e)}")
        return False

    except Exception as e:
        print(f"[ERROR] Unexpected error checking {resource_arn} | Error: {str(e)}")
        return False

def lambda_handler(event, context):
    print("[DEBUG] Lambda triggered")
    print(f"[DEBUG] Event received: {json.dumps(event)}")

    try:
        ssm = boto3.client('ssm')
        resource_client = boto3.client('resourcegroupstaggingapi')

        print("[DEBUG] Loading policies and whitelist from SSM")
        try:
            policies = json.loads(
                ssm.get_parameter(Name='/resource-management/policies')['Parameter']['Value']
            )
        except ClientError as e:
            print(f"[ERROR] Failed to load policies: {str(e)}")
            return {'statusCode': 500, 'error': 'Failed to load policies'}

        try:
            whitelist_param = ssm.get_parameter(Name='/resource-management/whitelist')['Parameter']['Value']
            whitelist = [arn.strip() for arn in whitelist_param.split(',') if arn.strip()] if whitelist_param else []
        except ClientError as e:
            print(f"[WARN] Failed to load whitelist, proceeding without: {str(e)}")
            whitelist = []

        print(f"[DEBUG] Loaded policy keys: {list(policies.keys())}")
        print(f"[DEBUG] Whitelist contains {len(whitelist)} items")
        for i, arn in enumerate(whitelist, 1):
            print(f"[DEBUG]   Whitelist[{i-1}]: {arn}")

        print("[DEBUG] Fetching tagged resources")
        resources = resource_client.get_resources(
            ResourceTypeFilters=[
                'ec2:instance',
                'ec2:vpc',
                's3:bucket',
                'rds:db',
                'rds:cluster',
                'fsx:file-system'
            ]
        )

        total_found = len(resources['ResourceTagMappingList'])
        print(f"[INFO] Total tagged resources found: {total_found}")

        results = []
        skipped_count = 0
        whitelist_skipped = 0
        deleted_skipped = 0

        for i, resource in enumerate(resources['ResourceTagMappingList'], 1):
            resource_arn = resource['ResourceARN']
            print(f"[DEBUG] Processing resource {i}/{total_found}: {resource_arn}")

            if resource_arn in whitelist:
                print(f"[INFO] Skipping whitelisted resource: {resource_arn}")
                whitelist_skipped += 1
                skipped_count += 1
                continue

            if not resource_exists(resource_arn):
                print(f"[INFO] Skipping non-existent/deleted resource: {resource_arn}")
                deleted_skipped += 1
                skipped_count += 1
                continue

            tags = {tag['Key']: tag['Value'] for tag in resource.get('Tags', [])}
            missing_tags = [tag for tag in policies['required_tags'] if tag not in tags]
            env_tag = tags.get('Environment', '').lower()

            if missing_tags:
                action = "SEND_WARNING"
                details = f"Missing required tags: {missing_tags}"
            elif env_tag in policies['auto_delete_environments']:
                action = "DELETE_IMMEDIATE"
                details = f"Auto-delete environment: {env_tag}"
            else:
                action = "NO_ACTION"
                details = "Resource compliant"

            results.append({
                'resource_arn': resource_arn,
                'action': action,
                'tags': tags,
                'details': details,
                'timestamp': datetime.utcnow().isoformat()
            })

            print(f"[RESULT] {resource_arn}: {action} - {details}")

        print("[SUMMARY] Scan completed")
        print(f"Total resources found     : {total_found}")
        print(f"Active resources processed: {len(results)}")
        print(f"Skipped (total)           : {skipped_count}")
        print(f"- Whitelisted           : {whitelist_skipped}")
        print(f"- Deleted/Non-existent  : {deleted_skipped}")
        print("Actions to take:")
        print(f"- Delete immediately    : {len([r for r in results if r['action'] == 'DELETE_IMMEDIATE'])}")
        print(f"- Send warning          : {len([r for r in results if r['action'] == 'SEND_WARNING'])}")
        print(f"- No action needed      : {len([r for r in results if r['action'] == 'NO_ACTION'])}")

        return {
            'statusCode': 200,
            'resources': results,
            'scan_summary': {
                'total_found': total_found,
                'active_resources': len(results),
                'skipped_resources': skipped_count,
                'whitelist_skipped': whitelist_skipped,
                'deleted_skipped': deleted_skipped,
                'actions': {
                    'delete_immediate': len([r for r in results if r['action'] == 'DELETE_IMMEDIATE']),
                    'send_warning': len([r for r in results if r['action'] == 'SEND_WARNING']),
                    'no_action': len([r for r in results if r['action'] == 'NO_ACTION'])
                }
            }
        }

    except Exception as e:
        print(f"[ERROR] Lambda execution failed: {str(e)}")
        return {
            'statusCode': 500,
            'error': str(e),
            'message': 'Resource scan failed'
        }
```

4. Click **"Deploy"** button to deploy the code

#### ResourceScanner Function Details

##### ARN Analysis and Resource Checking

The `resource_exists()` function performs:

**ARN Parsing**:

- Split ARN into components: service, region, account, resource
- Identify the type of resource to check

**Check each resource type**:

- **EC2 instances**: Check state is not `terminated` or `shutting-down`
- **VPC**: Check state is not `deleting` or `deleted`
- **S3 buckets**: Check bucket existence with `head_bucket()`
- **RDS instances/clusters**: Check status is not `deleting` or `deleted`
- **FSx file systems**: Check lifecycle is not `DELETING`, `DELETED`, or `FAILED`

##### Policies and Whitelist Processing

The `lambda_handler()` function performs:

**Load configuration from SSM**:

- `/resource-management/policies`: Contains required tags and auto-delete environments
- `/resource-management/whitelist`: List of protected ARNs

**Resource classification**:

- **SEND_WARNING**: Missing required tags
- **DELETE_IMMEDIATE**: Environment belongs to auto-delete list
- **NO_ACTION**: Compliant resource

##### Logging and Monitoring

The function provides detailed logging:

```
[DEBUG] - Debug information
[INFO] - Important information
[WARN] - Warnings
[ERROR] - Errors occurred
[RESULT] - Processing result for each resource
[SUMMARY] - Scan process summary
```

#### Additional Configuration

##### Timeout and Memory

![Choose lambda](/images/3.Lambda/003-configurationtimeout.png)
For production environments, adjustments needed:

1. **Timeout**: Increase to 10-15 minutes for large accounts
2. **Memory**: Increase to 512MB or 1GB
3. **Environment variables**: Configure region, logging level

## Results

After completing this section, you will have:

1. **ResourceScanner Lambda Function** with capabilities to:

   - Scan the specified tagged resources in AWS account
   - Check resource existence and state
   - Classify resources according to compliance policies
   - Support whitelist to protect critical resources
   - Detailed logging for monitoring and debugging

2. **Service integration**:
   - SSM Parameter Store to load policies

This Lambda function will be the core component in the resource management and cost optimization workflow.
