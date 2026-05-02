# 44 — AWS Cloud Deep Dive

> **[← Index](00_INDEX.md)** | **Related: [Cloud & Remote Access](17_Cloud_Remote_Access.md) · [IaC Terraform Ansible](28_IaC_Terraform_Ansible.md) · [Docker & Containers](30_Docker_Containers.md) · [Kubernetes Deep Dive](41_Kubernetes_Deep_Dive.md) · [Security Concepts](14_Security_Concepts.md)**

---

## AWS Global Infrastructure

```mermaid
graph TD
    subgraph REGION["AWS Region (e.g. ap-south-1 Mumbai)"]
        subgraph AZ1["Availability Zone 1\n(ap-south-1a)"]
            EC2A["EC2 instances"]
            RDSA["RDS Primary"]
            SUBA["Subnet A\n10.0.1.0/24"]
        end
        subgraph AZ2["Availability Zone 2\n(ap-south-1b)"]
            EC2B["EC2 instances"]
            RDSB["RDS Standby"]
            SUBB["Subnet B\n10.0.2.0/24"]
        end
        VPC["VPC\n10.0.0.0/16"]
        ALB["ALB\n(Application Load Balancer)"]
        S3["S3 Bucket\n(regional)"]
        CF["CloudFront CDN\n(global edge)"]
    end

    INTERNET["Internet"] --> CF --> ALB
    ALB --> EC2A & EC2B
    EC2A & EC2B --> RDSA
    RDSA -.->|"sync replication"| RDSB
```

---

## AWS CLI — Setup & Essentials

```bash
# Install
pip install awscli --upgrade
# or
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o aws.zip
unzip aws.zip && sudo ./aws/install

# Configure credentials
aws configure
# AWS Access Key ID: AKIAXXXXXXXXXXXXXXXX
# AWS Secret Access Key: xxxx
# Default region: ap-south-1
# Default output format: json

# Multiple profiles
aws configure --profile production
aws configure --profile staging

# Use profile
aws s3 ls --profile production
export AWS_PROFILE=production   # Set default for session

# Using IAM roles (on EC2/Lambda — no keys needed)
aws sts get-caller-identity     # Who am I?
aws sts get-session-token       # Temporary credentials

# Useful global flags
--region ap-south-1             # Override region
--profile production            # Override profile
--output table/json/text/yaml  # Output format
--query "Instances[0].InstanceId"  # JMESPath filter
--dry-run                       # Test without executing (some commands)
```

---

## VPC — Virtual Private Cloud

A VPC is your isolated private network within AWS.

