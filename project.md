# Cloud Cost Optimization

### Go to AWS Management console

Create an Ubuntu Ec2 instance and sure instance is active and running

Go to Snapshot and create Snapshot using created instance

## Go to Lambda console

Create lambda function

Choose Author from scratch

`Function name` - cost-optimization-ebs-snapshot
`Runtime` - python3.12

click Create function

Function overview 

Code

click on Test

#### Configure Event

`Event Name` - test-event

Save it

you do it manully that we if you do it using cloud watch then you don't create test event

now it will fail bydefault lambda execution time 3sec and it is failing soame permission issue
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
Go to configuration tab

default execution is 3 sec
![Screenshot 2024-08-30 234739](https://github.com/user-attachments/assets/c68dbb9d-51b0-4e6c-bdcc-b507fb0e004d)


change it 10 sec
it is better to keep the execution time as small as possibel because aws will charge you using this as a parameter like the lambda execution time is also one of the parameter for charging so make sure that you keep this time as less as possible

Go to code 
go to permission 
rolename go to new tab
add permission to it
new tab
policy
create policy
choose service 
ec2
add
delete snapshot
describe snapshot
describe instance 
describe volumes
resources ARN --- all
Give the policy name
cost-optimization-policy
save the code deploy and test
![Screenshot 2024-08-30 235000](https://github.com/user-attachments/assets/79c7367b-955f-4aa6-8ded-7ce07d238feb)

delete ec2 instance
save code deploy and test
![Screenshot 2024-08-30 235140](https://github.com/user-attachments/assets/286bcc45-cf28-446d-b5e6-1dd9ea16436a)


