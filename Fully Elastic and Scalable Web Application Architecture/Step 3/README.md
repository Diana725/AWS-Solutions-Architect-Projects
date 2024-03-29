# Splitting the database functionality from the EC2 instance running MariaDB to an RDS instance running the MySQL Engine

In this section of the project, I successfully split the database functionality from the EC2 instance running MariaDB to an RDS instance running the MySQL Engine. This will inturn allow the DB and Instance to scale independently, allowing the data to be secure past the lifetime of the EC2 instance.

Step 3A - Create RDS Subnet Group

A subnet group is what allows RDS to select from a range of subnets to put its databases inside
In this case you will give it a selection of 3 subnets sn-db-A / B and C
RDS can then decide freely which to use.
To do this:
* Click ````Subnet Groups````
* Click ````Create DB Subnet Group````
* Under ````Name```` enter ````WordPressRDSSubNetGroup````
* Under ````Description```` enter ````RDS Subnet Group for WordPress````
* Under ````VPC```` select ````A4LVPC````
* Under ````Add Subnets```` In ````Availability Zones```` select ````us-east-1a```` & ````us-east-1b```` & ````us-east-1c````
* Under ````Subnets```` check the box next to

** 10.16.16.0/20 (this is sn-db-A)
** 10.16.80.0/20 (this is sn-db-B)
** 10.16.144.0/20 (this is sn-db-C)
Click ````Create````

Step 3B- Create RDS Instance

In this stage of the demo, I provisioned an RDS instance using the subnet group to control placement within the VPC. I used a single AZ configuration  instead of the multi AZ one to keep costs low.
Here are the instructions for this step:

* Click ````Databases````
* Click ````Create Database````
* Click ````Standard Create````
* Click ````MySql````
* Under ````Version```` select ````MySQL 8.0.28```` (best aurora compatibility for snapshot migrations)
* Scroll down and select ````Free Tier```` under templates (this ensures there will be no costs for the database but it will be single AZ only)
* Under ````Db instance identifier```` enter ````a4lWordPress```` under ````Master Username```` enter enter the value from [here](https://console.aws.amazon.com/systems-manager/parameters/A4L/Wordpress/DBUser/description?region=us-east-1&tab=Table)
* Under ````Master Password```` and ````Confirm Password```` enter the value from [here](https://console.aws.amazon.com/systems-manager/parameters/A4L/Wordpress/DBPassword/description?region=us-east-1&tab=Table)
* Under ````DB Instance size````, then ````DB instance class````, then ````Burstable classes (includes t classes)```` make sure db.t2.micro is selected
* Scroll down, under ````Connectivity````, ````Virtual private cloud (VPC)```` select ````A4LVPC````
* Ensure under ````Subnet group```` that ````WordPressRDSSubNetGroup```` is selected
* Make sure ````Public Access```` is set to ````No````
* Under ````VPC security groups```` make sure ````choose existing```` is selected, remove ````default```` and add ````A4LVPC-SG-Database````
* Under ````Availability Zone```` set ````us-east-1a````
* Scroll down and expand ````Additional configuration````
* In the ````Initial database name```` box enter the value from [here](https://console.aws.amazon.com/systems-manager/parameters/A4L/Wordpress/DBName/description?region=us-east-1&tab=Table) 
* Scroll to the bottom and click ````Create Database````

Wait for this process to finish to move to the next step.

## Step 3C - Migrate WordPress data from MariaDB to RDS

* Open the EC2 console
* Click ````Instances````
* Locate the ````WordPress-LT```` instance, right click, ````Connect```` and choose ````Session Manager```` and then click ````Connect````
* Type ````sudo bash````
* Type ````cd````
* Type ````clear````
### Populate Environment Variables
Here, I did an export of the SQL database running on the local EC2 instance. But first, I run these commands to populate variables with the data from the Parameter Store to avoid having to keep locating password

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
### Take a Backup of the local DB
To take a backup of the database, I run:
```
mysqldump -h $DBEndpoint -u $DBUser -p$DBPassword $DBName > a4lWordPress.sql
```
### Restore that Backup into RDS

Move to the RDS Console
* Click the ````a4lWordPressdb```` instance
* Copy the ````endpoint```` into your clipboard
* Move to the Parameter [store](https://console.aws.amazon.com/systems-manager/parameters?region=us-east-1)
* Check the box next to ````/A4L/Wordpress/DBEndpoint```` and click ````Delete```` (please do delete this, not just edit the existing one)
* Click ````Create Parameter````

* Under ````Name```` enter ````/A4L/Wordpress/DBEndpoint````
* Under ````Descripton```` enter ````WordPress Endpoint Name````
* Under ````Tier```` select ````Standard````
* Under ````Type```` select ````String````
* Under ````Data Type```` select ````text````
* Under ````Value```` enter the RDS endpoint endpoint you just copied
* Click ````Create Parameter````

Update the DbEndpoint environment variable with:
```
DBEndpoint=$(aws ssm get-parameters --region us-east-1 --names /A4L/Wordpress/DBEndpoint --query Parameters[0].Value)
DBEndpoint=`echo $DBEndpoint | sed -e 's/^"//' -e 's/"$//'`
```
Restore the database export into RDS using:
```
mysql -h $DBEndpoint -u $DBUser -p$DBPassword $DBName < a4lWordPress.sql 
```
### Change the WordPress config file to use RDS

This command will substitute ````localhost```` in the config file for the contents of ````$DBEndpoint```` which is the RDS instance

```
sudo sed -i "s/'localhost'/'$DBEndpoint'/g" /var/www/html/wp-config.php
```
## Step 3D - Stop the MariaDB Service

```
sudo systemctl disable mariadb
sudo systemctl stop mariadb
```
## Step 3E - Test WordPress
Move to the EC2 Console
* Select the ````WordPress-LT```` Instance
* Copy the ````IPv4 Public IP```` into your clipboard
* Open the IP in a new tab
I saw that my blog was working, even though I had stopped and disabled MariaDB on the EC2 instance . Now, the application was using RDS

## Step 3F - Update the Launch Template so it doesn't Start and Enable MariaDB
Move to the EC2 Console
* Under ````Instances```` click ````Launch Templates````
* Select the ````WordPress```` launch Template (select, dont click) Click ````Actions```` and ````Modify Template (Create new version)````
* This template version will be based on the existing version ... so many of the values will be populated already
* Under ````Template version description```` enter ````Single server App Only````

* Scroll down to ````Advanced details```` and expand it
* Scroll down to ````User Data```` and expand the text box as much as possible

Locate and remove the following lines:
```
systemctl enable mariadb
systemctl start mariadb
mysqladmin -u root password $DBRootPassword


echo "CREATE DATABASE $DBName;" >> /tmp/db.setup
echo "CREATE USER '$DBUser'@'localhost' IDENTIFIED BY '$DBPassword';" >> /tmp/db.setup
echo "GRANT ALL ON $DBName.* TO '$DBUser'@'localhost';" >> /tmp/db.setup
echo "FLUSH PRIVILEGES;" >> /tmp/db.setup
mysql -u root --password=$DBRootPassword < /tmp/db.setup
rm /tmp/db.setup
```
* Click ````Create Template Version````
* Click ````View Launch Template````
* Select the template again (dont click) Click ````Actions```` and select ````Set Default Version````
* Under ````Template version```` select ````2````
* Click ````Set as default version````

## Step 3 - Finish
Now, the application and database are built automatically, and I have shifted the database to an RDS instance. This way, both the EC2 and RDS instances can scale without the other.

Still, we have the problem of the application media and UI store being local to the instance, meaning scaling IN/OUT would risk the media. Customer connections are also directly to an instance and there's no option to perform health cheacks or auto healing. The IP of the instance should also be hardcoded into the database.

From here, I moved to [Stage 4](https://github.com/Diana725/AWS-Solutions-Architect-Projects/tree/main/Fully%20Elastic%20and%20Scalable%20Web%20Application%20Architecture/Step%204).
