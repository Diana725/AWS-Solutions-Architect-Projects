
In  step 1 of this project, I got to:

* Setup a basic environment where Wordpress would run from.
* Configure some System Manager Parameters which the manual and automatic stages of this project.
* Perform a manual installation of wordpress and a database on the same EC2 instance.

## Step 1A - Login to an AWS account 
Using a user with admin privileges, login to an AWS account and set the region to ````us-east-1```` ````N. Virginia````

To auto configure the VPC, I used this [Link](https://us-east-1.console.aws.amazon.com/cloudformation/home?region=us-east-1#/stacks/quickcreate?templateURL=https://learn-cantrill-labs.s3.amazonaws.com/aws-elastic-wordpress-evolution/A4LVPC.yaml&stackName=A4LVPC) from [learn-cantrill-io-labs](https://github.com/acantril/learn-cantrill-io-labs/blob/master/aws-elastic-wordpress-evolution/02_LABINSTRUCTIONS/STAGE1%20-%20Setup%20and%20Manual%20wordpress%20build.md)

Click on ````Create Stack```` and wait for the stack to move into the ````CREATE_COMPLETE```` state before proceeding.

## Step 1B - Launch an EC2 Instance to run Wordpress

Here, I moved to the EC2 console of your AWS account and followed these steps:

* Click ````Launch Instance````
* For the ````name```` I used ````Wordpress-Manual````
* Choose an ````Amazon Linux 2 AMI (HVM, SSD Volume Type```` AMI with a ````64-bit (x86)```` architecture
* Select ````t2.micro```` for the instance type
* Under ````Key Pair(login)```` select ````Proceed without a KeyPair(not recommended)
* For ````Network Settings````, click ````Edit```` and in the VPC download select ````A4LVPC````
* For ````Subnet```` select ````sn-pub-A````
* Make sure for both ````Auto-assign public IP```` and ````Auto-assign IPv6 IP```` you set ````Enable````
* Under security Group Check ````Select an existing security group````
Select ````A4LVPC-SGWordpress```` 
* Leave the default storage settings as is
* Expand ````Advanced Details````
* For ````IAM instance profile role```` select ````A4LVPC-WordpressInstanceProfile````
* Find the ````Credit Specification Dropdown```` and choose ````Standard```` (some accounts aren't enabled for Unlimited)
* Click ````Launch Instance````
* Click ````View All Instances````
