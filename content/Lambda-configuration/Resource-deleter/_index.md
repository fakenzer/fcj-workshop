---
title: "Creating ResourceDeleter Lambda Function"
weight: 2
chapter: false
pre: " <b> 3.2. </b> "
---

In this section, we will create the **ResourceDeleter** Lambda function to automatically delete non-compliant AWS resources.

## Lambda Function: ResourceDeleter

#### Purpose

This function will perform the following critical functions:

- Receive resource information to be deleted from ResourceScanner
- Execute resource deletion in proper order and process
- Handle complex dependencies (such as RDS cluster instances)
- Log detailed deletion process for audit and troubleshooting
- Return success/failure results

#### Supported Resource Types

ResourceDeleter can delete the following types of resources:

- **EC2 instances**: Terminate instances
- **VPC**: Delete VPC (with dependency checks)
- **S3 buckets**: Empty and delete bucket
- **RDS instances**: Delete with skip final snapshot
- **RDS clusters**: Delete cluster and all instances
- **FSx file systems**: Delete file system

#### Steps to Create Lambda Function

##### Step 1: Access Lambda Console

1. Log in to AWS Console
2. Search for **"Lambda"** in the search bar
3. Select Lambda service

##### Step 2: Create New Function

1. Click **"Create function"** button
2. Select **"Author from scratch"**

##### Step 3: Configure Function

**Function configuration**:

- **Function name**: `ResourceDeleter`
- **Runtime**: `Python 3.10`

**Execution role**:

1. Find and click on **"Change default execution role"**
2. Select **"Use an existing role"**
3. In the dropdown, select the **"ResourceManagerRole"** created earlier

##### Step 4: Deploy Code

1. Click **"Create function"** to create the Lambda function
2. Delete the existing sample code in the editor
3. Paste the following Python code into the editor:

```python
import json
import boto3
from datetime import datetime
from botocore.exceptions import ClientError
import time


def delete_rds_cluster_instances(rds_client, cluster_id, region):
    """Delete all instances in a cluster before deleting the cluster"""
    try:
        print(f"[DEBUG] Checking for instances in cluster: {cluster_id}")
        response = rds_client.describe_db_clusters(DBClusterIdentifier=cluster_id)
        cluster = response['DBClusters'][0]

        db_cluster_members = cluster.get('DBClusterMembers', [])
        if not db_cluster_members:
            print(f"[INFO] No instances found in cluster {cluster_id}")
            return True

        print(f"[INFO] Found {len(db_cluster_members)} instances in cluster {cluster_id}")

        # Delete all instances in the cluster
        for member in db_cluster_members:
            instance_id = member['DBInstanceIdentifier']
            print(f"[DEBUG] Deleting cluster instance: {instance_id}")
            try:
                rds_client.delete_db_instance(
                    DBInstanceIdentifier=instance_id,
                    SkipFinalSnapshot=True,
                    DeleteAutomatedBackups=True
                )
                print(f"[INFO] Initiated deletion of cluster instance: {instance_id}")
            except ClientError as e:
                if 'InvalidDBInstanceState' in str(e):
                    print(f"[WARN] Instance {instance_id} already being deleted")
                else:
                    raise

        # Wait for instances to be deleted
        if db_cluster_members:
            print(f"[INFO] Waiting for {len(db_cluster_members)} instances to be deleted...")
            max_wait = 300  # 5 minutes
            wait_time = 0

            while wait_time < max_wait:
                try:
                    response = rds_client.describe_db_clusters(DBClusterIdentifier=cluster_id)
                    cluster = response['DBClusters'][0]
                    remaining_instances = cluster.get('DBClusterMembers', [])

                    if not remaining_instances:
                        print("[INFO] All cluster instances deleted successfully")
                        break

                    print(f"[DEBUG] Still waiting for {len(remaining_instances)} instances to be deleted...")
                    time.sleep(15)
                    wait_time += 15

                except ClientError as e:
                    if 'DBClusterNotFoundFault' in str(e):
                        print("[INFO] Cluster already deleted")
                        return True
                    raise

            if wait_time >= max_wait:
                print(f"[WARN] Timeout waiting for cluster instances to be deleted")
                return False

        return True

    except ClientError as e:
        if 'DBClusterNotFoundFault' in str(e):
            print(f"[INFO] Cluster {cluster_id} not found")
            return True
        raise


def lambda_handler(event, context):
    print("[DEBUG] Lambda delete function triggered")
    print(f"[DEBUG] Event received: {json.dumps(event)}")

    resource_arn = event.get('resource_arn')
    if not resource_arn:
        print("[ERROR] Missing resource_arn in event")
        return {'statusCode': 400, 'error': 'No resource ARN provided'}

    print(f"[INFO] Processing resource ARN: {resource_arn}")

    try:
        # Parse ARN to get region
        arn_parts = resource_arn.split(':')
        region = arn_parts[3] if len(arn_parts) > 3 else 'us-east-1'

        # Initialize clients with proper region
        ec2 = boto3.client('ec2', region_name=region)
        s3 = boto3.client('s3', region_name=region)
        rds = boto3.client('rds', region_name=region)
        fsx = boto3.client('fsx', region_name=region)

        if ':ec2:' in resource_arn and 'instance/' in resource_arn:
            # EC2 Instance
            instance_id = resource_arn.split('/')[-1]
            print(f"[DEBUG] Detected EC2 instance. ID: {instance_id}")
            ec2.terminate_instances(InstanceIds=[instance_id])
            result = f"Terminated EC2 instance: {instance_id}"

        elif ':ec2:' in resource_arn and 'vpc/' in resource_arn:
            # VPC
            vpc_id = resource_arn.split('/')[-1]
            print(f"[DEBUG] Detected VPC. ID: {vpc_id}")

            # VPC deletion requires removing dependencies first
            print("[DEBUG] Checking VPC dependencies...")

            # Get VPC details
            response = ec2.describe_vpcs(VpcIds=[vpc_id])
            if not response['Vpcs']:
                result = f"VPC {vpc_id} not found"
            elif response['Vpcs'][0]['IsDefault']:
                result = f"Cannot delete default VPC: {vpc_id}"
            else:
                # For now, just attempt deletion - in production you'd want to handle dependencies
                try:
                    ec2.delete_vpc(VpcId=vpc_id)
                    result = f"Deleted VPC: {vpc_id}"
                except ClientError as e:
                    if 'DependencyViolation' in str(e):
                        result = f"VPC {vpc_id} has dependencies - manual cleanup required"
                    else:
                        raise

        elif ':s3:' in resource_arn:
            # S3 Bucket
            bucket_name = resource_arn.split(':::')[-1]
            print(f"[DEBUG] Detected S3 bucket. Name: {bucket_name}")

            # Get bucket region
            try:
                bucket_region = s3.get_bucket_location(Bucket=bucket_name)['LocationConstraint']
                if bucket_region:
                    s3 = boto3.client('s3', region_name=bucket_region)
                    s3_resource = boto3.resource('s3', region_name=bucket_region)
                else:
                    s3_resource = boto3.resource('s3', region_name='us-east-1')
            except:
                s3_resource = boto3.resource('s3', region_name=region)

            bucket = s3_resource.Bucket(bucket_name)

            print("[DEBUG] Emptying S3 bucket contents")
            bucket.objects.all().delete()
            bucket.object_versions.all().delete()

            print("[DEBUG] Deleting S3 bucket")
            s3.delete_bucket(Bucket=bucket_name)
            result = f"Deleted S3 bucket: {bucket_name}"

        elif ':rds:' in resource_arn:
            # RDS Database or Cluster
            # ARN format: arn:aws:rds:region:account:db:instance-name or arn:aws:rds:region:account:cluster:cluster-name
            arn_parts = resource_arn.split(':')
            if len(arn_parts) >= 6:
                resource_type = arn_parts[5]  # 'db' or 'cluster'
                resource_identifier = arn_parts[6] if len(arn_parts) > 6 else arn_parts[5]

                print(f"[DEBUG] RDS ARN parts: {arn_parts}")
                print(f"[DEBUG] Resource type: {resource_type}, Identifier: {resource_identifier}")

                if resource_type == 'db':
                    # RDS Database Instance
                    print(f"[DEBUG] Detected RDS instance. ID: {resource_identifier}")
                    rds.delete_db_instance(
                        DBInstanceIdentifier=resource_identifier,
                        SkipFinalSnapshot=True,
                        DeleteAutomatedBackups=True
                    )
                    result = f"Deleted RDS instance: {resource_identifier}"

                elif resource_type == 'cluster':
                    # RDS Cluster
                    print(f"[DEBUG] Detected RDS cluster. ID: {resource_identifier}")

                    # First, delete all instances in the cluster
                    if delete_rds_cluster_instances(rds, resource_identifier, region):
                        print(f"[DEBUG] Deleting RDS cluster: {resource_identifier}")
                        rds.delete_db_cluster(
                            DBClusterIdentifier=resource_identifier,
                            SkipFinalSnapshot=True
                        )
                        result = f"Deleted RDS cluster and its instances: {resource_identifier}"
                    else:
                        result = f"Failed to delete all instances in cluster {resource_identifier}"
                else:
                    result = f"Unknown RDS resource type: {resource_type}"
            else:
                result = f"Invalid RDS ARN format: {resource_arn}"

        elif ':fsx:' in resource_arn:
            # FSx File System
            file_system_id = resource_arn.split('/')[-1]
            print(f"[DEBUG] Detected FSx file system. ID: {file_system_id}")
            fsx.delete_file_system(FileSystemId=file_system_id)
            result = f"Deleted FSx file system: {file_system_id}"

        else:
            print("[WARN] Unsupported resource type")
            result = f"Unsupported resource type: {resource_arn}"

        print(f"[SUCCESS] {result}")

        return {
            'statusCode': 200,
            'result': result,
            'resource_arn': resource_arn,
            'deleted_at': datetime.utcnow().isoformat()
        }

    except ClientError as e:
        error_code = e.response['Error']['Code']
        error_msg = f"AWS Error [{error_code}]: {str(e)}"
        print(f"[ERROR] {error_msg}")

        return {
            'statusCode': 500,
            'error': error_msg,
            'resource_arn': resource_arn,
            'error_code': error_code
        }

    except Exception as e:
        error_msg = f"Failed to delete {resource_arn}: {str(e)}"
        print(f"[ERROR] {error_msg}")

        return {
            'statusCode': 500,
            'error': error_msg,
            'resource_arn': resource_arn
        }
```