```bash
# ── Create VPC ────────────────────────────────────────
VPC_ID=$(aws ec2 create-vpc \
    --cidr-block 10.0.0.0/16 \
    --tag-specifications 'ResourceType=vpc,Tags=[{Key=Name,Value=production-vpc}]' \
    --query 'Vpc.VpcId' --output text)

# Enable DNS
aws ec2 modify-vpc-attribute --vpc-id $VPC_ID --enable-dns-hostnames
aws ec2 modify-vpc-attribute --vpc-id $VPC_ID --enable-dns-support

# ── Subnets ───────────────────────────────────────────
# Public subnets (for ALB, NAT Gateway, Bastion)
PUB_A=$(aws ec2 create-subnet \
    --vpc-id $VPC_ID \
    --cidr-block 10.0.1.0/24 \
    --availability-zone ap-south-1a \
    --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=public-1a}]' \
    --query 'Subnet.SubnetId' --output text)

PUB_B=$(aws ec2 create-subnet \
    --vpc-id $VPC_ID \
    --cidr-block 10.0.2.0/24 \
    --availability-zone ap-south-1b \
    --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=public-1b}]' \
    --query 'Subnet.SubnetId' --output text)

# Private subnets (for EC2, RDS)
PRIV_A=$(aws ec2 create-subnet \
    --vpc-id $VPC_ID \
    --cidr-block 10.0.11.0/24 \
    --availability-zone ap-south-1a \
    --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=private-1a}]' \
    --query 'Subnet.SubnetId' --output text)

# Enable auto-assign public IP for public subnets
aws ec2 modify-subnet-attribute --subnet-id $PUB_A --map-public-ip-on-launch

# ── Internet Gateway ──────────────────────────────────
IGW_ID=$(aws ec2 create-internet-gateway \
    --query 'InternetGateway.InternetGatewayId' --output text)
aws ec2 attach-internet-gateway --vpc-id $VPC_ID --internet-gateway-id $IGW_ID

# ── Route Tables ──────────────────────────────────────
# Public route table
PUB_RT=$(aws ec2 create-route-table --vpc-id $VPC_ID --query 'RouteTable.RouteTableId' --output text)
aws ec2 create-route --route-table-id $PUB_RT \
    --destination-cidr-block 0.0.0.0/0 --gateway-id $IGW_ID
aws ec2 associate-route-table --route-table-id $PUB_RT --subnet-id $PUB_A
aws ec2 associate-route-table --route-table-id $PUB_RT --subnet-id $PUB_B

# ── NAT Gateway (for private subnets to reach internet) ──
EIP=$(aws ec2 allocate-address --domain vpc --query 'AllocationId' --output text)
NAT_GW=$(aws ec2 create-nat-gateway \
    --subnet-id $PUB_A \
    --allocation-id $EIP \
    --query 'NatGateway.NatGatewayId' --output text)
# Wait for NAT GW to become available (2-3 min)
aws ec2 wait nat-gateway-available --nat-gateway-ids $NAT_GW

# Private route table
PRIV_RT=$(aws ec2 create-route-table --vpc-id $VPC_ID --query 'RouteTable.RouteTableId' --output text)
aws ec2 create-route --route-table-id $PRIV_RT \
    --destination-cidr-block 0.0.0.0/0 --nat-gateway-id $NAT_GW
aws ec2 associate-route-table --route-table-id $PRIV_RT --subnet-id $PRIV_A

# ── Security Groups ───────────────────────────────────
# Web server SG
WEB_SG=$(aws ec2 create-security-group \
    --group-name web-sg \
    --description "Web server security group" \
    --vpc-id $VPC_ID \
    --query 'GroupId' --output text)

aws ec2 authorize-security-group-ingress --group-id $WEB_SG \
    --protocol tcp --port 80 --cidr 0.0.0.0/0
aws ec2 authorize-security-group-ingress --group-id $WEB_SG \
    --protocol tcp --port 443 --cidr 0.0.0.0/0
aws ec2 authorize-security-group-ingress --group-id $WEB_SG \
    --protocol tcp --port 22 --cidr 10.0.0.0/8   # SSH from VPN only

# Database SG (only allow from web SG)
DB_SG=$(aws ec2 create-security-group \
    --group-name db-sg \
    --description "Database security group" \
    --vpc-id $VPC_ID \
    --query 'GroupId' --output text)

aws ec2 authorize-security-group-ingress --group-id $DB_SG \
    --protocol tcp --port 3306 --source-group $WEB_SG
```

---

## EC2 — Elastic Compute Cloud

