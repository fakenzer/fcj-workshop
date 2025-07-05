---
title: "Tạo Lambda Function ResourceScanner"
weight: 1
chapter: false
pre: " <b> 3.1. </b> "
---

Trong phần này, chúng ta sẽ tạo Lambda function **ResourceScanner** để quét và kiểm tra tags của tài nguyên AWS.

## Lambda Function: ResourceScanner

#### Mục đích

Function này sẽ thực hiện các chức năng quan trọng:

- Quét tất cả tài nguyên có tags trong AWS account
- Kiểm tra tài nguyên có tồn tại hay đã bị xóa
- Xác định tài nguyên thiếu required tags
- Phân loại tài nguyên theo Environment để tự động xóa
- Hỗ trợ whitelist để bảo vệ tài nguyên quan trọng

#### Các loại tài nguyên được hỗ trợ

ResourceScanner kiểm tra các loại tài nguyên sau:

- **EC2 instances**: Kiểm tra trạng thái running/stopped
- **VPC**: Kiểm tra trạng thái available/deleting
- **S3 buckets**: Kiểm tra bucket existence
- **RDS instances**: Kiểm tra DB instance status
- **RDS clusters**: Kiểm tra cluster status
- **FSx file systems**: Kiểm tra filesystem lifecycle

#### Các bước tạo Lambda Function

##### Bước 1: Truy cập Lambda Console

1. Đăng nhập vào AWS Console
2. Tìm kiếm **"Lambda"** trong thanh tìm kiếm
3. Chọn dịch vụ Lambda

   ![Choose lambda](/images/3.Lambda/001-lambda.png)

##### Bước 2: Tạo Function mới

1. Nhấp vào nút **"Create function"**
2. Chọn **"Author from scratch"**

##### Bước 3: Cấu hình Function

**Function configuration**:

- **Function name**: `ResourceScanner`
- **Runtime**: `Python 3.10`

**Execution role**:

1. Tìm và nhấp vào mục **"Change default execution role"**
2. Chọn **"Use an existing role"**
3. Trong dropdown, chọn role **"ResourceManagerRole"** đã tạo trước đó
   ![Create Scanner Function](/images/3.Lambda/002-createscanner.png)

##### Bước 4: Deploy Code

1. Nhấp **"Create function"** để tạo Lambda function
2. Xóa code mẫu hiện có trong editor
3. Paste đoạn code Python sau vào editor:

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

4. Nhấp nút **"Deploy"** để triển khai code

#### Chi tiết chức năng ResourceScanner

##### Phân tích ARN và kiểm tra tài nguyên

Function `resource_exists()` thực hiện:

**Parsing ARN**:

- Tách ARN thành các component: service, region, account, resource
- Xác định loại tài nguyên cần kiểm tra

**Kiểm tra từng loại tài nguyên**:

- **EC2 instances**: Kiểm tra state không phải `terminated` hoặc `shutting-down`
- **VPC**: Kiểm tra state không phải `deleting` hoặc `deleted`
- **S3 buckets**: Kiểm tra bucket existence với `head_bucket()`
- **RDS instances/clusters**: Kiểm tra status không phải `deleting` hoặc `deleted`
- **FSx file systems**: Kiểm tra lifecycle không phải `DELETING`, `DELETED`, hoặc `FAILED`

##### Xử lý policies và whitelist

Function `lambda_handler()` thực hiện:

**Load configuration từ SSM**:

- `/resource-management/policies`: Chứa required tags và auto-delete environments
- `/resource-management/whitelist`: Danh sách ARN được bảo vệ

**Phân loại tài nguyên**:

- **SEND_WARNING**: Thiếu required tags
- **DELETE_IMMEDIATE**: Environment thuộc auto-delete list
- **NO_ACTION**: Tài nguyên compliant

##### Logging và monitoring

Function cung cấp logging chi tiết:

```
[DEBUG] - Thông tin debug
[INFO] - Thông tin quan trọng
[WARN] - Cảnh báo
[ERROR] - Lỗi xảy ra
[RESULT] - Kết quả xử lý từng tài nguyên
[SUMMARY] - Tổng kết quá trình scan
```

#### Cấu hình bổ sung

##### Timeout và Memory

![Choose lambda](/images/3.Lambda/003-configurationtimeout.png)
Đối với môi trường production, cần điều chỉnh:

1. **Timeout**: Tăng lên 10-15 phút cho account lớn
2. **Memory**: Tăng lên 512MB hoặc 1GB
3. **Environment variables**: Cấu hình region, logging level

## Kết quả

Sau khi hoàn thành phần này, bạn sẽ có:

1. **ResourceScanner Lambda Function** với khả năng:

   - Quét các tagged resources đã nêu trong AWS account
   - Kiểm tra resource existence và state
   - Phân loại tài nguyên theo compliance policies
   - Hỗ trợ whitelist để bảo vệ critical resources
   - Logging chi tiết cho monitoring và debugging

2. **Tích hợp với service**:
   - SSM Parameter Store để load policiess

Lambda function này sẽ là component cốt lõi trong workflow resource management và cost optimization.
