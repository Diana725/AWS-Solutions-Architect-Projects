
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

## Step 1C - Create Systme Manager Parameter Store Values for Wordpress

In this sub section, I created parameters to store the important configuration items for the platform I was building.
The steps to achieving this begin at moving to the ````Systems Manager```` console and clicking ````Parameter Store```` on the menu on the left

### Create Parameter - DBUser (the login for the specific Wordpress DB)

* Click ````Create Parameter```` 
* Set Name to ````/A4L/Wordpress/DBUser```` Set
* Description to ````Wordpress Database User````
* Set Tier to ````Standard````
* Set Type to ````String````
* Set Data type to ````text````
* Set ````Value```` to ````a4lwordpressuser````
* Click ````Create Parameter````

### Create Parameter - DBName (the name of the Wordpress database)

* Click ````Create Parameter```` 
* Set Name to ````/A4L/Wordpress/DBName Set````
* Description to ````Wordpress Database Name````
* Set Tier to ````Standard````
* Set Type to ````String````
* Set Data type to ````text````
* Set ````Value```` to ````a4lwordpressdb````
* Click ````Create Parameter````

### Create Parameter - DBEndpoint (the endpoint for the Wordpress DB)

* Click ````Create Parameter```` 
* Set Name to ````/A4L/Wordpress/DBEndpoint````
* Set Description to ````Wordpress Endpoint Name````
* Set Tier to ````Standard````
* Set Type to ````String````
* Set Data type to ````text````
* Set ````Value to localhost````
* Click ````Create Parameter````

### Create Parameter - DBPassword (the password for the DBUser)

* Click ````Create Parameter```` 
* Set Name to ````/A4L/Wordpress/DBPassword````
* Set Description to ````Wordpress DB Password````
* Set Tier to ````Standard````
* Set Type to ````SecureString````
* Set ````KMS Key Source```` to ````My Current Account````
* Leave ````KMS Key ID```` as default 
* Set ````Value```` to ````4n1m4l54L1f3```` 
* Click ````Create Parameter````

### Create Parameter - DBRootPassword (the password for the database root user, used for self-managed admin)

* Click ````Create Parameter```` 
* Set Name to ````/A4L/Wordpress/DBRootPassword```` 
* Set Description to ````Wordpress DBRoot Password````
* Set Tier to ````Standard````
* Set Type to ````SecureString````
* Set ````KMS Key Source```` to ````My Current Account````
* Leave ````KMS Key ID```` as default Set ````Value```` to ````4n1m4l54L1f3````
* Click ````Create Parameter````

## Step 1D - Connect to the instance and install a database and Wordpress

* Right click on ````Wordpress-Manual```` choose ````Connect````
* Choose ````Session Manager````
* Click ````Connect````
* type ````sudo bash```` and press enter
* type ````cd```` and press enter
* type ````clear```` and press enter

### Bring in the parameter values from Systems Manager

I run the commands below to bring the parameter store values into ENV variables to make the manual build easier.

```
DBPassword=$(aws ssm get-parameters --region us-east-1 --names /A4L/Wordpress/DBPassword --with-decryption --query Parameters[0].Value)
DBPassword=`echo $DBPassword | sed -e 's/^"//' -e 's/"$//'`

DBRootPassword=$(aws ssm get-parameters --region us-east-1 --names /A4L/Wordpress/DBRootPassword --with-decryption --query Parameters[0].Value)
DBRootPassword=`echo $DBRootPassword | sed -e 's/^"//' -e 's/"$//'`

DBUser=$(aws ssm get-parameters --region us-east-1 --names /A4L/Wordpress/DBUser --query Parameters[0].Value)
DBUser=`echo $DBUser | sed -e 's/^"//' -e 's/"$//'`

DBName=$(aws ssm get-parameters --region us-east-1 --names /A4L/Wordpress/DBName --query Parameters[0].Value)
DBName=`echo $DBName | sed -e 's/^"//' -e 's/"$//'`

DBEndpoint=$(aws ssm get-parameters --region us-east-1 --names /A4L/Wordpress/DBEndpoint --query Parameters[0].Value)
DBEndpoint=`echo $DBEndpoint | sed -e 's/^"//' -e 's/"$//'`
```

