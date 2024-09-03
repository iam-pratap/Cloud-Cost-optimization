# Cloud Cost Optimization

### AWS Management console

Create an Ubuntu Ec2 instance and make sure instance is active and running

Go to Snapshot and create Snapshot using volume

## Lambda console

Create lambda function

Choose Author from scratch

`Function name` - cost-optimization-ebs-snapshot

`Runtime` - python3.12

Click create function

Go to Code Option and paste this boto3 python code

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

Click on Test

#### Configure Event

`Event Name` - test-event

`Event sharing settings` - Private

Save it

We do this manully, if you do it using cloud watch then you don't create test event

Deploy then test again, it will fail because bydefault **lambda execution time 3sec** and it is failing due to some permission issue.

Output:
```
Test Event Name
test-event

Response
{
  "errorType": "Sandbox.Timedout",
  "errorMessage": "RequestId: 105b7351-cb9a-46a2-ad59-f79ae25e30f0 Error: Task timed out after 3.00 seconds"
}

Function Logs
START RequestId: 105b7351-cb9a-46a2-ad59-f79ae25e30f0 Version: $LATEST
END RequestId: 105b7351-cb9a-46a2-ad59-f79ae25e30f0
REPORT RequestId: 105b7351-cb9a-46a2-ad59-f79ae25e30f0	Duration: 3000.00 ms	Billed Duration: 3000 ms	Memory Size: 128 MB	Max Memory Used: 86 MB	Init Duration: 305.50 ms	Status: timeout

Request ID
105b7351-cb9a-46a2-ad59-f79ae25e30f0
```
Go to configuration then general configuration

Default execution is 3 sec and set this 3 sec to 10 sec and save it

It is better to keep the execution time as small as possible because aws will charge you using this as a parameter like the lambda execution time is also one of the parameter for charging so make sure that you keep this time as less as possible.

Again save, deploy and test

Output should look like:

```
Test Event Name
test-event

Response
{
  "errorMessage": "An error occurred (UnauthorizedOperation) when calling the DescribeSnapshots operation: You are not authorized to perform this operation. User: arn:aws:sts::140146403711:assumed-role/cost-optimization-ebs-snapshot-role-lhvdh5v5/cost-optimization-ebs-snapshot is not authorized to perform: ec2:DescribeSnapshots because no identity-based policy allows the ec2:DescribeSnapshots action",
  "errorType": "ClientError",
  "requestId": "6478857b-b34a-414b-b4ae-4b98d250e353",
  "stackTrace": [
    "  File \"/var/task/lambda_function.py\", line 7, in lambda_handler\n    response = ec2.describe_snapshots(OwnerIds=['self'])\n",
    "  File \"/var/lang/lib/python3.12/site-packages/botocore/client.py\", line 565, in _api_call\n    return self._make_api_call(operation_name, kwargs)\n",
    "  File \"/var/lang/lib/python3.12/site-packages/botocore/client.py\", line 1021, in _make_api_call\n    raise error_class(parsed_response, operation_name)\n"
  ]
}

Function Logs
START RequestId: 6478857b-b34a-414b-b4ae-4b98d250e353 Version: $LATEST
LAMBDA_WARNING: Unhandled exception. The most likely cause is an issue in the function code. However, in rare cases, a Lambda runtime update can cause unexpected function behavior. For functions using managed runtimes, runtime updates can be triggered by a function change, or can be applied automatically. To determine if the runtime has been updated, check the runtime version in the INIT_START log entry. If this error correlates with a change in the runtime version, you may be able to mitigate this error by temporarily rolling back to the previous runtime version. For more information, see https://docs.aws.amazon.com/lambda/latest/dg/runtimes-update.html
[ERROR] ClientError: An error occurred (UnauthorizedOperation) when calling the DescribeSnapshots operation: You are not authorized to perform this operation. User: arn:aws:sts::140146403711:assumed-role/cost-optimization-ebs-snapshot-role-lhvdh5v5/cost-optimization-ebs-snapshot is not authorized to perform: ec2:DescribeSnapshots because no identity-based policy allows the ec2:DescribeSnapshots action
```
Go to code -----> configuration ----> permission -----> Rolename ----> Go to new tab 

Policy ---> create policy ---> choose service ---> ec2 ---> search

Add actions:- describe snapshot, describe instance, describe volumes

