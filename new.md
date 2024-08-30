### Go to AWS Management console and Create an Ubuntu Ec2 instance

Go to Snapshot

Create Snapshot using default volume

#### Make sure all are running and Active

### Go to Lambda console

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

Go to configuration tab

default execution is 3 sec

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
delete ec2 instance
save code deploy and test

