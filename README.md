## 🚀 AWS Lift and Shift Project – End-to-End Deployment

This project demonstrates a complete Lift and Shift migration strategy to AWS, focusing on deploying a Java-based web application using key AWS services like EC2, S3, ELB, Auto Scaling, Route 53, and more. It also incorporates backend services such as MySQL, RabbitMQ, and Memcached. 

### 🛠️ Technologies and AWS Services Used

- **Compute**: Amazon EC2, Launch Templates, Auto Scaling Group

- **Networking**: Elastic Load Balancer (ALB), Amazon Route 53

- **Storage**: Amazon S3

- **Security**: AWS Certificate Manager, IAM

- **Notification**: Amazon SNS

- **Database and Caching**:

  - MySQL

  - RabbitMQ

  - Memcached

- Web Server: Apache Tomcat
  
### ARCHITECTURE DIAGRAM

![ARCHITECTURE DIAGRAM](images/LiftaNDsHift%20Architecture%20Diagram.png)

### Prerequisites

1. AWS Free Tier account

2. Local tools installed:

 - JDK 17

 - Maven

 - AWS CLI

3. SSL/TLS certificate in ACM (Optional for HTTPS setup)

4. Domain name via GoDaddy or other DNS provider (Optional)

### 🔁 Workflow Overview

1. Login To AWS Account
2. Create Key Pairs
3. Create Security Groups
4. Launch Instances with user data [BASH SCRIPTS]
5. Update IP To name mapping in Route 53
6. Build Application from source code
7. Upload To s3 bucket
8. Download artifact To Tomcat Ec2 Instance
9. Setup ELB with HTTPS [Amazon Certificate Manager]
10. Map ELB Endpoint To website name in Godaddy DNS
11. Verify
12. Build Autoscaling Group for Tomcat Instances.

### STEP 1: CREATE THREE 3 SECURITY GROUPS

a. *CREATE SECURITY ELASTIC LOAD BALANCING SECURITY GROUP*

- HTTP ALLOW FROM ANYWHERE IPV4
- HTTP ALLOW FROM ANYWHERE IPV6
- HTTPS ALLOW FROM ANYWHERE IPV4
- HTTPS ALLOW FROM ANYWHERE IPV6
  
![ELB Security Group](images/ELB-Security%20Group.png)

b. *CREATE TOMCAT INSTANCE SECURITY GROUP(Tomcat-app01)*

- Give PORT 8080 RULE and ALLOW it from ELB Security Group
- ALLOW PORT 22 FROM MY IP
  
![Tomcat Security Group](images/Tomcat%20Security%20Group.png)
![Tomcat-app01 Security Group](images/Tomcat%20Security%20Group-2.png)

c. *CREATE BACKEND SECURITY GROUP*

- Allow MySQL/Aurora PORT(3306) from TOMCAT SECURIY GROUP
- Type 11211 in the PORT RANGE field and allow the PORT from Tomcat Security Group
- Type PORT 5672 in the PORT RANGE field and allow it from Tomcat Security Group
- Allow PORT 22 From MyIP
- ***NB: SELECT (EDIT INBOUND RULES) TO ALLOW ALL TRAFFIC FROM BACKEND SECURITY GROUP ITSELF***
  
![Backend Security Group](images/Backend%20security%20group%20allow%20itself.png)

### STEP 2: CREATE ONE KEY PAIR

![Key Pair](images/Create%20Key%20Pair.png)

### STEP 3: CREATE FOUR EC2 INSTANCES

![Launch EC2 Instance](images/Launch%20EC2%20instance.png)

**Clone the source Code into VSCODE and Switch to liftandshift branch**

![Clone Source Code](images/clone%20the%20source%20code.png)

```sh
https://github.com/hkhcoder/vprofile-project.git
```

