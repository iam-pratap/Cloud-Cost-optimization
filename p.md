##  Lambda function will run automatically based on the CloudWatch schedule, cleaning up unused EBS snapshots without needing manual test events. 

### Step 1: Create an EC2 Instance and Snapshot

Go to the AWS Management Console → EC2.

Launch an Ubuntu EC2 Instance.

Ensure the instance is running.

Go to Elastic Block Store (EBS) → Volumes.

Select the volume attached to your EC2 instance.

Click Actions → Create Snapshot → Provide a description → Create Snapshot.

### Step 2: Create the AWS Lambda Function

Go to the AWS Lambda Console.

Click Create function.

Select "Author from scratch".

Enter the Function name: cost-optimization-ebs-snapshot.

Select Runtime: Python 3.10.

Click Create function.

### Step 3: Add the Lambda Function Code

Go to the Code section.

Replace the existing code with the following Python Boto3 script:

python > Copy > Edit
```
import boto3

def lambda_handler(event, context):
    ec2 = boto3.client('ec2')

    # Get all EBS snapshots
    response = ec2.describe_snapshots(OwnerIds=['self'])

    # Get all active EC2 instance IDs
    instances_response = ec2.describe_instances(Filters=[{'Name': 'instance-state-name', 'Values': ['running']}])
    active_instance_ids = set()

    for reservation in instances_response['Reservations']:
        for instance in reservation['Instances']:
            active_instance_ids.add(instance['InstanceId'])

    # Iterate through each snapshot and delete if it's not attached to any volume or the volume is not attached to a running instance
    for snapshot in response['Snapshots']:
        snapshot_id = snapshot['SnapshotId']
        volume_id = snapshot.get('VolumeId')

        if not volume_id:
            # Delete the snapshot if it's not attached to any volume
            ec2.delete_snapshot(SnapshotId=snapshot_id)
            print(f"Deleted EBS snapshot {snapshot_id} as it was not attached to any volume.")
        else:
            # Check if the volume still exists
            try:
                volume_response = ec2.describe_volumes(VolumeIds=[volume_id])
                if not volume_response['Volumes'][0]['Attachments']:
                    ec2.delete_snapshot(SnapshotId=snapshot_id)
                    print(f"Deleted EBS snapshot {snapshot_id} as it was taken from a volume not attached to any running instance.")
            except ec2.exceptions.ClientError as e:
                if e.response['Error']['Code'] == 'InvalidVolume.NotFound':
                    # The volume associated with the snapshot is not found (it might have been deleted)
                    ec2.delete_snapshot(SnapshotId=snapshot_id)
                    print(f"Deleted EBS snapshot {snapshot_id} as its associated volume was not found.")

```

Click Deploy.

### Step 4: Increase Lambda Timeout

Go to the Configuration tab → General Configuration → Click Edit.

Change the Timeout from 3 seconds to 10 seconds → Click Save.

### Step 5: Assign IAM Permissions

Go to the Configuration tab → Permissions.

Click on the IAM Role linked to the Lambda function.

In the new tab, go to Permissions → Add Permissions → Create Policy.

Choose Service: EC2.

Search and add the following actions:
```
DescribeSnapshots
DescribeInstances
DescribeVolumes
DeleteSnapshot
```
Set Resources to All.

Click Next → Provide a Policy Name (cost-optimization-policy) → Create Policy.

Back in the IAM Role, Attach the Policy to your Lambda role.

### Step 6: Automate with CloudWatch Event Rule

Go to the AWS CloudWatch Console.

Click Rules → Create Rule.

Choose Event Source: Schedule.

Set a schedule expression:

Example: Run daily → rate(1 day)

Example: Run every 6 hours → rate(6 hours) → Click Next.

Choose Target: AWS Lambda Function.

Select your Lambda function (cost-optimization-ebs-snapshot).

Click Next → Create Rule.

### Step 7: Verify Execution

Wait for the scheduled time or manually trigger the CloudWatch rule.

Go to the Lambda Function → Monitor → Logs.

Check if snapshots are deleted as expected.

## Congratulations!!!