```bash
# ── Key pairs ─────────────────────────────────────────
aws ec2 create-key-pair \
    --key-name production-key \
    --query 'KeyMaterial' \
    --output text > ~/.ssh/production-key.pem
chmod 600 ~/.ssh/production-key.pem

# ── Find AMI ──────────────────────────────────────────
# Latest Ubuntu 22.04 in ap-south-1
aws ec2 describe-images \
    --owners 099720109477 \
    --filters "Name=name,Values=ubuntu/images/hvm-ssd/ubuntu-jammy-22.04-amd64-server-*" \
              "Name=state,Values=available" \
    --query 'sort_by(Images, &CreationDate)[-1].ImageId' \
    --output text

# ── Launch instance ───────────────────────────────────
INSTANCE_ID=$(aws ec2 run-instances \
    --image-id ami-0f5ee92e2d63afc18 \
    --instance-type t3.medium \
    --key-name production-key \
    --subnet-id $PRIV_A \
    --security-group-ids $WEB_SG \
    --iam-instance-profile Name=WebServerRole \
    --user-data file://userdata.sh \
    --block-device-mappings '[{"DeviceName":"/dev/sda1","Ebs":{"VolumeSize":30,"VolumeType":"gp3","Encrypted":true}}]' \
    --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=web-01},{Key=Environment,Value=production}]' \
    --metadata-options HttpTokens=required \
    --count 1 \
    --query 'Instances[0].InstanceId' \
    --output text)

# ── Instance operations ───────────────────────────────
aws ec2 describe-instances \
    --query 'Reservations[*].Instances[*].[InstanceId,State.Name,PublicIpAddress,PrivateIpAddress,Tags[?Key==`Name`].Value|[0]]' \
    --output table

aws ec2 stop-instances --instance-ids $INSTANCE_ID
aws ec2 start-instances --instance-ids $INSTANCE_ID
aws ec2 reboot-instances --instance-ids $INSTANCE_ID
aws ec2 terminate-instances --instance-ids $INSTANCE_ID

# Wait for state
aws ec2 wait instance-running --instance-ids $INSTANCE_ID

# ── Connect via SSM (no SSH needed) ───────────────────
aws ssm start-session --target $INSTANCE_ID

# ── Auto Scaling Group ────────────────────────────────
# Launch template first
aws ec2 create-launch-template \
    --launch-template-name web-lt \
    --version-description "v1" \
    --launch-template-data '{
        "ImageId": "ami-0f5ee92e2d63afc18",
        "InstanceType": "t3.medium",
        "KeyName": "production-key",
        "SecurityGroupIds": ["'$WEB_SG'"],
        "IamInstanceProfile": {"Name": "WebServerRole"},
        "UserData": "'$(base64 -w0 userdata.sh)'"
    }'

# Create ASG
aws autoscaling create-auto-scaling-group \
    --auto-scaling-group-name web-asg \
    --launch-template LaunchTemplateName=web-lt,Version='$Latest' \
    --min-size 2 \
    --max-size 10 \
    --desired-capacity 2 \
    --vpc-zone-identifier "$PRIV_A,$PRIV_B" \
    --target-group-arns $TG_ARN \
    --health-check-type ELB \
    --health-check-grace-period 300 \
    --tags Key=Name,Value=web-asg,PropagateAtLaunch=true

# Scaling policies
aws autoscaling put-scaling-policy \
    --auto-scaling-group-name web-asg \
    --policy-name cpu-tracking \
    --policy-type TargetTrackingScaling \
    --target-tracking-configuration '{
        "PredefinedMetricSpecification": {
            "PredefinedMetricType": "ASGAverageCPUUtilization"
        },
        "TargetValue": 70.0
    }'
```

---

## S3 — Simple Storage Service