4. Click **"Deploy"** button to deploy the code

#### ResourceDeleter Function Details

##### ARN Processing and Client Initialization

The `lambda_handler()` function performs:

**Parse ARN to determine region**:

- Split ARN into components
- Initialize AWS clients with correct region
- Support fallback region if not determinable

**Initialize clients by region**:

- EC2 client for instances and VPCs
- S3 client with region detection
- RDS client for databases and clusters
- FSx client for file systems

##### Processing Each Resource Type

**EC2 Instances**:

- Use `terminate_instances()` API
- Extract instance ID from ARN
- Log termination process

**VPC**:

- Check VPC existence
- Detect default VPC to avoid accidental deletion
- Handle dependency violations
- Require manual cleanup for complex VPCs

**S3 Buckets**:

- Auto-detect bucket region
- Empty bucket contents and versions
- Delete bucket after emptying
- Handle cross-region buckets

**RDS Instances**:

- Skip final snapshot for speed
- Delete automated backups
- Simple handling for standalone instances

**RDS Clusters**:

- Delete all instances in cluster first
- Wait for instances to be completely deleted
- Timeout protection (5 minutes)
- Delete cluster after instances are clean

**FSx File Systems**:

- Direct deletion with file system ID
- Handle ARN format extraction

##### Special Handling for RDS Clusters

The `delete_rds_cluster_instances()` function performs:

**Cluster members management**:

- List all instances in cluster
- Delete each instance with proper configuration
- Handle cases where instance is already being deleted

**Waiting mechanism**:

- Wait for all instances to complete deletion
- Check periodically every 15 seconds
- Timeout after 5 minutes to avoid infinite loop
- Proper error handling for cluster not found

##### Logging and Error Handling

The function provides comprehensive logging:

```
[DEBUG] - Detailed processing information
[INFO] - Important deletion information
[WARN] - Warnings about potential issues
[ERROR] - Errors occurred during deletion
[SUCCESS] - Confirmation when deletion successful
```

#### Additional Configuration

##### Timeout and Memory Settings

Adjust for production environment:

1. **Timeout**:

   - Minimum 10 minutes for RDS clusters
   - 15 minutes for complex resources
   - Increase if many dependencies exist

2. **Memory**:

   - 512MB for basic operations
   - 1GB for large S3 buckets
   - 2GB for complex multi-resource deletions

3. **Environment Variables**:
   - `AWS_REGION`: Default region
   - `LOG_LEVEL`: Logging verbosity
   - `MAX_WAIT_TIME`: Timeout for waiting operations

## Results

After completing this section, you will have:

**ResourceDeleter Lambda Function** with capabilities to:

- Delete 6 main AWS resource types
- Handle complex dependencies (RDS clusters)
- Timeout protection and error handling
- Detailed logging for audit trails
- Cross-region support

This Lambda function will ensure resource deletion is performed safely, with control, and auditability.