1. ### CREATE ONE INSTANCE FOR THE MYSQL DATABASE

    1. Give it a name (MySQL-db01) and tags `Key and Value` and Resource types (instance and volume)
    2. Choose AMI -> Amazon LINUX 2023 AMI and Type -> t2.micro
    3. Select your Key Pair and the **BACKEND SECURITY GROUP**
    4. Scroll Down To ADVANCED DETAILS AND COPY and PASTE *mysql.sh script* from the source code from this path userdata -> awsLiftandShit -> vprofile-project -> userdata -> mysql.sh
  
    ```bash
    #!/bin/bash
    DATABASE_PASS='admin123'
    sudo dnf update -y
    sudo dnf install git zip unzip -y
    sudo dnf install mariadb105-server -y
    # starting & enabling mariadb-server
    sudo systemctl start mariadb
    sudo systemctl enable mariadb
    cd /tmp/
    git clone -b main https://github.com/hkhcoder/vprofile-project.git
    #restore the dump file for the application
    sudo mysqladmin -u root password "$DATABASE_PASS"
    sudo mysql -u root -p"$DATABASE_PASS" -e "ALTER USER 'root'@'localhost' IDENTIFIED BY '$DATABASE_PASS'"
    sudo mysql -u root -p"$DATABASE_PASS" -e "DELETE FROM mysql.user WHERE User='root' AND Host NOT IN ('localhost', '127.0.0.1', '::1')"
    sudo mysql -u root -p"$DATABASE_PASS" -e "DELETE FROM mysql.user WHERE User=''"
    sudo mysql -u root -p"$DATABASE_PASS" -e "DELETE FROM mysql.db WHERE Db='test' OR Db='test\_%'"
    sudo mysql -u root -p"$DATABASE_PASS" -e "FLUSH PRIVILEGES"
    sudo mysql -u root -p"$DATABASE_PASS" -e "create database accounts"
    sudo mysql -u root -p"$DATABASE_PASS" -e "grant all privileges on accounts.* TO 'admin'@'localhost' identified by 'admin123'"
    sudo mysql -u root -p"$DATABASE_PASS" -e "grant all privileges on accounts.* TO 'admin'@'%' identified by 'admin123'"
    sudo mysql -u root -p"$DATABASE_PASS" accounts < /tmp/vprofile-project/src/main/resources/db_backup.sql
    sudo mysql -u root -p"$DATABASE_PASS" -e "FLUSH PRIVILEGES"
    ```

    5. Launch the EC2 Instance

2. ### CREATE MEMCACHE EC2 INSTANCE

    1. Give a name (Memcache-mc01) and tags `Key and Value` and Resource types (instance and Volume)
    2. Choose AMI -> Amazon LINUX 2023 AMI and Type -> t2.micro
    3. Select your Key Pair and the **BACKEND SECURITY GROUP**
    4. Scroll Down To ADVANCED DETAILS AND COPY and PASTE *memcache.sh script* from the source code from this path userdata -> awsLiftandShit -> vprofile-project -> userdata -> memcache.sh

    ```bash
    #!/bin/bash
    sudo dnf install memcached -y
    sudo systemctl start memcached
    sudo systemctl enable memcached
    sudo systemctl status memcached
    sed -i 's/127.0.0.1/0.0.0.0/g' /etc/sysconfig/memcached
    sudo systemctl restart memcached
    sudo memcached -p 11211 -U 11111 -u memcached -d
    ```

    4. Launch the EC2 Instance
  
3. ### CREATE RABBITMQ EC2 INSTANCE

    1. Give a name (RabbitMQ-rmq01) and tags `Key and Value` and Resource types (instance and Volume)
    2. Choose AMI -> Amazon LINUX 2023 AMI and Type -> t2.micro
    3. Select your Key Pair and the **BACKEND SECURITY GROUP**
    4. Scroll Down To ADVANCED DETAILS AND COPY and PASTE *rabbitmq.sh script* from the source code from this path  userdata -> awsLiftandShit -> vprofile-project -> userdata -> rabbitmq.sh

    ```bash
    #!/bin/bash
    ## primary RabbitMQ signing key
    rpm --import 'https://github.com/rabbitmq/signing-keys/releases/download/3.0/rabbitmq-release-signing-key.asc'
    ## modern Erlang repository
    rpm --import 'https://github.com/rabbitmq/signing-keys/releases/download/3.0/cloudsmith.rabbitmq-erlang.E495BB49CC4BBE5B.key'
    ## RabbitMQ server repository
    rpm --import 'https://github.com/rabbitmq/signing-keys/releases/download/3.0/cloudsmith.rabbitmq-server.9F4587F226208342.key'
    curl -o /etc/yum.repos.d/rabbitmq.repo https://raw.githubusercontent.com/hkhcoder/vprofile-project/refs/heads/awsliftandshift/al2023rmq.repo
    dnf update -y
    ## install these dependencies from standard OS repositories
    dnf install socat logrotate -y
    ## install RabbitMQ and zero dependency Erlang
    dnf install -y erlang rabbitmq-server
    systemctl enable rabbitmq-server
    systemctl start rabbitmq-server
    sudo sh -c 'echo "[{rabbit, [{loopback_users, []}]}]." > /etc/rabbitmq/rabbitmq.config'
    sudo rabbitmqctl add_user test test
    sudo rabbitmqctl set_user_tags test administrator
    rabbitmqctl set_permissions -p / test ".*" ".*" ".*"

    sudo systemctl restart rabbitmq-server
    ```

    5. Launch the EC2 Instance
  