```bash
# ── Bucket operations ─────────────────────────────────
# Create bucket
aws s3 mb s3://my-unique-bucket-name --region ap-south-1

# List buckets
aws s3 ls

# List contents
aws s3 ls s3://my-bucket/
aws s3 ls s3://my-bucket/prefix/ --recursive --human-readable

# ── Object operations ─────────────────────────────────
# Upload
aws s3 cp local-file.txt s3://my-bucket/remote-file.txt
aws s3 cp local-file.txt s3://my-bucket/ --storage-class STANDARD_IA
aws s3 cp large-file.zip s3://my-bucket/ --expected-size 1073741824  # Multipart

# Download
aws s3 cp s3://my-bucket/file.txt ./local-file.txt

# Sync (like rsync for S3)
aws s3 sync ./local-dir/ s3://my-bucket/remote-dir/ --delete
aws s3 sync s3://my-bucket/ ./local-backup/ --exclude "*.tmp"

# Delete
aws s3 rm s3://my-bucket/file.txt
aws s3 rm s3://my-bucket/prefix/ --recursive

# ── Permissions ───────────────────────────────────────
# Block all public access (recommended)
aws s3api put-public-access-block \
    --bucket my-bucket \
    --public-access-block-configuration \
    "BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true"

# Bucket policy (allow CloudFront only)
aws s3api put-bucket-policy --bucket my-bucket --policy '{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": {"Service": "cloudfront.amazonaws.com"},
    "Action": "s3:GetObject",
    "Resource": "arn:aws:s3:::my-bucket/*",
    "Condition": {
      "StringEquals": {
        "AWS:SourceArn": "arn:aws:cloudfront::ACCOUNT_ID:distribution/DIST_ID"
      }
    }
  }]
}'

# ── Versioning & lifecycle ────────────────────────────
aws s3api put-bucket-versioning \
    --bucket my-bucket \
    --versioning-configuration Status=Enabled

# Lifecycle rules
aws s3api put-bucket-lifecycle-configuration --bucket my-bucket --lifecycle-configuration '{
  "Rules": [{
    "ID": "transition-and-expire",
    "Status": "Enabled",
    "Filter": {"Prefix": "logs/"},
    "Transitions": [
      {"Days": 30,  "StorageClass": "STANDARD_IA"},
      {"Days": 90,  "StorageClass": "GLACIER"},
      {"Days": 365, "StorageClass": "DEEP_ARCHIVE"}
    ],
    "Expiration": {"Days": 2557},
    "NoncurrentVersionExpiration": {"NoncurrentDays": 30}
  }]
}'

# ── Pre-signed URL (temporary access) ─────────────────
aws s3 presign s3://my-bucket/private-file.pdf --expires-in 3600
# Returns URL valid for 1 hour
```

---

## IAM — Identity and Access Management

```bash
# ── Users ─────────────────────────────────────────────
# Create user
aws iam create-user --user-name alice
aws iam create-access-key --user-name alice   # Programmatic access
aws iam create-login-profile --user-name alice --password "TempPass123!" --password-reset-required

# Attach policy to user
aws iam attach-user-policy \
    --user-name alice \
    --policy-arn arn:aws:iam::aws:policy/ReadOnlyAccess

# ── Groups ────────────────────────────────────────────
aws iam create-group --group-name Developers
aws iam attach-group-policy \
    --group-name Developers \
    --policy-arn arn:aws:iam::aws:policy/AmazonEC2ReadOnlyAccess
aws iam add-user-to-group --user-name alice --group-name Developers

# ── Roles ─────────────────────────────────────────────
# Create role for EC2 instances
aws iam create-role \
    --role-name WebServerRole \
    --assume-role-policy-document '{
        "Version": "2012-10-17",
        "Statement": [{
            "Effect": "Allow",
            "Principal": {"Service": "ec2.amazonaws.com"},
            "Action": "sts:AssumeRole"
        }]
    }'

aws iam attach-role-policy \
    --role-name WebServerRole \
    --policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess

# Create instance profile (attach role to EC2)
aws iam create-instance-profile --instance-profile-name WebServerRole
aws iam add-role-to-instance-profile \
    --instance-profile-name WebServerRole \
    --role-name WebServerRole

# ── Custom policies ───────────────────────────────────
aws iam create-policy \
    --policy-name S3BucketAccess \
    --policy-document '{
        "Version": "2012-10-17",
        "Statement": [{
            "Effect": "Allow",
            "Action": [
                "s3:GetObject",
                "s3:PutObject",
                "s3:DeleteObject",
                "s3:ListBucket"
            ],
            "Resource": [
                "arn:aws:s3:::my-app-bucket",
                "arn:aws:s3:::my-app-bucket/*"
            ]
        }]
    }'

# ── MFA enforcement ───────────────────────────────────
# Policy to require MFA
aws iam create-policy --policy-name RequireMFA --policy-document '{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "AllowViewAccountInfo",
            "Effect": "Allow",
            "Action": ["iam:GetAccountPasswordPolicy","iam:ListVirtualMFADevices"],
            "Resource": "*"
        },
        {
            "Sid": "DenyWithoutMFA",
            "Effect": "Deny",
            "NotAction": ["iam:CreateVirtualMFADevice","iam:EnableMFADevice","iam:GetUser","iam:ListMFADevices","iam:ListVirtualMFADevices","iam:ResyncMFADevice","sts:GetSessionToken"],
            "Resource": "*",
            "Condition": {"BoolIfExists": {"aws:MultiFactorAuthPresent": "false"}}
        }
    ]
}'
```