### Install Updates

```
sudo yum -y update
sudo yum -y upgrade
```
### Install Pre-Reqs and Web Server

```
sudo yum install -y mariadb-server httpd wget
sudo amazon-linux-extras install -y lamp-mariadb10.2-php7.2 php7.2
sudo amazon-linux-extras install epel -y
sudo yum install stress -y
```
### Set DB and HTTP Server to running and start by default

```
sudo systemctl enable httpd
sudo systemctl enable mariadb
sudo systemctl start httpd
sudo systemctl start mariadb
```
### Set the MariaDB Root Password

```
sudo mysqladmin -u root password $DBRootPassword
```

### Download and extract Wordpress

```
sudo wget http://wordpress.org/latest.tar.gz -P /var/www/html
cd /var/www/html
sudo tar -zxvf latest.tar.gz
sudo cp -rvf wordpress/* .
sudo rm -R wordpress
sudo rm latest.tar.gz
```
### Configure the wordpress wp-config.php file

```
sudo cp ./wp-config-sample.php ./wp-config.php
sudo sed -i "s/'database_name_here'/'$DBName'/g" wp-config.php
sudo sed -i "s/'username_here'/'$DBUser'/g" wp-config.php
sudo sed -i "s/'password_here'/'$DBPassword'/g" wp-config.php
```
### Fix Permissions on the filesystem

```
sudo usermod -a -G apache ec2-user   
sudo chown -R ec2-user:apache /var/www
sudo chmod 2775 /var/www
sudo find /var/www -type d -exec chmod 2775 {} \;
sudo find /var/www -type f -exec chmod 0664 {} \;
```
### Create Wordpress User, set its password, create the database and configure permissions

```
sudo echo "CREATE DATABASE $DBName;" >> /tmp/db.setup
sudo echo "CREATE USER '$DBUser'@'localhost' IDENTIFIED BY '$DBPassword';" >> /tmp/db.setup
sudo echo "GRANT ALL ON $DBName.* TO '$DBUser'@'localhost';" >> /tmp/db.setup
sudo echo "FLUSH PRIVILEGES;" >> /tmp/db.setup
sudo mysql -u root --password=$DBRootPassword < /tmp/db.setup
sudo rm /tmp/db.setup
```
### Test Wordpress is installed
I went to the EC2 console, selected the ````Wordpress-Manual````, copied the ````IPv4 Public IP```` into my clipboard, and opened it in a new tab. This loaded to the Wordpress welcome page.

### Perform Initial Configuration and make a post

* In ````Site Title```` enter ````Catagram````
* In ````Username```` enter ````admin```` in ````Password```` it should suggest a strong password for the Wordpress admin user.
* In ````Your Email```` enter your email address
* Click ````Install WordPress```` Click ````Log In````
* In ````Username or Email```` Address enter ````admin````
* In ````Password```` enter the previously noted down strong password
* Click ````Log In````
* Click ````Posts```` in the menu on the left
* Select ````Hello World!```` Click ````Bulk Actions```` and select ````Move to Trash```` Click ````Apply````
* Click ````Add New````
* If you see any popups close them down
* For title ````The Best Animal(s)!````
* Click the ````+```` under the title, select ````Gallery```` Click ````Upload````
* Select some animal pictures.... if you dont have any use google images to download some then
* Upload them
* Click ````Publish````
* Click ````Publish```` Click ````View Post````

All this was working, meaning my manual installation and configuration of Wordpress was successful.

## STAGE 1E - FINISH

* On the EC2 console, right click ````Wordpress-Manual```` , ````Instance State````, ````Terminate````, ````Yes, Terminate````

## THE LIMITATIONS OF THIS CONFIGURATION
* Since this application and database were built manually, there was no automation which made the deployment process slow. 
* The database and application are on the same instance so neither can scale without the other
* Customer connections are to an instance directly so I could not apply health checks or autohealing
* Starting and stopping the EC2 instance causes the IP address to change and therefore, I ended up losing my images

From Here, I moved on to Step 2