4. ### CREATE TOMCAT EC2 INSTANCE

    1. Give a name (Tomcat-app01) and tags `Key and Value` and Resource types (instance and Volume)
    2. Choose AMI -> Ubuntu 24 AMI and Type -> t3.micro
    3. Select your Key Pair and the **TOMCAT SECURITY GROUP**
    4. Scroll Down To ADVANCED DETAILS AND COPY and PASTE *tomcat_ubuntu.sh script* from the source code from this path userdata -> awsLiftandShit -> vprofile-project -> userdata -> tomcat_ubuntu.sh
  
    ```bash
    #!/bin/bash
    sudo apt update
    sudo apt upgrade -y
    sudo apt install openjdk-17-jdk -y
    sudo apt install tomcat10 tomcat10-admin tomcat10-docs tomcat10-common git -y
    ```

    5. Launch the EC2 Instance
  
  ![All the Running Instances](images/All%20the%20Running%20Instances.png)

   *SSH INTO ALL THE INSTANCES TO VERIFY IF THEY ARE WORKING AS THEY SHOULD BE*

   ```sh
   ssh -i path/to/the/keypair instanceusername@instancePublicIP
   ```

### STEP 4: CREATE PRIVATE HOSTED ZONE IN ROUTE 53

![Create private Hosted Zone](images/Route%2053.png)

- Search and select Route 53 in your AWS Management Console
- Click on create hosted zone in the ROUTE 53 Dashboard and click on Get Started

![Create private Hosted Zone](images/Route%2053%20hosted%20zone_%20Get%20Started.png)

c. Give it a name in the Domain name field (I USED vprofile.in as my domain name)

- TYPE: Choose private hosted zone for internal name resolution
  
![PRIVATE HOSTED ZONE](images/Domain%20name%20hosted%20zone%20type.png)

- Choose Region (Your Region such as US East (N.Virginia) )
  
![HOSTED ZONE REGION AND VPC](images/Choose%20you%20Domain%20Region%20and%20VPC.png)

- Choose your VPC and Click on create hosted Zone
  
 ![Hosted Zone](images/Create%20private%20zone%20Records.png)

- Get ALL THE INSTANCES PRIVATE IPs one-by-one and go to the hosted zone you created and click on CREATE RECORD
- Give the record a name like (db01) and paste the MySQL-db01 instance private IP in the value field and click on create records
  
  ![MySQL-db01 Record](images/MySQL%20Record.png)
  
- *FOLLOW THE SAME PROCESS TO CREATE RECORD FOR RabbitMQ-rmq01, Tomcat-app01 and Memcache-mc01 INSTANCES AND GIVE RECORD NAME AS rmq01, app01 AND mc01 RESPECTIVELY*

- **Verify if the records are resolving to IP**

- SSH into `Tomcat-app01` EC2 Instance and type the command below

  ```sh
  ping -c 4 db01.vprofile.in
  ```

  ![Ping Hosted Zones](images/ping%20-c%204%20db01.vprofile.in.png)

*NB: IF it is not responding add ALL ICMP rule From ANYWHERE in The Backend Security Group*

### STEP 5: CREATE S3 BUCKET

1. Choose General Purpose and give a unique name to your s3 bucket

![CREATE S3 BUCKET](images/Create%20S3%20Bucket.png)

2. Click Create Bucket

![S3 BUCKET](images/Created%20S3%20Bucket.png)