---

## RDS — Relational Database Service

```bash
# ── Subnet group ──────────────────────────────────────
aws rds create-db-subnet-group \
    --db-subnet-group-name production-db-subnet \
    --db-subnet-group-description "Production database subnets" \
    --subnet-ids $PRIV_A $PRIV_B

# ── Create RDS instance ───────────────────────────────
aws rds create-db-instance \
    --db-instance-identifier prod-mysql \
    --db-instance-class db.t3.medium \
    --engine mysql \
    --engine-version 8.0 \
    --master-username admin \
    --master-user-password "$DB_PASSWORD" \
    --db-name myappdb \
    --allocated-storage 100 \
    --storage-type gp3 \
    --storage-encrypted \
    --vpc-security-group-ids $DB_SG \
    --db-subnet-group-name production-db-subnet \
    --backup-retention-period 7 \
    --preferred-backup-window "02:00-03:00" \
    --preferred-maintenance-window "sun:04:00-sun:05:00" \
    --multi-az \
    --deletion-protection \
    --enable-performance-insights \
    --performance-insights-retention-period 7 \
    --tags Key=Environment,Value=production

# Wait for available
aws rds wait db-instance-available --db-instance-identifier prod-mysql

# ── Snapshots ─────────────────────────────────────────
aws rds create-db-snapshot \
    --db-instance-identifier prod-mysql \
    --db-snapshot-identifier prod-mysql-$(date +%Y%m%d)

aws rds describe-db-snapshots --db-instance-identifier prod-mysql

# Restore from snapshot
aws rds restore-db-instance-from-db-snapshot \
    --db-instance-identifier prod-mysql-restored \
    --db-snapshot-identifier prod-mysql-20240422

# ── Read replica ──────────────────────────────────────
aws rds create-db-instance-read-replica \
    --db-instance-identifier prod-mysql-replica \
    --source-db-instance-identifier prod-mysql \
    --db-instance-class db.t3.small \
    --availability-zone ap-south-1b
```

---

## Lambda — Serverless Functions

