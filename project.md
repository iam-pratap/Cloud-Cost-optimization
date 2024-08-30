
GO to aws management console and cretae ec2 instance(ubuntu)

go to snapshots

create snapshot using default volume


go to lambda function

finction name ---> cost optimization ebs snapshot
runtime ---> python3.10
architecture --> x86_64
create

code will fails

then 
go to code source
in lambda-function section paste code
ctrl+s
click deploy button
click on test

configure test evnt
event name -test
event sharing setting ---private
templete - hello world

save it
you do it manully that we if you do it using cloud watch then you don't create test event

now it will fail bydefault lambda execution time 3sec and it is failing soame permission issue.

go to configure tab and edit
default executin time for lambda is 3ses
edit to 10 sec save it

it is better to keep the execution time as small as possibel because aws will charge you using this as a parameter like the lambda execution time is also one of the parameter for charging so make sure that you keep this time as less as possible

go to the code
go to permission ---> rolename
go to new tab and add the permission to it

add permission ----> attach policie
create policy ---> service --ec2 --->search snapshort ---delete snapshot and
delte snapshot
describe instance describe volumes

resources all

give the policy nam e and create policy and attch this policy



and test again

delete ec2

test again