### STEP 6: CREATE IAM USER

![IAM USER](images/Create%20IAM%20user.png)
  
- Create an IAM USER and attach s3 FULL ACCESS Permission

![IAM USER S3 FULL ACCESS PERMISSION](images/Attach%20S3%20Full%20Access.png)

- After creating the IAM User generate an ACCESS KEYS AND SECRET ACCESS KEYS give `COMMANDLINE INTERFACE ACCESS`

![IAM USER](images/Generate%20Access%20Keys.png)

- Give `COMMAND LINE INTERFACE` access

![IAM USER](images/Command%20Line%20Interface%20Access.png)

### STEP 7: CREATE IAM ROLE

- Search IAM and select roles
- Click on Create roles and select AWS Service
- In the use use case section, choose EC2 and GO NEXT
  
![IAM ROLE](images/IAM%20Role%20Select%20trusted%20entity.png)

- In the Permission policies search box search for S3 and add S3 FULL ACCESS Permission and GO NEXT

![IAM ROLE S3 FULL ACCESS PERMISSION](images/IAM%20Role%20s3%20Full%20Access.png)

- Give the role a name and Click on Create Role

![IAM ROLE](images/Give%20a%20To%20the%20IAM%20Role.png)

### STEP 8: APPLY THE IAM ROLE To `Tomcat-app01` EC2 INSTANCE

- Select the `Tomcat-app01` instance and click on Action
- Select Security and click on Modify IAM role

![APPLY IAM ROLE](images/Apply%20the%20IAM%20Role%20the%20Tomcat-app01%20EC2%20Instance.png)

- In the NEXT PAGE select your IAM role you created and click on update IAM role
  
  ![CREATED IAM ROLE](images/Select%20the%20IAM%20Role%20You%20Created%20to%20Apply%20to%20the%20app01%20Instance.png)

### STEP 9: OPEN THE REPOSITORY IN VSCODE

* Open the __application.properties__ file and update the following;<br>

1. Update <br>
   *`jdbc.url` VALUE From jdbc:mysql://db01:3306/accounts?useUnicode=true&characterEncoding=UTF-8&zeroDateTimeBehavior=convertToNull* `TO` <br>
    *jdbc:mysql://db01.vprofile.in:3306/accounts?useUnicode=true&characterEncoding=UTF-8&zeroDateTimeBehavior=convertToNull*
 <br/>

2. Update `memcache.active.host` value From *memcached.active.host=mc01* To *memcached.active.host=mc01.vprofile.in*
3. Update `rabbitmq.address` value From *rabbitmq.address=rmq01* To *rabbitmq.address=rmq01.vprofile.in*

    **NOTE: Change `vprofile.in` if you assigned different name To your domain**

### STEP 10: BUILD THE ARTIFACT

1. Open your project repository in a git bash terminal in the vscode
2. Type the commands below in your git terminal To check maven and aws version respectively
   ```bash
   mvn --version
   ```
   ```bash
   aws --version
   ```
3. Type command below to build the artifact
    ```bash 
    mvn install
    ``` 
    ![Build Artifact the Artifact](images/mvn%20install%20To%20build%20the%20artifact.png)

### STEP 11: CONFIGURE AWS COMMAND LINE INTERFACE (CLI)

* Type the command below 
  
  ```sh
  aws configure
  ```
* aws access key ID: xxxxxxxxxxxxxxxxxxxxxxxxxxxx 
* aws secret access key: xxxxxxxxxxxxxxxxxxxxxxxxxxxx
* Default region name: us-east-1
* Default output format: json

### STEP 12: COPY THE ARTIFACT(.war) From `target/vprofile-v2.war` To S3

* Type the command below to copy the artifact
  
  ```sh
  aws s3 cp target/vprofile-v2.war s3://yourbucketname
  ```
![The Artifact](images/Copied%20the%20Artifact%20To%20S3.png)

### STEP 13: SSH INTO Tomcat-app01 EC2 INSTANCE

```sh
ssh -i Path/To/YouKeyPair user@InstancePublicIP
```
*Switch to root user terminal*
```sh
sudo -i
```
**Install AWS CLI on Tomcat-app01 Instance**

```sh
snap install aws-cli --classic
```
**Copy The Artifact From The S3 Bucket To `/tmp/` Directory**

