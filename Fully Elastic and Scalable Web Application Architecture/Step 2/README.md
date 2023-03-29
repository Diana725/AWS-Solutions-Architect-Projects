# Creating an EC2 Launch Template containing Build Instructions

In Step 2 of this project, I created a launch templete which could automate the build process of Wordpress. The architecture will still use the single instance for both the WordPress application and database, the only change will be an automatic build rather than manual.

## Step 2A - Create the Launch Template
I opened the EC2 console and followed the steps below:
* Click ````Launch Templates```` under ````Instances```` on the left menu
* Click ````Create Launch Template````
* Under ````Launch Template Name```` enter ````Wordpress````
* Under ````Template version description```` enter ````Single server DB and App````
* Check the ````Provide guidance to help me set up a template that I can use with EC2 Auto Scaling```` box
* Under ````Application and OS Images (Amazon Machine Image)```` click ````Quick Start````
* Click ````Amazon Linux````
* In the ````Amazon Machine Image```` dropdown, locate ````Amazon Linux 2 AMI (HVM), SSD Volume TYpe```` and set the Architecture to ````64-bit (x86)````
* Under ````Instance Type```` select ````t2.micro````
* Under ````Key pair (login)```` select ````Don't include in launch template````
* Under ````networking Settings```` select ````select existing security group```` and choose ````A4LVPC-SGWordpress```` 
* Leave storage volumes unchanged
* Leave Resource Tags Unchanged
* Expand ````Advanced Details```` Under ````IAM instance profile```` select ````A4LVPC-WordpressInstanceProfile````
* Under ````Credit specification```` select ````Standard````

## Step 2B - Add UserData
At this point I added the build configuration which for the EC2 instance. I did this in the ````User Data```` box under ````Advanced Details````:

```
#!/bin/bash -xe

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

yum -y update
yum -y upgrade

yum install -y mariadb-server httpd wget
amazon-linux-extras install -y lamp-mariadb10.2-php7.2 php7.2
amazon-linux-extras install epel -y
yum install stress -y

systemctl enable httpd
systemctl enable mariadb
systemctl start httpd
systemctl start mariadb

mysqladmin -u root password $DBRootPassword

wget http://wordpress.org/latest.tar.gz -P /var/www/html
cd /var/www/html
tar -zxvf latest.tar.gz
cp -rvf wordpress/* .
rm -R wordpress
rm latest.tar.gz

sudo cp ./wp-config-sample.php ./wp-config.php
sed -i "s/'database_name_here'/'$DBName'/g" wp-config.php
sed -i "s/'username_here'/'$DBUser'/g" wp-config.php
sed -i "s/'password_here'/'$DBPassword'/g" wp-config.php
sed -i "s/'localhost'/'$DBEndpoint'/g" wp-config.php

usermod -a -G apache ec2-user   
chown -R ec2-user:apache /var/www
chmod 2775 /var/www
find /var/www -type d -exec chmod 2775 {} \;
find /var/www -type f -exec chmod 0664 {} \;

echo "CREATE DATABASE $DBName;" >> /tmp/db.setup
echo "CREATE USER '$DBUser'@'localhost' IDENTIFIED BY '$DBPassword';" >> /tmp/db.setup
echo "GRANT ALL ON $DBName.* TO '$DBUser'@'localhost';" >> /tmp/db.setup
echo "FLUSH PRIVILEGES;" >> /tmp/db.setup
mysql -u root --password=$DBRootPassword < /tmp/db.setup
rm /tmp/db.setup

```
Ensure to leave a blank line at the end
Click ````Create Launch Template````
Click ````Launch Templates```` towards the top of the screen

## Step 2C - Launch an instance using the template

* Select the launch template in the list. It should be called ````Wordpress````
* Click ````Actions```` and ````Launch instance from template```` 
* Under ````Key Pair(login)```` click the dropdown and select ````Proceed without a key pair(Not Recommended)````
* Scroll down to ````Network settings```` and under ````Subnet```` select ````sn-pub-A````
Scroll to ````Resource Tags```` click ````Add Tag```` Set ````Key```` to ````Name```` and Value to ````Wordpress-LT````
* Scroll to the bottom and click ````Launch Instance````
* Click the instance id in the ````Success```` box
## Step 2D - Test

I Opened the EC2 console, selected the Wordpress-LT instance, copied the IPv4 Public IP into my clipboard, and opened that Ip in a new tab.
That being done, I saw the wordpress welcome page;
![Wordpress1](https://user-images.githubusercontent.com/122386130/228511030-22bdd363-d6d4-49d7-8bdd-22d98fd2f884.PNG)
## Perform Initial Configuration and make a post

* In ````Site Title```` enter ````Catagram````
* In ````Username```` enter ````admin```` in ````Password```` it should suggest a strong password for the wordpress admin user
* In ````Your Email```` enter your email address
* Click ````Install WordPress```` Click ````Log In````
* In ````Username or Email Address```` enter ````admin````
* In ````Password```` enter the previously noted down strong password
* Click ````Log In````
* Click ````Posts```` in the menu on the left
* Select ````Hello World!```` Click ````Bulk Actions```` and select ````Move to Trash```` Click ````Apply````
* Click ````Add New````
* If you see any popups close them down
* For title ````The Best Animal(s)!````
* Click the ````+```` under the title, select ````Gallery```` Click ````Upload````
* Select some animal pictures.... if you don't have any use google images to download some
* Upload them
* Click ````Publish````
* Click ````Publish```` again, then Click ````view Post````

This worked for me and I was able to see launch my template successfully.
![Wordpress1b](https://user-images.githubusercontent.com/122386130/228513710-d10f4d45-5462-4de8-8fcc-425da5dd1403.PNG)

## Step 2D - FINISH
Here, I solved the problem of having to build an EC2 instance manually by automating the process. However, the problem of the database and application being on the same instance remained, as neither could scale without the other.

From here, I moved to [Step 3](https://github.com/Diana725/AWS-Solutions-Architect-Projects/tree/main/Fully%20Elastic%20and%20Scalable%20Web%20Application%20Architecture/Step%203)
