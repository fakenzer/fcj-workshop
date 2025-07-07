---
title: "Creating ResourceScanner Lambda Function"
weight: 3
chapter: false
pre: " <b> 3.3. </b> "
---

In this section, we will create the **ResourceScanner** Lambda function to scan and check tags of AWS resources.

## Lambda Function: ResourceScanner

### Purpose

This function will perform important tasks:

- Scan resources in AWS account
- Check if resources exist or have been deleted
- Identify resources missing required tags
- Classify resources by Environment for automatic deletion
- Support whitelist to protect critical resources

### Supported Resource Types

ResourceScanner checks the following resource types:

- **EC2 instances**: Check running/stopped status
- **VPC**: Check available/deleting status
- **S3 buckets**: Check bucket existence
- **RDS instances**: Check DB instance status
- **RDS clusters**: Check cluster status
- **FSx file systems**: Check filesystem lifecycle

### Steps to Create Lambda Function

#### Step 1: Access Lambda Console

1. Log in to AWS Console
2. Search for **"Lambda"** in the search bar
3. Select Lambda service

   ![Choose lambda](/images/3.Lambda/001-lambda.png)

#### Step 2: Create New Function

1. Click **"Create function"** button
2. Choose **"Author from scratch"**

#### Step 3: Configure Function

**Function configuration**:

- **Function name**: `ResourceScanner`
- **Runtime**: `Python 3.10`

**Execution role**:

1. Find and click **"Change default execution role"**
2. Select **"Use an existing role"**
3. In dropdown, choose **"ResourceManagerRole"** created earlier
   ![Create Scanner Function](/images/3.Lambda/002-createscanner.png)

#### Step 4: Deploy Code

1. Click **"Create function"** to create the Lambda function
2. Delete existing sample code in the editor
3. Paste the following Python code into the editor:

```python
import json
import boto3
from datetime import datetime
from botocore.exceptions import ClientError, NoCredentialsError, BotoCoreError
from concurrent.futures import ThreadPoolExecutor, as_completed
import threading

# Thread-safe counter for logging
class ThreadSafeCounter:
    def __init__(self):
        self._value = 0
        self._lock = threading.Lock()

    def increment(self):
        with self._lock:
            self._value += 1
            return self._value

    @property
    def value(self):
        with self._lock:
            return self._value

def get_all_regions():
    """Retrieve all AWS regions."""
    try:
        ec2 = boto3.client('ec2', region_name='us-east-1')
        response = ec2.describe_regions()
        regions = [region['RegionName'] for region in response['Regions']]
        print(f"[DEBUG] Found {len(regions)} AWS regions")
        return regions
    except Exception as e:
        print(f"[ERROR] Failed to get regions: {str(e)}")
        return ['us-east-1', 'us-west-1', 'us-west-2', 'eu-west-1', 'ap-southeast-1']

# ==================== SERVICE-SPECIFIC SCANNERS ====================

def scan_ec2_resources(region):
    """Scan EC2 instances and VPCs in a region."""
    print(f"[DEBUG] Scanning EC2 resources in {region}")
    resources = []

    try:
        resource_client = boto3.client('resourcegroupstaggingapi', region_name=region)
        ec2_resources = resource_client.get_resources(
            ResourceTypeFilters=['ec2:instance', 'ec2:vpc']
        )

        for resource in ec2_resources['ResourceTagMappingList']:
            resources.append({
                'ResourceARN': resource['ResourceARN'],
                'Tags': resource.get('Tags', []),
                'Service': 'ec2',
                'Region': region
            })

        print(f"[DEBUG] Found {len(resources)} EC2 resources in {region}")
        return resources

    except ClientError as e:
        print(f"[ERROR] Error scanning EC2 in {region}: {str(e)}")
        return []

def get_s3_buckets_without_tags():
    """Find all S3 buckets, including those without tags."""
    print("[DEBUG] Scanning S3 buckets for untagged resources")
    s3 = boto3.client('s3')
    resources = []

    try:
        response = s3.list_buckets()
        buckets = response.get('Buckets', [])
        print(f"[DEBUG] Found {len(buckets)} S3 buckets to check")

        for bucket in buckets:
            bucket_name = bucket['Name']
            bucket_arn = f"arn:aws:s3:::{bucket_name}"
            try:
                tags_response = s3.get_bucket_tagging(Bucket=bucket_name)
                tags = [{'Key': tag['Key'], 'Value': tag['Value']} for tag in tags_response.get('TagSet', [])]
            except ClientError as e:
                if e.response['Error']['Code'] == 'NoSuchTagSet':
                    tags = []
                    print(f"[INFO] Bucket {bucket_name} has no tags")
                else:
                    print(f"[ERROR] Error getting tags for bucket {bucket_name}: {str(e)}")
                    continue

            resources.append({
                'ResourceARN': bucket_arn,
                'Tags': tags,
                'Service': 's3',
                'Region': 'global',
                'CreationDate': bucket['CreationDate'].isoformat()
            })

        print(f"[DEBUG] Found {len(resources)} S3 buckets")
        return resources

    except ClientError as e:
        print(f"[ERROR] Error listing S3 buckets: {str(e)}")
        return []

def scan_rds_resources(region):
    """Scan RDS resources (clusters and instances) with proper deletion types."""
    print(f"[DEBUG] Scanning RDS resources in {region}")
    resources = []

    try:
        rds = boto3.client('rds', region_name=region)

        # Scan RDS Clusters
        try:
            clusters_response = rds.describe_db_clusters()
            for cluster in clusters_response.get('DBClusters', []):
                cluster_arn = cluster['DBClusterArn']
                tags_response = rds.list_tags_for_resource(ResourceName=cluster_arn)
                tags = [{'Key': tag['Key'], 'Value': tag['Value']} for tag in tags_response.get('TagList', [])]

                resources.append({
                    'ResourceARN': cluster_arn,
                    'Tags': tags,
                    'Service': 'rds',
                    'Region': region,
                    'ResourceType': 'cluster',
                    'DeletionType': 'parent',
                    'Status': cluster.get('Status', 'unknown'),
                    'Engine': cluster.get('Engine', 'unknown')
                })
        except ClientError as e:
            print(f"[WARN] Error scanning RDS clusters in {region}: {str(e)}")

        # Scan RDS Instances
        try:
            instances_response = rds.describe_db_instances()
            for instance in instances_response.get('DBInstances', []):
                instance_arn = instance['DBInstanceArn']
                tags_response = rds.list_tags_for_resource(ResourceName=instance_arn)
                tags = [{'Key': tag['Key'], 'Value': tag['Value']} for tag in tags_response.get('TagList', [])]

                deletion_type = 'standalone'
                if instance.get('ReadReplicaSourceDBInstanceIdentifier'):
                    deletion_type = 'child'
                elif instance.get('ReadReplicaDBInstanceIdentifiers'):
                    deletion_type = 'parent'
                elif instance.get('DBClusterIdentifier'):
                    deletion_type = 'cluster_member'

                resources.append({
                    'ResourceARN': instance_arn,
                    'Tags': tags,
                    'Service': 'rds',
                    'Region': region,
                    'ResourceType': 'instance',
                    'DeletionType': deletion_type,
                    'Status': instance.get('DBInstanceStatus', 'unknown'),
                    'Engine': instance.get('Engine', 'unknown'),
                    'ClusterIdentifier': instance.get('DBClusterIdentifier'),
                    'ReplicaSource': instance.get('ReadReplicaSourceDBInstanceIdentifier'),
                    'ReplicaTargets': instance.get('ReadReplicaDBInstanceIdentifiers', [])
                })
        except ClientError as e:
            print(f"[WARN] Error scanning RDS instances in {region}: {str(e)}")

        print(f"[DEBUG] Found {len(resources)} RDS resources in {region}")
        return resources

    except ClientError as e:
        print(f"[ERROR] Error scanning RDS in {region}: {str(e)}")
        return []

def scan_fsx_resources(region):
    """Scan FSx file systems in a region."""
    print(f"[DEBUG] Scanning FSx resources in {region}")
    resources = []

    try:
        resource_client = boto3.client('resourcegroupstaggingapi', region_name=region)
        fsx_resources = resource_client.get_resources(
            ResourceTypeFilters=['fsx:file-system']
        )

        for resource in fsx_resources['ResourceTagMappingList']:
            resources.append({
                'ResourceARN': resource['ResourceARN'],
                'Tags': resource.get('Tags', []),
                'Service': 'fsx',
                'Region': region
            })

        print(f"[DEBUG] Found {len(resources)} FSx resources in {region}")
        return resources

    except ClientError as e:
        print(f"[ERROR] Error scanning FSx in {region}: {str(e)}")
        return []

# ==================== RESOURCE EXISTENCE CHECKER ====================

def resource_exists(resource_arn):
    """Check if a resource exists."""
    print(f"[DEBUG] Checking resource existence for ARN: {resource_arn}")

    try:
        arn_parts = resource_arn.split(":")
        if len(arn_parts) < 6:
            print(f"[WARN] Invalid ARN format: {resource_arn}")
            return False

        service = arn_parts[2]
        region = arn_parts[3] if len(arn_parts) > 3 and arn_parts[3] else None
        resource_part = arn_parts[5]

        if service == 'ec2':
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
                return True
            except ClientError as e:
                if e.response['Error']['Code'] in ['DBInstanceNotFound', 'DBClusterNotFound']:
                    print(f"[INFO] RDS {resource_type} not found: {resource_id}")
                    return False
                raise

        elif service == 'fsx':
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
                return True
            except ClientError as e:
                if e.response['Error']['Code'] == 'FileSystemNotFound':
                    print(f"[INFO] FSx file system not found: {resource_id}")
                    return False
                raise

        else:
            print(f"[WARN] Unsupported service in ARN: {service}")
            return False

        return True

    except Exception as e:
        print(f"[ERROR] Unexpected error checking {resource_arn}: {str(e)}")
        return False

# ==================== MAIN SCANNING FUNCTIONS ====================

def scan_region_resources(region, counter):
    """Scan all resources in a region."""
    region_counter = counter.increment()
    print(f"[DEBUG] [{region_counter}] Starting scan for region: {region}")

    all_resources = []

    try:
        ec2_resources = scan_ec2_resources(region)
        rds_resources = scan_rds_resources(region)
        fsx_resources = scan_fsx_resources(region)

        all_resources.extend(ec2_resources)
        all_resources.extend(rds_resources)
        all_resources.extend(fsx_resources)

        print(f"[DEBUG] [{region_counter}] Region {region}: Found {len(all_resources)} total resources")

        return {
            'region': region,
            'resources': all_resources,
            'count': len(all_resources),
            'breakdown': {
                'ec2': len(ec2_resources),
                'rds': len(rds_resources),
                'fsx': len(fsx_resources)
            }
        }

    except Exception as e:
        print(f"[ERROR] [{region_counter}] Error scanning region {region}: {str(e)}")
        return {
            'region': region,
            'resources': [],
            'count': 0,
            'error': str(e)
        }

def lambda_handler(event, context):
    print("[DEBUG] Lambda triggered")
    print(f"[DEBUG] Event received: {json.dumps(event)}")

    try:
        ssm = boto3.client('ssm')

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

        regions = get_all_regions()
        print(f"[DEBUG] Will scan {len(regions)} regions")

        all_resources = []
        region_summary = {}
        counter = ThreadSafeCounter()

        print("[DEBUG] Starting multi-region scan...")

        max_workers = min(10, len(regions))
        with ThreadPoolExecutor(max_workers=max_workers) as executor:
            future_to_region = {
                executor.submit(scan_region_resources, region, counter): region
                for region in regions
            }

            for future in as_completed(future_to_region):
                try:
                    result = future.result()
                    region = result['region']
                    region_summary[region] = {
                        'count': result['count'],
                        'breakdown': result.get('breakdown', {}),
                        'error': result.get('error')
                    }
                    all_resources.extend(result['resources'])
                except Exception as e:
                    region = future_to_region[future]
                    print(f"[ERROR] Failed to get result for region {region}: {str(e)}")
                    region_summary[region] = {
                        'count': 0,
                        'error': str(e)
                    }

        print(f"[DEBUG] Multi-region scan completed. Total resources from regions: {len(all_resources)}")

        # Scan S3 buckets (global)
        print("[DEBUG] Scanning S3 buckets...")
        s3_resources = get_s3_buckets_without_tags()
        all_resources.extend(s3_resources)

        total_found = len(all_resources)
        print(f"[INFO] Total resources to process: {total_found}")

        results = []
        skipped_count = 0
        whitelist_skipped = 0
        deleted_skipped = 0

        for i, resource in enumerate(all_resources, 1):
            resource_arn = resource['ResourceARN']
            if i % 100 == 0:
                print(f"[DEBUG] Processing resource {i}/{total_found}")

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

            extra_info = {}
            if resource.get('Service') == 'rds':
                extra_info['deletion_type'] = resource.get('DeletionType', 'unknown')
                extra_info['resource_type'] = resource.get('ResourceType', 'unknown')
                if resource.get('ClusterIdentifier'):
                    extra_info['cluster_identifier'] = resource['ClusterIdentifier']

            results.append({
                'resource_arn': resource_arn,
                'action': action,
                'tags': tags,
                'details': details,
                'service': resource.get('Service', 'unknown'),
                'region': resource.get('Region', 'unknown'),
                'extra_info': extra_info,
                'timestamp': datetime.utcnow().isoformat()
            })

        print("[SUMMARY] Multi-region scan completed")
        print(f"Total resources found     : {total_found}")
        print(f"- Regional resources      : {total_found - len(s3_resources)}")
        print(f"- S3 buckets (global)     : {len(s3_resources)}")
        print(f"Active resources processed: {len(results)}")
        print(f"Skipped (total)           : {skipped_count}")
        print(f"- Whitelisted             : {whitelist_skipped}")
        print(f"- Deleted/Non-existent    : {deleted_skipped}")
        print("Actions to take:")
        print(f"- Delete immediately      : {len([r for r in results if r['action'] == 'DELETE_IMMEDIATE'])}")
        print(f"- Send warning            : {len([r for r in results if r['action'] == 'SEND_WARNING'])}")
        print(f"- No action needed        : {len([r for r in results if r['action'] == 'NO_ACTION'])}")

        print("\nRegion breakdown:")
        for region, info in region_summary.items():
            if info.get('error'):
                print(f"- {region}: ERROR - {info['error']}")
            else:
                breakdown = info.get('breakdown', {})
                print(f"- {region}: {info['count']} resources (EC2: {breakdown.get('ec2', 0)}, RDS: {breakdown.get('rds', 0)}, FSx: {breakdown.get('fsx', 0)})")

        return {
            'statusCode': 200,
            'resources': results,
            'scan_summary': {
                'total_found': total_found,
                'regional_resources': total_found - len(s3_resources),
                's3_buckets': len(s3_resources),
                'active_resources': len(results),
                'skipped_resources': skipped_count,
                'whitelist_skipped': whitelist_skipped,
                'deleted_skipped': deleted_skipped,
                'regions_scanned': len(regions),
                'region_breakdown': region_summary,
                'service_breakdown': {
                    'ec2': len([r for r in results if r.get('service') == 'ec2']),
                    'rds': len([r for r in results if r.get('service') == 'rds']),
                    's3': len([r for r in results if r.get('service') == 's3']),
                    'fsx': len([r for r in results if r.get('service') == 'fsx'])
                },
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
            'message': 'Multi-region resource scan failed'
        }
```

4. Click **"Deploy"** button to deploy the code

#### Policy and Whitelist Processing

**Load configuration from SSM**:

- `/resource-management/policies`: Contains required tags and auto-delete environments
- `/resource-management/whitelist`: List of protected ARNs

**Resource classification**:

- **SEND_WARNING**: Missing required tags
- **DELETE_IMMEDIATE**: Environment in auto-delete list
- **NO_ACTION**: Compliant resource

### Additional Configuration

#### Timeout and Memory

![Choose lambda](/images/3.Lambda/003-configurationtimeout.png)
For production environments, adjust:

1. **Timeout**: Increase to 10-15 minutes for large accounts
2. **Memory**: Increase to 512MB or 1GB
3. **Environment variables**: Configure region, logging level

## Results

After completing this section, you will have:

1. **ResourceScanner Lambda Function** with capabilities to:

   - Scan tagged resources mentioned in AWS account
   - Check resource existence and state
   - Classify resources according to compliance policies
   - Support whitelist to protect critical resources
   - Detailed logging for monitoring and debugging

2. **Service integration**:
   - SSM Parameter Store to load policies

This Lambda function will be the core component in the resource management and cost optimization workflow.