```sh
aws s3 cp s3://bucketname/vprofile-v2.war /tmp/
```

### STEP 14: DEPLOYMENT OF THE ARTIFACT

Type the following commands in the `Tomcat-app01` terminal

```sh
systemctl stop tomcat10.service
```

```sh
systemctl daemon-reload
```
```sh
systemctl stop tomcat10.service
```
```sh
ls /var/lib/tomcat10/webapps/
```
* REMOVE `Root` DIRECTORY
```sh
rm -rf /var/lib/tomcat10/webapps/Root
```
* COPY THE ARTIFACT FROM `/tmp/` DIRECTORY TO `webapps` DIRECTORY
```sh
cp /tmp/vprofile-v2.war /var/lib/tomcat10/webapps/Root.war
```

* START `tomcat service` 
```sh
systemctl start tomcat10.service
```
* CHECK THE COPIED ARTIFACT
```sh
ls /var/lib/tomcat10/webapps
```

### STEP 15: TEST THE APPLICATION
  
  * *Update the `Tomcat-app01` Security Group and add PORT 8080 allow from `MY IP`* <br>
  
  ![Add PORT 8080 From MY IP](images/Test%20if%20the%20app%20is%20running%20using%20tomcat%20public%20ip%20and%20PORT%208080.png)
  * *Copy the `Tomcat-app01` EC2 Instance Public IP and put in the browser -> `yourInstancePublicIP:8080`*
  
  * You will See a Login Page
  
  ![Application Running](images/Running%20Webpage.png)

  ### STEP 16: CREATE APPLICATION LOAD BALANCER

  1. #### CREATE A TARGET GROUP
  
  * GO To The Target Groups and click on `Create Target Group`

  ![Create Target Group](images/Create%20Target%20Group-0.png)

  * Choose `instances` as a target type
  * Scroll down and give a name to the target group and give PORT `8080 -> HTTP` to Protocol Port
  
  ![Target Group Name and Port](images/Target%20Group%20name%20and%20Port.png)
  * Scroll down to `Health Checks` click on ADVANCED HEALTH CHECK SETTINGS and select override and type `8080` instead of `80` and go NEXT
  
  1. Find your `Tomcat-app01` instance under available instance and click the check mark *PORTS FOR THE SELECTED INSTANCES* make sure it is `8080` and click ***Including as pending*** below 
   ![Including Pending](images/Select%20Available%20Instance.png)
  2. Click Create target group
  
  ![Target Group Created](images/Created%20Target%20Group.png)
   
  ### STEP 17: CREATE LOAD BALANCER

  * Search *Elastic Load Balancing* and select application Load Balancer
  * Select Create Load Balancer
  ![Elastic Load Balancer](images/Create%20Elastic%20Load%20Balancer-0.png)
  * Give a name to the load balancer scroll down to **VPC** and select all the availability zones
  * On the Security group choose your load balancer security group
  
  ![Choose Security Group](images/Choose%20Load%20Balancer%20Security%20Group.png)
  * On the *Listeners and routing* select your target group: HTTP -> 80 -> yourTargetGroup
  * Select your certificate (ACM) if your are using ***HTTPS and SSL/TLS*** server certificate and click on create load balancer (*OPTIONAL*) 
  
  ![AWS ACM](images/Using%20SSL%20certificate%20in%20Load%20Balancer.png)

### STEP 18: GO To GODADDY.COM *(Do This if you have purchased Domain)*

* Select DNS on the Domain in the Domain Section
* Add New Record and Choose 
* Type: CNAME
* Name: anyNameYouPreferred
* Value: yourLoadBalancerEndpoint
* Click Save  

![GoDaddy.com](images/GoDaddy%20CName%20Record.png)

*Check your target group health by navigating to the target group you created <br/> **NOTE:** If it is `Unhealthy`, check your tomcat instance security group if it is allowing load balancer security group*
![Healthy Target Group](images/Created%20Target%20Group.png)

### STEP 19: ACCESSING YOUR WEBSITE USING LOAD BALANCER ENDPOINT

i. Copy your load balancer endpoint(DNS Name) and put it inside any Browser of your choice

![Application Webpage](images/Using%20the%20Load%20Balancer%20DNS%20Name%20to%20Access%20the%20application.png)

