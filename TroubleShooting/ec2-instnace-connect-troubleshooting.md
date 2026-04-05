I'll create a comprehensive README file documenting all the troubleshooting steps we went through to fix your EC2 Instance Connect issue.

# EC2 Instance Connect Troubleshooting Guide

## Problem Description
Getting "Failed to connect to your instance - Error establishing SSH connection to your instance. Try again later." when trying to use EC2 Instance Connect on an Elastic Beanstalk managed EC2 instance.

## Instance Details
- **Instance ID**: i-0fc21c50935795c47
- **Region**: ap-south-2 (Asia Pacific - Hyderabad)
- **Instance Type**: t3.micro
- **AMI**: Amazon Linux 2023
- **Environment**: Elastic Beanstalk managed instance
- **Public IP**: 16.112.9.35 (IPv4)

## Root Cause
The security group was configured with the **IPv6** EC2 Instance Connect prefix list (`pl-07e2569258d218a61`) instead of the **IPv4** prefix list, while the instance was using IPv4 networking.

## Troubleshooting Steps Performed

### Step 1: Initial Security Group Check
```bash
# Check instance details and security group configuration
aws ec2 describe-instances --instance-ids i-0fc21c50935795c47 --region ap-south-2

Finding: Security group sg-08dc9e3caf0f9a626 had SSH port 22 open to prefix list pl-07e2569258d218a61

Step 2: Instance Health Verification
# Verify instance status and health checks
aws ec2 describe-instance-status --instance-ids i-0fc21c50935795c47 --region ap-south-2
```

Finding: All status checks were passing ✅

System Status Check: Passed
Instance Status Check: Passed
EBS Status: Passed
Step 3: Alternative Access Method
Since EC2 Instance Connect wasn't working, we used AWS Systems Manager Session Manager to access the instance:

Navigate to Systems Manager Session Manager 
Select instance i-0fc21c50935795c47
Click "Start session"
Result: Successfully accessed the instance ✅

Step 4: SSH Service Verification
```bash
Inside the instance via Session Manager:

# Check if SSH service is running
sudo systemctl status sshd

# Check SSH configuration
sudo grep -E "Port|PasswordAuthentication|PubkeyAuthentication" /etc/ssh/sshd_config
```

Finding: SSH service was running and properly configured ✅

Step 5: EC2 Instance Connect Package Check

```bash
# Check if EC2 Instance Connect is installed
sudo yum list installed | grep ec2-instance-connect

Run in CloudShell
Output:

ec2-instance-connect.noarch            1.1-19.amzn2023                    @System
ec2-instance-connect-selinux.noarch    1.1-19.amzn2023                    @System
```

Finding: EC2 Instance Connect package was installed ✅

Step 6: Service Status Check

```bash
# Check if EC2 Instance Connect service is running
sudo systemctl status ec2-instance-connect

Output: Unit ec2-instance-connect.service could not be found.
```

Note: This is normal for newer versions - EC2 Instance Connect integrates directly with SSH, not as a separate service.

Step 7: SSH Configuration for EC2 Instance Connect
```bash
# Check SSH configuration for EC2 Instance Connect
sudo grep -E "AuthorizedKeysCommand|AuthorizedKeysCommandUser" /etc/ssh/sshd_config

# Verify the script exists
ls -la /opt/aws/bin/eic_run_authorized_keys


Finding: Configuration was already correct ✅

AuthorizedKeysCommand /opt/aws/bin/eic_run_authorized_keys %u %f
AuthorizedKeysCommandUser ec2-instance-connect
```

Step 8: SELinux Check
```bash
# Check SELinux status
sudo getenforce

Output: Permissive

Finding: SELinux was not blocking connections ✅
```

Step 9: Connection Attempt Monitoring
```bash
# Monitor SSH logs during connection attempts
sudo tail -f /var/log/secure

Finding: No log entries appeared when attempting EC2 Instance Connect, indicating traffic wasn't reaching the instance.
```

Step 10: Security Group Deep Dive
```bash
# Get detailed security group information
aws ec2 describe-security-groups --group-ids sg-08dc9e3caf0f9a626 --region ap-south-2

# Check what the prefix list contains
aws ec2 describe-managed-prefix-lists --prefix-list-ids pl-07e2569258d218a61 --region ap-south-2

Critical Finding: The prefix list was for IPv6 EC2 Instance Connect:

{
    "PrefixListId": "pl-07e2569258d218a61",
    "AddressFamily": "IPv6",
    "PrefixListName": "com.amazonaws.ap-south-2.ipv6.ec2-instance-connect"
}
```
But the instance was using IPv4 networking!

Solution
Add IPv4 EC2 Instance Connect Prefix List
Method 1: AWS CLI
```bash
# Find the correct IPv4 prefix list
aws ec2 describe-managed-prefix-lists \
    --filters "Name=prefix-list-name,Values=com.amazonaws.ap-south-2.ec2-instance-connect" \
    --region ap-south-2

# Add IPv4 EC2 Instance Connect access to security group
aws ec2 authorize-security-group-ingress \
    --group-id sg-08dc9e3caf0f9a626 \
    --protocol tcp \
    --port 22 \
    --prefix-list-id pl-0b28aa2cc0e5f0e7b \
    --region ap-south-2
```
Method 2: AWS Console

Go to Security Groups 
Select security group sg-08dc9e3caf0f9a626
Click "Edit inbound rules"
Click "Add rule"
Configure:
Type: SSH
Protocol: TCP
Port Range: 22
Source: Prefix list
Prefix list: com.amazonaws.ap-south-2.ec2-instance-connect (IPv4 version)
Click "Save rules"
Result
✅ EC2 Instance Connect now works successfully!

Key Lessons Learned
IP Version Matching: Always ensure the EC2 Instance Connect prefix list matches your instance's IP version:

IPv4 instances → Use com.amazonaws.region.ec2-instance-connect
IPv6 instances → Use com.amazonaws.region.ipv6.ec2-instance-connect
Troubleshooting Approach: When EC2 Instance Connect fails:

Check security group configuration first
Verify instance health and status checks
Use Session Manager as alternative access method
Check SSH service and EC2 Instance Connect package
Monitor logs during connection attempts
Investigate network-level issues if no logs appear
Elastic Beanstalk Considerations:

Elastic Beanstalk instances may not have SSH enabled by default
Session Manager is often the preferred access method for managed instances
Security groups are managed by CloudFormation and may reset
Prevention
To avoid this issue in the future:

Always verify the IP version (IPv4/IPv6) of your prefix lists
Use descriptive names when creating security group rules
Test EC2 Instance Connect immediately after instance creation
Document security group configurations for managed services
Useful Commands Reference
```bash
# Check instance details
aws ec2 describe-instances --instance-ids <instance-id> --region <region>

# Check security groups
aws ec2 describe-security-groups --group-ids <sg-id> --region <region>

# List EC2 Instance Connect prefix lists
aws ec2 describe-managed-prefix-lists \
    --filters "Name=prefix-list-name,Values=*ec2-instance-connect*" \
    --region <region>

# Check SSH service status
sudo systemctl status sshd

# Monitor SSH connection attempts
sudo tail -f /var/log/secure

# Check EC2 Instance Connect package
sudo yum list installed | grep ec2-instance-connect
```

Additional Resources
EC2 Instance Connect Prerequisites 
Troubleshoot EC2 Instance Connect 
AWS Systems Manager Session Manager 
EC2 Security Groups