```bash
# ── Create Lambda function ────────────────────────────
# Package code
zip function.zip lambda_function.py

# Create function
aws lambda create-function \
    --function-name my-function \
    --runtime python3.12 \
    --zip-file fileb://function.zip \
    --handler lambda_function.handler \
    --role arn:aws:iam::ACCOUNT_ID:role/lambda-role \
    --timeout 30 \
    --memory-size 256 \
    --environment Variables='{DB_HOST=rds.example.com,ENV=production}' \
    --vpc-config SubnetIds=$PRIV_A,SecurityGroupIds=$WEB_SG \
    --tags Environment=production

# Update code
aws lambda update-function-code \
    --function-name my-function \
    --zip-file fileb://function.zip

# Invoke function
aws lambda invoke \
    --function-name my-function \
    --payload '{"key": "value"}' \
    --cli-binary-format raw-in-base64-out \
    response.json
cat response.json

# ── Event triggers ────────────────────────────────────
# S3 trigger (on object upload)
aws lambda add-permission \
    --function-name my-function \
    --statement-id s3-trigger \
    --action lambda:InvokeFunction \
    --principal s3.amazonaws.com \
    --source-arn arn:aws:s3:::my-bucket

aws s3api put-bucket-notification-configuration \
    --bucket my-bucket \
    --notification-configuration '{
        "LambdaFunctionConfigurations": [{
            "LambdaFunctionArn": "arn:aws:lambda:ap-south-1:ACCOUNT:function:my-function",
            "Events": ["s3:ObjectCreated:*"],
            "Filter": {"Key": {"FilterRules": [{"Name": "suffix","Value": ".csv"}]}}
        }]
    }'

# EventBridge (cron) trigger
aws events put-rule \
    --name daily-cleanup \
    --schedule-expression "cron(0 2 * * ? *)" \
    --state ENABLED

aws events put-targets \
    --rule daily-cleanup \
    --targets '[{"Id":"1","Arn":"arn:aws:lambda:ap-south-1:ACCOUNT:function:my-function"}]'

# ── Lambda function example ───────────────────────────
cat > lambda_function.py << 'EOF'
import json, boto3, os, logging

logger = logging.getLogger()
logger.setLevel(logging.INFO)

s3 = boto3.client('s3')
ses = boto3.client('ses')

def handler(event, context):
    logger.info(f"Event: {json.dumps(event)}")

    try:
        # Process S3 event
        for record in event.get('Records', []):
            bucket = record['s3']['bucket']['name']
            key    = record['s3']['object']['key']
            logger.info(f"Processing: s3://{bucket}/{key}")

            # Download file
            response = s3.get_object(Bucket=bucket, Key=key)
            content  = response['Body'].read().decode('utf-8')

            # Process...
            result = process_file(content)

            # Store result
            s3.put_object(
                Bucket=bucket,
                Key=f"processed/{key}",
                Body=json.dumps(result)
            )

        return {'statusCode': 200, 'body': 'Success'}

    except Exception as e:
        logger.error(f"Error: {e}", exc_info=True)
        raise

def process_file(content):
    return {"lines": len(content.splitlines()), "chars": len(content)}
EOF
```

---

## CloudWatch — Monitoring & Logs

```bash
# ── Metrics ───────────────────────────────────────────
aws cloudwatch get-metric-statistics \
    --namespace AWS/EC2 \
    --metric-name CPUUtilization \
    --dimensions Name=InstanceId,Value=$INSTANCE_ID \
    --start-time $(date -u -d '1 hour ago' +%Y-%m-%dT%H:%M:%S) \
    --end-time $(date -u +%Y-%m-%dT%H:%M:%S) \
    --period 300 \
    --statistics Average \
    --output table

# ── Alarms ────────────────────────────────────────────
aws cloudwatch put-metric-alarm \
    --alarm-name "High-CPU-web-01" \
    --alarm-description "CPU > 80% for 10 minutes" \
    --metric-name CPUUtilization \
    --namespace AWS/EC2 \
    --statistic Average \
    --period 300 \
    --evaluation-periods 2 \
    --threshold 80 \
    --comparison-operator GreaterThanThreshold \
    --dimensions Name=InstanceId,Value=$INSTANCE_ID \
    --alarm-actions arn:aws:sns:ap-south-1:ACCOUNT:ops-alerts \
    --ok-actions arn:aws:sns:ap-south-1:ACCOUNT:ops-alerts \
    --treat-missing-data notBreaching

# ── Logs ──────────────────────────────────────────────
# Create log group
aws logs create-log-group --log-group-name /myapp/production
aws logs put-retention-policy --log-group-name /myapp/production --retention-in-days 30

# Query logs
aws logs start-query \
    --log-group-name /myapp/production \
    --start-time $(date -d '1 hour ago' +%s) \
    --end-time $(date +%s) \
    --query-string 'fields @timestamp, @message | filter @message like /ERROR/ | sort @timestamp desc | limit 20'

QUERY_ID=$(aws logs start-query ... --query queryId --output text)
aws logs get-query-results --query-id $QUERY_ID

# CloudWatch Insights example queries
# fields @timestamp, @message | filter @message like "ERROR" | stats count(*) by bin(5m)
# fields @timestamp, level, message | filter level = "ERROR" | sort @timestamp desc
# stats avg(duration), max(duration), p99(duration) by service
```

