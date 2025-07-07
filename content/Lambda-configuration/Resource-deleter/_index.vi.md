---
title: "Tạo Lambda Function ResourceDeleter"
weight: 4
chapter: false
pre: " <b> 3.4. </b> "
---

Trong phần này, chúng ta sẽ tạo Lambda function **ResourceDeleter** để thực hiện xóa tự động các tài nguyên AWS không tuân thủ quy định.

## Lambda Function: ResourceDeleter

### Mục đích

Function này sẽ thực hiện các chức năng quan trọng:

- Nhận thông tin tài nguyên cần xóa từ ResourceScanner
- Thực hiện xóa tài nguyên theo đúng thứ tự và quy trình
- Xử lý các phụ thuộc phức tạp (như RDS cluster instances)
- Ghi log chi tiết quá trình xóa để audit và troubleshooting
- Trả về kết quả thành công/thất bại

### Các loại tài nguyên được hỗ trợ

ResourceDeleter có khả năng xóa các loại tài nguyên sau:

- **EC2 instances**: Terminate instances
- **VPC**: Delete VPC (với kiểm tra dependencies)
- **S3 buckets**: Empty và delete bucket
- **RDS instances**: Delete với skip final snapshot
- **RDS clusters**: Delete cluster và tất cả instances
- **FSx file systems**: Delete file system

### Các bước tạo Lambda Function

#### Bước 1: Truy cập Lambda Console

1. Đăng nhập vào AWS Console
2. Tìm kiếm **"Lambda"** trong thanh tìm kiếm
3. Chọn dịch vụ Lambda

#### Bước 2: Tạo Function mới

1. Nhấp vào nút **"Create function"**
2. Chọn **"Author from scratch"**

#### Bước 3: Cấu hình Function

**Function configuration**:

- **Function name**: `ResourceDeleter`
- **Runtime**: `Python 3.10`

**Execution role**:

1. Tìm và nhấp vào mục **"Change default execution role"**
2. Chọn **"Use an existing role"**
3. Trong dropdown, chọn role **"ResourceManagerRole"** đã tạo trước đó

#### Bước 4: Deploy Code

1. Nhấp **"Create function"** để tạo Lambda function
2. Xóa code mẫu hiện có trong editor
3. Paste đoạn code Python sau vào editor:

```python
import json
import boto3
from datetime import datetime
from botocore.exceptions import ClientError
import time
import re

def get_cluster_from_instance_arn(instance_arn):
    """Convert instance ARN to cluster ARN"""
    instance_name = instance_arn.split(':')[-1]
    patterns = [
        r'^(.+)-instance-\d+$',  # cluster-name-instance-1
        r'^(.+)-\d+$',           # cluster-name-1
        r'^(.+)-writer$',        # cluster-name-writer
        r'^(.+)-reader$',        # cluster-name-reader
        r'^(.+)-replica-\d+$',   # cluster-name-replica-1
    ]
    for pattern in patterns:
        match = re.match(pattern, instance_name)
        if match:
            cluster_name = match.group(1)
            cluster_arn = instance_arn.replace(':db:', ':cluster:').replace(instance_name, cluster_name)
            return cluster_arn, cluster_name
    return None, None

def delete_rds_cluster_instances(rds_client, cluster_id, region):
    """Handle different RDS cluster deletion scenarios"""
    try:
        print(f"[DEBUG] Checking cluster details: {cluster_id}")
        response = rds_client.describe_db_clusters(DBClusterIdentifier=cluster_id)
        cluster = response['DBClusters'][0]
        engine = cluster.get('Engine', '').lower()
        engine_mode = cluster.get('EngineMode', '').lower()
        db_cluster_members = cluster.get('DBClusterMembers', [])
        cluster_status = cluster.get('Status', '').lower()

        print(f"[DEBUG] Cluster engine: {engine}, mode: {engine_mode}, status: {cluster_status}, members: {len(db_cluster_members)}")

        if cluster_status in ['deleting', 'deleted']:
            print(f"[INFO] Cluster {cluster_id} is already being deleted")
            return True

        # Aurora clusters: deleting cluster will auto-delete instances
        is_aurora_cluster = (
            engine.startswith('aurora') or
            engine_mode in ['provisioned', 'serverless'] or
            (engine in ['mysql', 'postgresql'] and len(db_cluster_members) > 0)
        )

        if is_aurora_cluster:
            print(f"[INFO] Aurora cluster detected - instances will be auto-deleted with cluster")
            return True

        # Non-Aurora clusters: delete instances first
        print(f"[INFO] Non-Aurora cluster - deleting {len(db_cluster_members)} instances")
        for member in db_cluster_members:
            instance_id = member['DBInstanceIdentifier']
            print(f"[DEBUG] Deleting instance: {instance_id}")
            try:
                rds_client.delete_db_instance(
                    DBInstanceIdentifier=instance_id,
                    SkipFinalSnapshot=True,
                    DeleteAutomatedBackups=True
                )
                print(f"[INFO] Initiated deletion of instance: {instance_id}")
            except ClientError as e:
                if 'InvalidDBInstanceState' in str(e):
                    print(f"[WARN] Instance {instance_id} already being deleted")
                else:
                    print(f"[ERROR] Failed to delete instance {instance_id}: {e}")
                    raise

        # Wait for instances to be deleted
        max_wait = 600
        wait_time = 0
        while wait_time < max_wait:
            try:
                response = rds_client.describe_db_clusters(DBClusterIdentifier=cluster_id)
                remaining_instances = response['DBClusters'][0].get('DBClusterMembers', [])
                if not remaining_instances:
                    print("[INFO] All cluster instances deleted")
                    return True
                print(f"[DEBUG] Waiting for {len(remaining_instances)} instances to delete...")
                time.sleep(30)
                wait_time += 30
            except ClientError as e:
                if 'DBClusterNotFoundFault' in str(e):
                    print("[INFO] Cluster already deleted")
                    return True
                raise
        print(f"[WARN] Timeout waiting for instances to delete")
        return False

    except ClientError as e:
        if 'DBClusterNotFoundFault' in str(e):
            print(f"[INFO] Cluster {cluster_id} not found")
            return True
        print(f"[ERROR] Error handling cluster {cluster_id}: {e}")
        raise

def empty_s3_bucket_with_tags(s3_client, s3_resource, bucket_name):
    """Empty S3 bucket and remove all tags/configurations"""
    try:
        print(f"[DEBUG] Emptying S3 bucket: {bucket_name}")
        bucket = s3_resource.Bucket(bucket_name)

        # Delete all object versions and objects
        try:
            bucket.object_versions.delete()
            bucket.objects.all().delete()
            print(f"[DEBUG] Deleted all objects/versions from bucket: {bucket_name}")
        except Exception as e:
            print(f"[DEBUG] Error deleting objects: {e}")

        # Remove bucket configurations
        configs = [
            ('tagging', s3_client.delete_bucket_tagging, 'NoSuchTagSet'),
            ('policy', s3_client.delete_bucket_policy, 'NoSuchBucketPolicy'),
            ('cors', s3_client.delete_bucket_cors, 'NoSuchCORSConfiguration'),
            ('lifecycle', s3_client.delete_bucket_lifecycle, 'NoSuchLifecycleConfiguration'),
            ('website', s3_client.delete_bucket_website, 'NoSuchWebsiteConfiguration'),
            ('notification', s3_client.put_bucket_notification_configuration, None),
            ('replication', s3_client.delete_bucket_replication, 'ReplicationConfigurationNotFoundError'),
            ('versioning', lambda **kwargs: s3_client.put_bucket_versioning(
                Bucket=bucket_name, VersioningConfiguration={'Status': ' reset to suspended'}), None),
        ]

        for config_name, delete_func, ignore_error in configs:
            try:
                if config_name == 'notification':
                    delete_func(Bucket=bucket_name, NotificationConfiguration={})
                elif config_name == 'versioning':
                    delete_func()
                else:
                    delete_func(Bucket=bucket_name)
                print(f"[DEBUG] Removed {config_name} from: {bucket_name}")
            except ClientError as e:
                if ignore_error and ignore_error in str(e):
                    print(f"[DEBUG] No {config_name} found on: {bucket_name}")
                else:
                    print(f"[DEBUG] Error removing {config_name}: {e}")

        return True
    except Exception as e:
        print(f"[ERROR] Error emptying bucket {bucket_name}: {e}")
        return False

def delete_vpc_dependencies(ec2_client, vpc_id):
    """Delete VPC dependencies to allow VPC deletion"""
    try:
        print(f"[DEBUG] Checking VPC dependencies for: {vpc_id}")

        # Subnets
        subnets = ec2_client.describe_subnets(Filters=[{'Name': 'vpc-id', 'Values': [vpc_id]}])['Subnets']
        for subnet in subnets:
            subnet_id = subnet['SubnetId']
            try:
                ec2_client.delete_subnet(SubnetId=subnet_id)
                print(f"[DEBUG] Deleted subnet: {subnet_id}")
            except ClientError as e:
                print(f"[WARN] Failed to delete subnet {subnet_id}: {e}")

        # Security groups (except default)
        security_groups = ec2_client.describe_security_groups(
            Filters=[{'Name': 'vpc-id', 'Values': [vpc_id]}]
        )['SecurityGroups']
        for sg in [sg for sg in security_groups if sg['GroupName'] != 'default']:
            try:
                ec2_client.delete_security_group(GroupId=sg['GroupId'])
                print(f"[DEBUG] Deleted security group: {sg['GroupId']}")
            except ClientError as e:
                print(f"[WARN] Failed to delete security group {sg['GroupId']}: {e}")

        # Route tables (except main)
        route_tables = ec2_client.describe_route_tables(
            Filters=[{'Name': 'vpc-id', 'Values': [vpc_id]}]
        )['RouteTables']
        for rt in [rt for rt in route_tables if not any(assoc.get('Main') for assoc in rt.get('Associations', []))]:
            try:
                ec2_client.delete_route_table(RouteTableId=rt['RouteTableId'])
                print(f"[DEBUG] Deleted route table: {rt['RouteTableId']}")
            except ClientError as e:
                print(f"[WARN] Failed to delete route table {rt['RouteTableId']}: {e}")

        # Internet gateways
        igws = ec2_client.describe_internet_gateways(
            Filters=[{'Name': 'attachment.vpc-id', 'Values': [vpc_id]}]
        )['InternetGateways']
        for igw in igws:
            igw_id = igw['InternetGatewayId']
            try:
                ec2_client.detach_internet_gateway(InternetGatewayId=igw_id, VpcId=vpc_id)
                ec2_client.delete_internet_gateway(InternetGatewayId=igw_id)
                print(f"[DEBUG] Deleted internet gateway: {igw_id}")
            except ClientError as e:
                print(f"[WARN] Failed to delete internet gateway {igw_id}: {e}")

        # NAT gateways
        nat_gws = ec2_client.describe_nat_gateways(
            Filters=[{'Name': 'vpc-id', 'Values': [vpc_id]}]
        )['NatGateways']
        for ng in [ng for ng in nat_gws if ng['State'] not in ['deleted', 'deleting']]:
            try:
                ec2_client.delete_nat_gateway(NatGatewayId=ng['NatGatewayId'])
                print(f"[DEBUG] Initiated deletion of NAT gateway: {ng['NatGatewayId']}")
            except ClientError as e:
                print(f"[WARN] Failed to delete NAT gateway {ng['NatGatewayId']}: {e}")

        # Verify dependencies
        dependencies = {
            'subnets': len(subnets),
            'security_groups': len([sg for sg in security_groups if sg['GroupName'] != 'default']),
            'route_tables': len([rt for rt in route_tables if not any(assoc.get('Main') for assoc in rt.get('Associations', []))]),
            'internet_gateways': len(igws),
            'nat_gateways': len([ng for ng in nat_gws if ng['State'] not in ['deleted', 'deleting']])
        }
        total_deps = sum(dependencies.values())
        print(f"[DEBUG] Remaining dependencies: {dependencies}")
        return total_deps == 0

    except Exception as e:
        print(f"[ERROR] Error checking VPC dependencies: {e}")
        return False

def lambda_handler(event, context):
    print("[DEBUG] Lambda delete function triggered")
    print(f"[DEBUG] Event received: {json.dumps(event, indent=2)}")

    resource_arn = event.get('resource_arn')
    action = event.get('action')
    extra_info = event.get('extra_info', {})
    service = event.get('service')
    region = event.get('region', 'us-east-1')

    if not resource_arn or not action:
        print("[ERROR] Missing resource_arn or action in event")
        return {'statusCode': 400, 'error': 'Missing resource_arn or action'}

    print(f"[INFO] Processing resource ARN: {resource_arn}, Action: {action}, Service: {service}, Region: {region}")

    if action != 'DELETE_IMMEDIATE':
        print(f"[INFO] Skipping resource {resource_arn} - action is {action}")
        return {'statusCode': 200, 'result': f"Skipped resource {resource_arn} - action not DELETE_IMMEDIATE"}

    try:
        arn_parts = resource_arn.split(':')
        if len(arn_parts) < 6:
            raise ValueError(f"Invalid ARN format: {resource_arn}")

        service = arn_parts[2]
        region = arn_parts[3] if arn_parts[3] else 'us-east-1'
        resource_part = ':'.join(arn_parts[5:])

        # Initialize clients
        clients = {
            'ec2': boto3.client('ec2', region_name=region),
            's3': boto3.client('s3'),  # S3 is global
            'rds': boto3.client('rds', region_name=region),
            'fsx': boto3.client('fsx', region_name=region),
        }

        # Route to appropriate handler
        if service == 'ec2':
            result = handle_ec2_resource(clients['ec2'], resource_arn, resource_part)
        elif service == 's3':
            result = handle_s3_resource(clients['s3'], resource_arn, resource_part)
        elif service == 'rds':
            result = handle_rds_resource(clients['rds'], resource_arn, resource_part, region, extra_info)
        elif service == 'fsx':
            result = handle_fsx_resource(clients['fsx'], resource_arn, resource_part)
        else:
            result = f"Unsupported service: {service}"

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

def handle_ec2_resource(ec2_client, resource_arn, resource_part):
    """Handle EC2 resource deletion"""
    if 'instance/' in resource_part:
        instance_id = resource_part.split('/')[-1]
        print(f"[DEBUG] Deleting EC2 instance: {instance_id}")
        try:
            response = ec2_client.describe_instances(InstanceIds=[instance_id])
            instance = response['Reservations'][0]['Instances'][0]
            if instance['State']['Name'] in ['terminated', 'terminating']:
                return f"EC2 instance {instance_id} is already {instance['State']['Name']}"
            ec2_client.terminate_instances(InstanceIds=[instance_id])
            return f"Terminated EC2 instance: {instance_id}"
        except ClientError as e:
            if 'InvalidInstanceID.NotFound' in str(e):
                return f"EC2 instance {instance_id} not found"
            raise

    elif 'vpc/' in resource_part:
        vpc_id = resource_part.split('/')[-1]
        print(f"[DEBUG] Deleting VPC: {vpc_id}")
        try:
            response = ec2_client.describe_vpcs(VpcIds=[vpc_id])
            vpc = response['Vpcs'][0]
            if vpc['IsDefault']:
                return f"Cannot delete default VPC: {vpc_id}"
            if not delete_vpc_dependencies(ec2_client, vpc_id):
                return f"VPC {vpc_id} has dependencies - manual cleanup required"
            ec2_client.delete_vpc(VpcId=vpc_id)
            return f"Deleted VPC: {vpc_id}"
        except ClientError as e:
            if 'InvalidVpcID.NotFound' in str(e):
                return f"VPC {vpc_id} not found"
            raise

    return f"Unsupported EC2 resource type: {resource_part}"

def handle_s3_resource(s3_client, resource_arn, resource_part):
    """Handle S3 resource deletion"""
    bucket_name = resource_part
    print(f"[DEBUG] Deleting S3 bucket: {bucket_name}")

    try:
        s3_client.head_bucket(Bucket=bucket_name)
        bucket_location = s3_client.get_bucket_location(Bucket=bucket_name)
        bucket_region = bucket_location['LocationConstraint'] or 'us-east-1'
        s3_regional = boto3.client('s3', region_name=bucket_region)
        s3_resource = boto3.resource('s3', region_name=bucket_region)

        if not empty_s3_bucket_with_tags(s3_regional, s3_resource, bucket_name):
            return f"Failed to empty S3 bucket: {bucket_name}"

        for attempt in range(3):
            try:
                s3_regional.delete_bucket(Bucket=bucket_name)
                return f"Deleted S3 bucket: {bucket_name}"
            except ClientError as e:
                if 'BucketNotEmpty' in str(e) and attempt < 2:
                    print(f"[DEBUG] Bucket still not empty, retrying in 10 seconds... (attempt {attempt + 1}/3)")
                    time.sleep(10)
                    continue
                raise
    except ClientError as e:
        if e.response['Error']['Code'] == '404':
            return f"S3 bucket {bucket_name} not found"
        raise

def handle_rds_resource(rds_client, resource_arn, resource_part, region, extra_info):
    """Handle RDS resource deletion based on DeletionType"""
    parts = resource_part.split(':')
    if len(parts) < 2:
        return f"Invalid RDS ARN format: {resource_arn}"

    resource_type = parts[0]
    resource_id = parts[1]
    deletion_type = extra_info.get('deletion_type', 'unknown')

    print(f"[DEBUG] RDS resource type: {resource_type}, id: {resource_id}, deletion_type: {deletion_type}")

    if resource_type == 'db':
        if deletion_type == 'cluster_member':
            cluster_id = extra_info.get('cluster_identifier')
            if cluster_id:
                print(f"[INFO] Instance {resource_id} is part of cluster {cluster_id} - deleting cluster")
                return handle_rds_cluster_deletion(rds_client, cluster_id, region)
            else:
                print(f"[WARN] Cluster member instance {resource_id} has no cluster_identifier - treating as standalone")
                deletion_type = 'standalone'

        if deletion_type == 'child':
            print(f"[INFO] Read replica instance {resource_id} - deleting directly")
        elif deletion_type == 'parent':
            print(f"[INFO] Parent instance {resource_id} with replicas - checking replicas first")
            try:
                response = rds_client.describe_db_instances(DBInstanceIdentifier=resource_id)
                instance = response['DBInstances'][0]
                replicas = instance.get('ReadReplicaDBInstanceIdentifiers', [])
                if replicas:
                    print(f"[WARN] Cannot delete parent instance {resource_id} - has {len(replicas)} replicas")
                    return f"Cannot delete parent instance {resource_id} - delete replicas first"
            except ClientError as e:
                if 'DBInstanceNotFound' in str(e):
                    return f"RDS instance {resource_id} not found"
                raise

        # Handle standalone or child instance deletion
        try:
            response = rds_client.describe_db_instances(DBInstanceIdentifier=resource_id)
            instance = response['DBInstances'][0]
            if instance['DBInstanceStatus'] in ['deleting', 'deleted']:
                return f"RDS instance {resource_id} is already {instance['DBInstanceStatus']}"

            if instance.get('DeletionProtection', False):
                print(f"[INFO] Disabling deletion protection for instance: {resource_id}")
                rds_client.modify_db_instance(
                    DBInstanceIdentifier=resource_id,
                    DeletionProtection=False,
                    ApplyImmediately=True
                )
                time.sleep(10)

            rds_client.delete_db_instance(
                DBInstanceIdentifier=resource_id,
                SkipFinalSnapshot=True,
                DeleteAutomatedBackups=True
            )
            return f"Deleted RDS instance: {resource_id}"
        except ClientError as e:
            if 'DBInstanceNotFound' in str(e):
                return f"RDS instance {resource_id} not found"
            raise

    elif resource_type == 'cluster':
        return handle_rds_cluster_deletion(rds_client, resource_id, region)

    return f"Unknown RDS resource type: {resource_type}"

def handle_rds_cluster_deletion(rds_client, cluster_id, region):
    """Handle RDS cluster deletion"""
    print(f"[DEBUG] Deleting RDS cluster: {cluster_id}")
    try:
        response = rds_client.describe_db_clusters(DBClusterIdentifier=cluster_id)
        cluster = response['DBClusters'][0]
        if cluster.get('DeletionProtection', False):
            print(f"[INFO] Disabling deletion protection for cluster: {cluster_id}")
            rds_client.modify_db_cluster(
                DBClusterIdentifier=cluster_id,
                DeletionProtection=False,
                ApplyImmediately=True
            )
            time.sleep(10)

        if not delete_rds_cluster_instances(rds_client, cluster_id, region):
            return f"Failed to prepare cluster {cluster_id} for deletion"

        for attempt in range(5):
            try:
                rds_client.delete_db_cluster(
                    DBClusterIdentifier=cluster_id,
                    SkipFinalSnapshot=True
                )
                return f"Deleted RDS cluster: {cluster_id}"
            except ClientError as e:
                if 'InvalidClusterStateFault' in str(e) and attempt < 4:
                    wait_time = 60 * (attempt + 1)
                    print(f"[WARN] Cluster not ready, retrying in {wait_time} seconds... (attempt {attempt + 1}/5)")
                    time.sleep(wait_time)
                    continue
                elif 'DBClusterNotFoundFault' in str(e):
                    return f"RDS cluster {cluster_id} was already deleted"
                raise
    except ClientError as e:
        if 'DBClusterNotFoundFault' in str(e):
            return f"RDS cluster {cluster_id} not found"
        raise

def handle_fsx_resource(fsx_client, resource_arn, resource_part):
    """Handle FSx resource deletion"""
    file_system_id = resource_part.split('/')[-1]
    print(f"[DEBUG] Deleting FSx file system: {file_system_id}")
    try:
        fsx_client.delete_file_system(FileSystemId=file_system_id)
        return f"Deleted FSx file system: {file_system_id}"
    except ClientError as e:
        if 'FileSystemNotFound' in str(e):
            return f"FSx file system {file_system_id} not found"
        raise
```

4. Nhấp nút **"Deploy"** để triển khai code

### Cấu hình bổ sung

#### Timeout và Memory Settings

Điều chỉnh cho production environment:

1. **Timeout**:

   - Minimum 10 phút cho RDS clusters
   - 15 phút cho complex resources
   - Tăng thêm nếu có nhiều dependencies

2. **Memory**:

   - 512MB cho basic operations
   - 1GB cho S3 buckets lớn
   - 2GB cho complex multi-resource deletions

3. **Environment Variables**:
   - `AWS_REGION`: Default region
   - `LOG_LEVEL`: Logging verbosity
   - `MAX_WAIT_TIME`: Timeout cho waiting operations

## Kết quả

Sau khi hoàn thành phần này, bạn sẽ có:

**ResourceDeleter Lambda Function** với khả năng:

- Xóa 6 loại tài nguyên AWS chính
- Xử lý dependencies phức tạp (RDS clusters)
- Timeout protection và error handling
- Logging chi tiết cho audit trails
- Cross-region support

Lambda function này sẽ đảm bảo việc xóa tài nguyên được thực hiện một cách an toàn, có kiểm soát và có thể audit được.
