# AWS EBS Lab - Complete Guide

A comprehensive hands-on lab for learning Amazon Elastic Block Store (EBS) operations using AWS CLI and CloudShell.

## Table of Contents
- [Prerequisites](#prerequisites)
- [Lab Overview](#lab-overview)
- [Step 1: Environment Setup](#step-1-environment-setup)
- [Step 2: EBS Volume Operations](#step-2-ebs-volume-operations)
- [Step 3: Snapshot Operations](#step-3-snapshot-operations)
- [Step 4: Volume Modifications](#step-4-volume-modifications)
- [Step 5: EC2 Instance Launch](#step-5-ec2-instance-launch)
- [Step 6: Volume Attachment](#step-6-volume-attachment)
- [Step 7: Instance Access](#step-7-instance-access)
- [Step 8: Volume Setup on Instance](#step-8-volume-setup-on-instance)
- [Step 9: Cost Monitoring](#step-9-cost-monitoring)
- [Step 10: Cleanup](#step-10-cleanup)
- [Troubleshooting](#troubleshooting)

## Prerequisites

- AWS Sandbox environment with CloudShell access
- Basic understanding of AWS CLI
- SSH client (for instance access)

## Lab Overview

This lab covers:
- EBS volume creation and management
- Snapshot operations
- Volume encryption and modification
- EC2 instance launch and volume attachment
- Cost monitoring and optimization

## Step 1: Environment Setup

### 1.1 Access CloudShell
```bash
# Open AWS CloudShell from the AWS Console
# Verify your AWS identity
aws sts get-caller-identity
```

### 1.2 Check Current Resources
```bash
# Check existing EC2 instances
aws ec2 describe-instances --query "Reservations[*].Instances[*].{InstanceId:InstanceId,State:State.Name,PublicIpAddress:PublicIpAddress,Tags:Tags}" --output table

# Check existing EBS volumes
aws ec2 describe-volumes --query "Volumes[*].{VolumeId:VolumeId,Size:Size,State:State,AvailabilityZone:AvailabilityZone,Encrypted:Encrypted}" --output table
```

## Step 2: EBS Volume Operations

### 2.1 Create EBS Volume
```bash
# Create a 100 GiB encrypted EBS volume
aws ec2 create-volume \
    --availability-zone eu-north-1a \
    --volume-type gp3 \
    --size 100 \
    --encrypted \
    --tag-specifications 'ResourceType=volume,Tags=[{Key=Name,Value=EBS-Lab-Volume-1}]'
```

### 2.2 Verify Volume Creation
```bash
# Check volume details
aws ec2 describe-volumes --volume-ids vol-0957fbab0b31de0e2

# List all volumes
aws ec2 describe-volumes --query "Volumes[*].{VolumeId:VolumeId,Size:Size,Type:VolumeType,State:State,Encrypted:Encrypted}" --output table
```

## Step 3: Snapshot Operations

### 3.1 Create Snapshot
```bash
# Create snapshot of your volume
aws ec2 create-snapshot \
    --volume-id vol-0957fbab0b31de0e2 \
    --description "EBS Lab Snapshot" \
    --tag-specifications 'ResourceType=snapshot,Tags=[{Key=Name,Value=EBS-Lab-Snapshot-1}]'
```

### 3.2 Monitor Snapshot Status
```bash
# Check snapshot status
aws ec2 describe-snapshots --snapshot-ids snap-04978dd0280d25b2f --query "Snapshots[*].{SnapshotId:SnapshotId,State:State,Progress:Progress}"

# List all snapshots
aws ec2 describe-snapshots --owner-ids self --query "Snapshots[*].{SnapshotId:SnapshotId,VolumeId:VolumeId,State:State,StartTime:StartTime,Description:Description}" --output table
```

## Step 4: Volume Modifications

### 4.1 Modify Volume Type
```bash
# Change volume type from gp3 to io1 with higher IOPS
aws ec2 modify-volume \
    --volume-id vol-0957fbab0b31de0e2 \
    --volume-type io1 \
    --iops 4000
```

### 4.2 Check Modification Status
```bash
# Monitor volume modification progress
aws ec2 describe-volumes --volume-ids vol-0957fbab0b31de0e2 --query "Volumes[*].{VolumeId:VolumeId,VolumeType:VolumeType,Iops:Iops,State:State,ModificationState:ModificationState}"
```

## Step 5: EC2 Instance Launch

### 5.1 Create VPC and Subnet
```bash
# Create VPC
aws ec2 create-vpc \
    --cidr-block 10.0.0.0/24 \
    --tag-specifications 'ResourceType=vpc,Tags=[{Key=Name,Value=EBS-Lab-VPC}]'

# Create subnet
aws ec2 create-subnet \
    --vpc-id vpc-0e62b1241062ea768 \
    --cidr-block 10.0.0.0/28 \
    --availability-zone eu-north-1a \
    --tag-specifications 'ResourceType=subnet,Tags=[{Key=Name,Value=EBS-Lab-Subnet}]'
```

### 5.2 Create Internet Gateway
```bash
# Create internet gateway
aws ec2 create-internet-gateway \
    --tag-specifications 'ResourceType=internet-gateway,Tags=[{Key=Name,Value=EBS-Lab-IGW}]'

# Attach to VPC
aws ec2 attach-internet-gateway \
    --internet-gateway-id igw-03e0643b3a88526f0 \
    --vpc-id vpc-0e62b1241062ea768
```

### 5.3 Create Security Group
```bash
# Create security group
aws ec2 create-security-group \
    --group-name EBS-Lab-SG \
    --description "Security group for EBS lab" \
    --vpc-id vpc-0e62b1241062ea768

# Add SSH rule
aws ec2 authorize-security-group-ingress \
    --group-id sg-04f757ac86fd11272 \
    --protocol tcp \
    --port 22 \
    --cidr 0.0.0.0/0
```

### 5.4 Create Key Pair
```bash
# Create key pair
aws ec2 create-key-pair \
    --key-name EBS-Lab-KeyPair \
    --query 'KeyMaterial' \
    --output text > EBS-Lab-KeyPair.pem

# Set proper permissions
chmod 400 EBS-Lab-KeyPair.pem
```

### 5.5 Launch EC2 Instance
```bash
# Launch instance
aws ec2 run-instances \
    --image-id ami-0854d4f8e4bd6b834 \
    --instance-type t3.micro \
    --key-name EBS-Lab-KeyPair \
    --security-group-ids sg-04f757ac86fd11272 \
    --subnet-id subnet-0bbf492e944e747e3 \
    --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=EBS-Lab-Instance}]'
```

## Step 6: Volume Attachment

### 6.1 Create Elastic IP
```bash
# Allocate Elastic IP
aws ec2 allocate-address --domain vpc
```

### 6.2 Configure Route Table
```bash
# Add route to internet gateway
aws ec2 create-route \
    --route-table-id rtb-0c37a97c83206bc6d \
    --destination-cidr-block 0.0.0.0/0 \
    --gateway-id igw-03e0643b3a88526f0
```

### 6.3 Associate Elastic IP
```bash
# Associate Elastic IP with instance
aws ec2 associate-address \
    --instance-id i-04c9d19997750521d \
    --allocation-id eipalloc-06e45d8f6c6b6396d
```

### 6.4 Attach EBS Volume
```bash
# Attach volume to instance
aws ec2 attach-volume \
    --volume-id vol-0957fbab0b31de0e2 \
    --instance-id i-04c9d19997750521d \
    --device /dev/sdf
```

## Step 7: Instance Access

### 7.1 Check Instance Status
```bash
# Verify instance is running
aws ec2 describe-instances --instance-ids i-04c9d19997750521d --query "Reservations[*].Instances[*].{InstanceId:InstanceId,State:State.Name,PublicIpAddress:PublicIpAddress,PrivateIpAddress:PrivateIpAddress}" --output table
```

### 7.2 SSH to Instance
```bash
# SSH to instance (replace with your public IP)
ssh -i EBS-Lab-KeyPair.pem ec2-user@13.61.218.210
```

## Step 8: Volume Setup on Instance

### 8.1 Check Attached Volumes
```bash
# List block devices
lsblk

# Check volume details
sudo fdisk -l /dev/nvme1n1
```

### 8.2 Format and Mount Volume
```bash
# Format the volume
sudo mkfs.ext4 /dev/nvme1n1

# Create mount point
sudo mkdir /mnt/ebs-volume

# Mount the volume
sudo mount /dev/nvme1n1 /mnt/ebs-volume

# Set permissions
sudo chown ec2-user:ec2-user /mnt/ebs-volume
```

### 8.3 Test Volume
```bash
# Check mounted volume
df -h

# Create test file
echo "Hello from EBS volume!" > /mnt/ebs-volume/test.txt

# Verify file
cat /mnt/ebs-volume/test.txt
```

## Step 9: Cost Monitoring

### 9.1 Check Current Month Costs
```bash
# Get current month costs
aws ce get-cost-and-usage \
    --time-period Start=2025-10-01,End=2025-10-31 \
    --granularity MONTHLY \
    --metrics BlendedCost
```

### 9.2 Get Cost Breakdown by Service
```bash
# Get costs by service
aws ce get-cost-and-usage \
    --time-period Start=2025-10-01,End=2025-10-31 \
    --granularity MONTHLY \
    --metrics BlendedCost \
    --group-by Type=DIMENSION,Key=SERVICE
```

### 9.3 Check EC2 and EBS Costs
```bash
# Check EC2 costs
aws ce get-cost-and-usage \
    --time-period Start=2025-10-01,End=2025-10-31 \
    --granularity MONTHLY \
    --metrics BlendedCost \
    --group-by Type=DIMENSION,Key=SERVICE \
    --filter '{"Dimensions":{"Key":"SERVICE","Values":["Amazon Elastic Compute Cloud - Compute"]}}'

# Check EBS costs
aws ce get-cost-and-usage \
    --time-period Start=2025-10-01,End=2025-10-31 \
    --granularity MONTHLY \
    --metrics BlendedCost \
    --group-by Type=DIMENSION,Key=SERVICE \
    --filter '{"Dimensions":{"Key":"SERVICE","Values":["Amazon Elastic Block Store"]}}'
```

## Step 10: Cleanup

### 10.1 Detach and Delete Volume
```bash
# Detach volume
aws ec2 detach-volume --volume-id vol-0957fbab0b31de0e2

# Delete volume
aws ec2 delete-volume --volume-id vol-0957fbab0b31de0e2
```

### 10.2 Terminate Instance
```bash
# Terminate instance
aws ec2 terminate-instances --instance-ids i-04c9d19997750521d
```

### 10.3 Clean Up Networking
```bash
# Disassociate Elastic IP
aws ec2 disassociate-address --association-id eipassoc-0951b1afe5a2966cc

# Release Elastic IP
aws ec2 release-address --allocation-id eipalloc-06e45d8f6c6b6396d

# Delete security group
aws ec2 delete-security-group --group-id sg-04f757ac86fd11272

# Detach and delete internet gateway
aws ec2 detach-internet-gateway --internet-gateway-id igw-03e0643b3a88526f0 --vpc-id vpc-0e62b1241062ea768
aws ec2 delete-internet-gateway --internet-gateway-id igw-03e0643b3a88526f0

# Delete subnet
aws ec2 delete-subnet --subnet-id subnet-0bbf492e944e747e3

# Delete VPC
aws ec2 delete-vpc --vpc-id vpc-0e62b1241062ea768
```

### 10.4 Delete Snapshots
```bash
# Delete snapshots
aws ec2 delete-snapshot --snapshot-id snap-04978dd0280d25b2f
```

## Troubleshooting

### Common Issues

**1. Instance Launch Fails**
```bash
# Check security group rules
aws ec2 describe-security-groups --group-ids sg-04f757ac86fd11272

# Verify VPC and subnet
aws ec2 describe-vpcs --vpc-ids vpc-0e62b1241062ea768
aws ec2 describe-subnets --subnet-ids subnet-0bbf492e944e747e3
```

**2. SSH Connection Timeout**
```bash
# Check route table
aws ec2 describe-route-tables --filters "Name=vpc-id,Values=vpc-0e62b1241062ea768"

# Verify internet gateway attachment
aws ec2 describe-internet-gateways --internet-gateway-ids igw-03e0643b3a88526f0
```

**3. Volume Attachment Issues**
```bash
# Check volume state
aws ec2 describe-volumes --volume-ids vol-0957fbab0b31de0e2

# Verify instance state
aws ec2 describe-instances --instance-ids i-04c9d19997750521d
```

## Lab Summary

This lab successfully demonstrates:
- ✅ EBS volume creation and management
- ✅ Snapshot operations for backup
- ✅ Volume encryption and modification
- ✅ EC2 instance launch and configuration
- ✅ Volume attachment and filesystem setup
- ✅ Cost monitoring and optimization
- ✅ Proper cleanup procedures

**Total Lab Cost:** $0 (Free Tier eligible)
**Skills Learned:** EBS operations, EC2 management, AWS CLI, cost monitoring

---

## Additional Resources

- [AWS EBS Documentation](https://docs.aws.amazon.com/ebs/)
- [AWS CLI Reference](https://docs.aws.amazon.com/cli/latest/reference/)
- [AWS Free Tier](https://aws.amazon.com/free/)

**Happy Learning! 🚀**