---

## Route 53 — DNS

```bash
# ── Hosted zones ──────────────────────────────────────
# Create hosted zone
aws route53 create-hosted-zone \
    --name example.com \
    --caller-reference $(date +%s)

ZONE_ID=$(aws route53 list-hosted-zones \
    --query "HostedZones[?Name=='example.com.'].Id" \
    --output text | sed 's|/hostedzone/||')

# ── DNS records ───────────────────────────────────────
# Create/update A record
aws route53 change-resource-record-sets \
    --hosted-zone-id $ZONE_ID \
    --change-batch '{
        "Changes": [{
            "Action": "UPSERT",
            "ResourceRecordSet": {
                "Name": "www.example.com",
                "Type": "A",
                "TTL": 300,
                "ResourceRecords": [{"Value": "93.184.216.34"}]
            }
        }]
    }'

# Alias to ALB (preferred over CNAME for apex domains)
aws route53 change-resource-record-sets \
    --hosted-zone-id $ZONE_ID \
    --change-batch '{
        "Changes": [{
            "Action": "UPSERT",
            "ResourceRecordSet": {
                "Name": "example.com",
                "Type": "A",
                "AliasTarget": {
                    "HostedZoneId": "ZP97RAFLXTNZK",
                    "DNSName": "my-alb-1234.ap-south-1.elb.amazonaws.com",
                    "EvaluateTargetHealth": true
                }
            }
        }]
    }'

# Health check
aws route53 create-health-check \
    --caller-reference $(date +%s) \
    --health-check-config '{
        "IPAddress": "93.184.216.34",
        "Port": 443,
        "Type": "HTTPS",
        "ResourcePath": "/health",
        "FullyQualifiedDomainName": "example.com",
        "RequestInterval": 30,
        "FailureThreshold": 3
    }'
```

---

## AWS Cost Optimization Tips

```bash
# Check current month cost
aws ce get-cost-and-usage \
    --time-period Start=$(date +%Y-%m-01),End=$(date +%Y-%m-%d) \
    --granularity MONTHLY \
    --metrics "UnblendedCost" \
    --query 'ResultsByTime[0].Total.UnblendedCost'

# Find unused resources
# Unattached EBS volumes
aws ec2 describe-volumes \
    --filters Name=status,Values=available \
    --query 'Volumes[*].[VolumeId,Size,CreateTime]' \
    --output table

# Old snapshots (>90 days)
aws ec2 describe-snapshots --owner-ids self \
    --query "Snapshots[?StartTime<='$(date -d '90 days ago' +%Y-%m-%d)'].[SnapshotId,StartTime,VolumeSize]" \
    --output table

# Idle EC2 (< 5% CPU over 2 weeks) — use Cost Explorer or Trusted Advisor
aws support describe-trusted-advisor-check-summaries \
    --check-ids Qch7DwouX   # Low utilization Amazon EC2 instances
```

---

## Related Topics

- [Cloud & Remote Access ←](17_Cloud_Remote_Access.md) — SSH, RDP, VMs
- [IaC Terraform ←](28_IaC_Terraform_Ansible.md) — provision AWS with Terraform
- [Docker & Containers ←](30_Docker_Containers.md) — ECS/EKS
- [Kubernetes Deep Dive ←](41_Kubernetes_Deep_Dive.md) — EKS
- [DNS Deep Dive ←](22_DNS_Deep_Dive.md) — Route 53
- [SSL/TLS ←](26_SSL_TLS_Certificates.md) — ACM (AWS Certificate Manager)
- [Security Concepts ←](14_Security_Concepts.md) — IAM security

---

> [Index](00_INDEX.md)
