# Terraform ASG standby and maintainance repo

This is Proof of concept for ASG standby and maintenance.

# Start by using terraform

```bash
terraform init
terraform plan
echo yes | terraform apply
```

# ssh to the instance and create a file, assuming you have a public key called "chenglimteo"

```bash
ssh ubuntu@[public_ip]
dd if=/dev/zero of=1GB_file bs=1M count=1024
```

# "ASG Min" must be less than desired capacity so that we can detach from ASG

```bash
$ aws autoscaling enter-standby \
>     --instance-ids i-05a27f8278dbb608d \
>     --auto-scaling-group-name terraform-20250621044320415900000001 \
>     --should-decrement-desired-capacity

An error occurred (ValidationError) when calling the EnterStandby operation: AutoScalingGroup terraform-20250621044320415900000001 has min-size=1, max-size=1, and desired-size=1. To place into standby 1 instance, please update the AutoScalingGroup sizes appropriately.
```

# https://docs.aws.amazon.com/autoscaling/ec2/userguide/as-enter-exit-standby.html

1. Move out the instance

```bash
aws autoscaling enter-standby \
    --instance-ids i-05a27f8278dbb608d \
    --auto-scaling-group-name terraform-20250621044320415900000001 \
    --should-decrement-desired-capacity
{
    "Activities": [
        {
            "ActivityId": "f3a65e43-4f08-ffcb-1051-7744a312c18c",
            "AutoScalingGroupName": "terraform-20250621044320415900000001",
            "Description": "Moving EC2 instance to Standby: i-05a27f8278dbb608d",
            "Cause": "At 2025-06-21T04:51:26Z instance i-05a27f8278dbb608d was moved to standby in response to a user request, shrinking the capacity from 1 to 0.",
            "StartTime": "2025-06-21T04:51:26.399000+00:00",
            "StatusCode": "InProgress",
            "Progress": 50,
            "Details": "{\"Subnet ID\":\"subnet-09222ab4f60b58fdf\",\"Availability Zone\":\"ap-southeast-1a\"}"
        }
    ]
}
```

2. Confirmed the state is in Standby before we do anything

```bash
aws autoscaling describe-auto-scaling-instances --instance-ids i-05a27f8278dbb608d
{
    "AutoScalingInstances": [
        {
            "InstanceId": "i-05a27f8278dbb608d",
            "InstanceType": "t2.micro",
            "AutoScalingGroupName": "terraform-20250621044320415900000001",
            "AvailabilityZone": "ap-southeast-1a",
            "LifecycleState": "Standby",
            "HealthStatus": "HEALTHY",
            "LaunchTemplate": {
                "LaunchTemplateId": "lt-06c8a59ca43495c63",
                "LaunchTemplateName": "asg-template-20250621044022820100000002",
                "Version": "1"
            },
            "ProtectedFromScaleIn": false
        }
    ]
}
```

3. Do whatever you want to do (Stopped/Resize/Start)

# ssh to the instance and create a file, assuming you have a public key called "chenglimteo"

```bash
ssh ubuntu@[public_ip]
ubuntu@ip-10-13-1-220:~$ ls -la
total 1048612
drwxr-x--- 4 ubuntu ubuntu       4096 Jun 21 04:49 .
drwxr-xr-x 3 root   root         4096 Jun 21 04:47 ..
-rw-rw-r-- 1 ubuntu ubuntu 1073741824 Jun 21 04:49 1GB_file
-rw------- 1 ubuntu ubuntu         53 Jun 21 04:49 .bash_history
-rw-r--r-- 1 ubuntu ubuntu        220 Jan  6  2022 .bash_logout
-rw-r--r-- 1 ubuntu ubuntu       3771 Jan  6  2022 .bashrc
drwx------ 2 ubuntu ubuntu       4096 Jun 21 04:49 .cache
-rw-r--r-- 1 ubuntu ubuntu        807 Jan  6  2022 .profile
drwx------ 2 ubuntu ubuntu       4096 Jun 21 04:47 .ssh
```

4. Rejoin (exit-standby) ASG

```bash
aws autoscaling exit-standby \
    --instance-ids i-05a27f8278dbb608d \
    --auto-scaling-group-name terraform-20250621044320415900000001
{
    "Activities": [
        {
            "ActivityId": "a1a65e43-5fb1-1824-f5f7-e5e7ff219fd7",
            "AutoScalingGroupName": "terraform-20250621044320415900000001",
            "Description": "Moving EC2 instance out of Standby: i-05a27f8278dbb608d",
            "Cause": "At 2025-06-21T04:55:59Z instance i-05a27f8278dbb608d was moved out of standby in response to a user request, increasing the capacity from 0 to 1.",
            "StartTime": "2025-06-21T04:55:59.302000+00:00",
            "StatusCode": "PreInService",
            "Progress": 30,
            "Details": "{\"Subnet ID\":\"subnet-09222ab4f60b58fdf\",\"Availability Zone\":\"ap-southeast-1a\"}"
        }
    ]
}
```

5. Confirmed the state is in InService

```bash
aws autoscaling describe-auto-scaling-instances --instance-ids i-05a27f8278dbb608d
{
    "AutoScalingInstances": [
        {
            "InstanceId": "i-05a27f8278dbb608d",
            "InstanceType": "t2.micro",
            "AutoScalingGroupName": "terraform-20250621044320415900000001",
            "AvailabilityZone": "ap-southeast-1a",
            "LifecycleState": "Pending",
            "HealthStatus": "HEALTHY",
            "LaunchTemplate": {
                "LaunchTemplateId": "lt-06c8a59ca43495c63",
                "LaunchTemplateName": "asg-template-20250621044022820100000002",
                "Version": "1"
            },
            "ProtectedFromScaleIn": false
        }
    ]
}

{
    "AutoScalingInstances": [
        {
            "InstanceId": "i-05a27f8278dbb608d",
            "InstanceType": "t2.micro",
            "AutoScalingGroupName": "terraform-20250621044320415900000001",
            "AvailabilityZone": "ap-southeast-1a",
            "LifecycleState": "InService",
            "HealthStatus": "HEALTHY",
            "LaunchTemplate": {
                "LaunchTemplateId": "lt-06c8a59ca43495c63",
                "LaunchTemplateName": "asg-template-20250621044022820100000002",
                "Version": "1"
            },
            "ProtectedFromScaleIn": false
        }
    ]
}
```

# clean up with terraform

```bash
echo yes | terraform destroy

```