Resources ARN ---> all

Give the policy name --> cost-optimization-policy

Search policy name and select then attach and save the code, deploy and test

Output should look like
```
Test Event Name
test-event

Response
null

Function Logs
START RequestId: 770c9765-6cc8-4afd-b78a-c9b9bfdb097a Version: $LATEST
END RequestId: 770c9765-6cc8-4afd-b78a-c9b9bfdb097a
REPORT RequestId: 770c9765-6cc8-4afd-b78a-c9b9bfdb097a	Duration: 3982.99 ms	Billed Duration: 3983 ms	Memory Size: 128 MB	Max Memory Used: 86 MB	Init Duration: 301.22 ms

Request ID
770c9765-6cc8-4afd-b78a-c9b9bfdb097a
```

Now, manually delete the ec2 instance and make sure instance and volume is deleted

Save the code deploy and test

Output should look like:

```
Test Event Name
test-event

Response
{
  "errorMessage": "An error occurred (UnauthorizedOperation) when calling the DeleteSnapshot operation: You are not authorized to perform this operation. User: arn:aws:sts::140146403711:assumed-role/cost-optimization-ebs-snapshot-role-lhvdh5v5/cost-optimization-ebs-snapshot is not authorized to perform: ec2:DeleteSnapshot on resource: arn:aws:ec2:us-east-1::snapshot/snap-0a57f46f6df0dde22 because no identity-based policy allows the ec2:DeleteSnapshot action. Encoded authorization failure message: dxTJylyesiVj6BFRsjYUXzIFua_h5T_NhOWM0xU5daqTd-pdy0Otk0ARze8hjPjunbUIEwcwbRt9UeIqCAFJWv3eFj8zkKxw0ad9bcPCH3iK-jHDz1BpxPMGfTMXAlMaQBH3vtJXI7X75QCj8Fp_RawoCZ4IAaZ4CwLUUhJS5H0eFd2bM9DpvxrjzD7XUeOy9ovLQRTjMCi3cpNmhU4xwIFHRbVhH_p-4BiDKwX3BLImcIcq5-ANNGxJ5KRr3ftnsHswm_Xzy70_qTyGwu6yT7d6xUuGbxdZEyDURDGWaqHJLZmBZ_PdiqKIoVamfbzdW874SBTPiKHYHne1C4ldn1InsPIy0nHNL8Y5P4Drx-wPGM-nAi4EGJqR0UaFMrT7BfVelC7aQSsEaiqp6R5qScqpKDd6KsBMw9M5IE1FkPQM2WEjfU-J_tCFL5FVs9fMA1zbiBdq_Hz2jxSMUr6UGRbdYTTtoiwghYNIHREZHrbUBadMz1cTUyee_yoq285n4_h_GxOFRyugkxDOwn-mqVrWun8-sG1MkWJFKq9Z979pciJAsn-BHJZX2g2rmWEGqtb6PqNmtr2Ak1OzEwkD4AM6nNyxHPy__PC4d-oJl64sFs-6Vha-TB4mjs9DhbFUpWCM0jAAMGlnDILMq_f_tOxc2xtthpMjuwjNZKamczcbsg",
  "errorType": "ClientError",
  "requestId": "262d9b90-8c48-4a8f-b07f-c487f5e221e1",
  "stackTrace": [
    "  File \"/var/task/lambda_function.py\", line 36, in lambda_handler\n    ec2.delete_snapshot(SnapshotId=snapshot_id)\n",
    "  File \"/var/lang/lib/python3.12/site-packages/botocore/client.py\", line 565, in _api_call\n    return self._make_api_call(operation_name, kwargs)\n",
    "  File \"/var/lang/lib/python3.12/site-packages/botocore/client.py\", line 1021, in _make_api_call\n    raise error_class(parsed_response, operation_name)\n"
  ]
}

Function Logs
START RequestId: 262d9b90-8c48-4a8f-b07f-c487f5e221e1 Version: $LATEST
LAMBDA_WARNING: Unhandled exception. The most likely cause is an issue in the function code. However, in rare cases, a Lambda runtime update can cause unexpected function behavior. For functions using managed runtimes, runtime updates can be triggered by a function change, or can be applied automatically. To determine if the runtime has been updated, check the runtime version in the INIT_START log entry. If this error correlates with a change in the runtime version, you may be able to mitigate this error by temporarily rolling back to the previous runtime version. For more information, see https://docs.aws.amazon.com/lambda/latest/dg/runtimes-update.html
[ERROR] ClientError: An error occurred (UnauthorizedOperation) when calling the DeleteSnapshot operation: You are not authorized to perform this operation. User: arn:aws:sts::140146403711:assumed-role/cost-optimization-ebs-snapshot-role-lhvdh5v5/cost-optimization-ebs-snapshot is not authorized to perform: ec2:DeleteSnapshot on resource: arn:aws:ec2:us-east-1::snapshot/snap-0a57f46f6df0dde22 because no identity-based policy allows the ec2:DeleteSnapshot action. Encoded authorization failure message: dxTJylyesiVj6BFRsjYUXzIFua_h5T_NhOWM0xU5daqTd-pdy0Otk0ARze8hjPjunbUIEwcwbRt9UeIqCAFJWv3eFj8zkKxw0ad9bcPCH3iK-jHDz1BpxPMGfTMXAlMaQBH3vtJXI7X75QCj8Fp_RawoCZ4IAaZ4CwLUUhJS5H0eFd2bM9DpvxrjzD7XUeOy9ovLQRTjMCi3cpNmhU4xwIFHRbVhH_p-4BiDKwX3BLImcIcq5-ANNGxJ5KRr3ftnsHswm_Xzy70_qTyGwu6yT7d6xUuGbxdZEyDURDGWaqHJLZmBZ_PdiqKIoVamfbzdW874SBTPiKHYHne1C4ldn1InsPIy0nHNL8Y5P4Drx-wPGM-nAi4EGJqR0UaFMrT7BfVelC7aQSsEaiqp6R5qScqpKDd6KsBMw9M5IE1FkPQM2WEjfU-J_tCFL5FVs9fMA1zbiBdq_Hz2jxSMUr6UGRbdYTTtoiwghYNIHREZHrbUBadMz1cTUyee_yoq285n4_h_GxOFRyugkxDOwn-mqVrWun8-sG1MkWJFKq9Z979pciJAsn-BHJZX2g2rmWEGqtb6PqNmtr2Ak1OzEwkD4AM6nNyxHPy__PC4d-oJl64sFs-6Vha-TB4mjs9DhbFUpWCM0jAAMGlnDILMq_f_tOxc2xtthpMjuwjNZKamczcbsg
Traceback (most recent call last):
  File "/var/task/lambda_function.py", line 36, in lambda_handler
    ec2.delete_snapshot(SnapshotId=snapshot_id)
  File "/var/lang/lib/python3.12/site-packages/botocore/client.py", line 565, in _api_call
    return self._make_api_call(operation_name, kwargs)
  File "/var/lang/lib/python3.12/site-packages/botocore/client.py", line 1021, in _make_api_call
    raise error_class(parsed_response, operation_name)END RequestId: 262d9b90-8c48-4a8f-b07f-c487f5e221e1
REPORT RequestId: 262d9b90-8c48-4a8f-b07f-c487f5e221e1	Duration: 4744.95 ms	Billed Duration: 4745 ms	Memory Size: 128 MB	Max Memory Used: 86 MB	Init Duration: 306.05 ms

Request ID
262d9b90-8c48-4a8f-b07f-c487f5e221e1
```
Add "delete snapshot" action in iam policy

Policy ---> select cost-optimization-policy ---> permissions ---> edit --> visual --> ec2 --> search --> delete snapshot --> next --> save changes

Again deploy and test

Output should look like:
```
Test Event Name
test-event

Response
null

Function Logs
START RequestId: 13d2e237-72b0-4504-979b-8250bb699293 Version: $LATEST
Deleted EBS snapshot snap-0a57f46f6df0dde22 as its associated volume was not found.
END RequestId: 13d2e237-72b0-4504-979b-8250bb699293
REPORT RequestId: 13d2e237-72b0-4504-979b-8250bb699293	Duration: 4694.69 ms	Billed Duration: 4695 ms	Memory Size: 128 MB	Max Memory Used: 86 MB	Init Duration: 322.08 ms

Request ID
13d2e237-72b0-4504-979b-8250bb699293
```
#### EBS snapshot is deleted because there is no associated volume was found

#### Congratulations!!!

