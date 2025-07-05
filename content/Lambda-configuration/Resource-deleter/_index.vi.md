---
title: "Tạo Lambda Function ResourceDeleter"
weight: 2
chapter: false
pre: " <b> 3.2. </b> "
---

Trong phần này, chúng ta sẽ tạo Lambda function **ResourceDeleter** để thực hiện xóa tự động các tài nguyên AWS không tuân thủ quy định.

## Lambda Function: ResourceDeleter

#### Mục đích

Function này sẽ thực hiện các chức năng quan trọng:

- Nhận thông tin tài nguyên cần xóa từ ResourceScanner
- Thực hiện xóa tài nguyên theo đúng thứ tự và quy trình
- Xử lý các phụ thuộc phức tạp (như RDS cluster instances)
- Ghi log chi tiết quá trình xóa để audit và troubleshooting
- Trả về kết quả thành công/thất bại

#### Các loại tài nguyên được hỗ trợ

ResourceDeleter có khả năng xóa các loại tài nguyên sau:

- **EC2 instances**: Terminate instances
- **VPC**: Delete VPC (với kiểm tra dependencies)
- **S3 buckets**: Empty và delete bucket
- **RDS instances**: Delete với skip final snapshot
- **RDS clusters**: Delete cluster và tất cả instances
- **FSx file systems**: Delete file system

#### Các bước tạo Lambda Function

##### Bước 1: Truy cập Lambda Console

1. Đăng nhập vào AWS Console
2. Tìm kiếm **"Lambda"** trong thanh tìm kiếm
3. Chọn dịch vụ Lambda

##### Bước 2: Tạo Function mới

1. Nhấp vào nút **"Create function"**
2. Chọn **"Author from scratch"**

##### Bước 3: Cấu hình Function

**Function configuration**:

- **Function name**: `ResourceDeleter`
- **Runtime**: `Python 3.10`

**Execution role**:

1. Tìm và nhấp vào mục **"Change default execution role"**
2. Chọn **"Use an existing role"**
3. Trong dropdown, chọn role **"ResourceManagerRole"** đã tạo trước đó

##### Bước 4: Deploy Code

1. Nhấp **"Create function"** để tạo Lambda function
2. Xóa code mẫu hiện có trong editor
3. Paste đoạn code Python sau vào editor:

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

4. Nhấp nút **"Deploy"** để triển khai code

#### Chi tiết chức năng ResourceDeleter

##### Xử lý ARN và khởi tạo clients

Function `lambda_handler()` thực hiện:

**Parse ARN để xác định region**:

- Tách ARN thành các component
- Khởi tạo AWS clients với đúng region
- Hỗ trợ fallback region nếu không xác định được

**Khởi tạo clients theo region**:

- EC2 client cho instances và VPCs
- S3 client với region detection
- RDS client cho databases và clusters
- FSx client cho file systems

##### Xử lý từng loại tài nguyên

**EC2 Instances**:

- Sử dụng `terminate_instances()` API
- Xử lý instance ID từ ARN
- Ghi log quá trình terminate

**VPC**:

- Kiểm tra VPC existence
- Phát hiện default VPC để tránh xóa nhầm
- Xử lý dependency violations
- Yêu cầu manual cleanup cho VPC phức tạp

**S3 Buckets**:

- Tự động detect bucket region
- Empty bucket contents và versions
- Delete bucket sau khi đã empty
- Xử lý cross-region buckets

**RDS Instances**:

- Skip final snapshot để tăng tốc
- Delete automated backups
- Xử lý đơn giản cho standalone instances

**RDS Clusters**:

- Xóa tất cả instances trong cluster trước
- Chờ instances được xóa hoàn toàn
- Timeout protection (5 phút)
- Xóa cluster sau khi instances đã clean

**FSx File Systems**:

- Direct deletion với file system ID
- Xử lý từ ARN format

##### Xử lý đặc biệt cho RDS Clusters

Function `delete_rds_cluster_instances()` thực hiện:

**Quản lý cluster members**:

- Liệt kê tất cả instances trong cluster
- Xóa từng instance với proper configuration
- Xử lý trường hợp instance đã trong quá trình xóa

**Waiting mechanism**:

- Chờ tất cả instances hoàn tất deletion
- Kiểm tra định kỳ mỗi 15 giây
- Timeout sau 5 phút để tránh infinite loop
- Proper error handling cho cluster not found

##### Logging và Error Handling

Function cung cấp logging toàn diện:

```
[DEBUG] - Chi tiết quá trình xử lý
[INFO] - Thông tin quan trọng về deletion
[WARN] - Cảnh báo về vấn đề tiềm ẩn
[ERROR] - Lỗi xảy ra trong quá trình xóa
[SUCCESS] - Confirmation khi xóa thành công
```

#### Cấu hình bổ sung

##### Timeout và Memory Settings

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