ii. To access the site using *HTTPS* copy your Godaddy DNS
![HTTPS](images/Accessing%20the%20application%20a%20HTTPs%20Connection.png)
  
 *USERNAME: admin_vp* <br>
 *PASSWORD: admin_vp*

 ### STEP 20: CREATE AMI IMAGE FROM *Tomcat-app01* EC2 INSTANCE

 * Select the *Tomcat-app01* EC2 instance
 * Go To Actions -> Image and Template -> Create Image

![AMI Image](images/Create%20AMI%20image.png)

 * Give image a name and image description and then create image
 * To check your AMI image go To: images -> AMIs 

![Created AMI image](images/Created%20AMI%20image.png)

### STEP 21: CREATE LAUNCH TEMPLATE

* Go To instance -> launch template -> Create launch template
![Launch Template](images/Create%20Launch%20Template.png)
* Give it a name and description
* Scroll down and select My AMI *(make sure Owned by me is selected)* and search for the AMI you created in *STEP 20* 
![My AMI](images/Find%20your%20AMI%20image%20you%20created.png)
* Select instance type, key pair, security group*(Tomcat-app01 instance security group)*
* Add instance and volume tags under *Resource and Tags*
* Click on *Advanced details* and under the *IAM instance profile* *Add IAM Role* To launch with the Role and scroll all the way down and click *Create launch template*
![Advanced details](images/Advanced%20Details%20add%20IAM%20Role%20To%20Launch%20With%20Role.png)
  
### STEP 22: CREATE AUTO SCALING OF *Tomcat-app01*

* Search for Auto Scaling and go to Auto Scaling groups -> Create Auto Scaling group
![Create Auto scaling group](images/Create%20Auto%20Scaling-0.png)
* Give it a name select the launch template you created in *STEP 21* and go *NEXT*
![Auto Scaling Group](images/Create%20Auto%20Scaling%20and%20choose%20launch%20template.png)
* On the *NEXT PAGE* select your network VPC and the number of availability zones you perferred and go *NEXT* 
* You will see Load Balancing on the *NEXT PAGE* choose *Attach to an existing load balancer*
* Under the Attach to an existing Load Balancer, choose and select your target group
![Existing load balancer](images/Attach%20your%20existing%20load%20balancer%20to%20auto%20scaling%20group.png)

* Under the `Health Checks` check the *Turn on Elastic Load Balancing health checks* and go *NEXT*
![ASG Health Check](images/Auto%20Scaling%20load%20balancer%20health%20check.png)
* Under group size keep it *ONE*
* Under Scaling -> Scale Limits
min = 1, max = 4 and Desired = 1
* Select *Target tracking Scaling Policy* under Automatic Scaling
* Under *Metric Type* choose *Average CPU utilization*
  Target Value -> 50
![Target Tracking](images/Auto%20Scaling%20Group%20Target%20Tracking%20scaling%20Policy.png)
* Go *NEXT*
* Add Notification 
* Select your SNS Topic *If you have already setup SNS Topic*
* Select all the event types and click *NEXT*
* Add Tags and go *NEXT*
![SNS Topic](images/Add%20notification.png)
* Review your settings
* Click Create Auto Scaling group on the *NEXT PAGE* and wait for it to finish creating
![Created Auto Scaling Group](images/Create%20Auto%20Scaling%20group.png)
* Go to Target Groups; select your target group -> Attributes -> Edit
![Stickness](images/Edit%20Target%20Group%20To%20Turn%20On%20Tickness.png)
* Under the Target Selection Configuration, Tick *Turn on stickness* and save changes
![Edit Target group](images/Target%20Selection%20Configuration%20For%20tickness.png)
* Go to Target Group and select your target group, check if you have two target groups if so deregister the one you will not need
* Go to instances and terminate *Tomcat-app01* EC2 instance
* **NOTE: The Auto Scaling Group should launch a new instance automatically**
* Then you can check in your browser to see if your application is running by visiting the load balancer *DNS* or your *GoDaddy domain* 

### 🧪 Final Testing

  **Login with**:

  - Username: admin_vp

  - Password: admin_vp
  
![Running the application](images/Accessing%20the%20application%20a%20HTTPs%20Connection.png)
