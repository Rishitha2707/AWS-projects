Title: Deploy Java Web Application in AWS using CloudFormation

🔧 1. Create Stack using AWS CloudFormation

Save your YAML script locally as vpc.yaml

```
AWSTemplateFormatVersion: 2010-09-09
Description: VPC template
# Resources section
Resources:
  myVPC:
    Type: AWS::EC2::VPC
    Properties:
      CidrBlock: 10.0.0.0/19
      EnableDnsSupport: 'true'
      EnableDnsHostnames: 'true'
      Tags:
       - Key: Name
         Value: Aja-VPC

  myInternetGateway:
    Type: AWS::EC2::InternetGateway
    Properties:
      Tags:
      - Key: Name
        Value: Aja-IGW

  AttachGateway:
    Type: AWS::EC2::VPCGatewayAttachment
    Properties:
      VpcId:
         Ref: myVPC
      InternetGatewayId:
         Ref: myInternetGateway

 
  myPubSubnet:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref myVPC
      CidrBlock: 10.0.0.0/20
      AvailabilityZone: "us-east-1a"
      Tags:
      - Key: Name
        Value: Aja-app-sub

  myPvtSubnet:
    Type: AWS::EC2::Subnet
    Properties:
      VpcId: !Ref myVPC
      CidrBlock: 10.0.16.0/20
      AvailabilityZone: "us-east-1b"
      Tags:
      - Key: Name
        Value: Aja-db-sub

  myPUBRouteTable:
    Type: AWS::EC2::RouteTable
    Properties:
      VpcId:  
        Ref: myVPC
      Tags:
      - Key: Name
        Value: Aja-app-rt

  myPvtRouteTable:
    Type: AWS::EC2::RouteTable
    Properties:
      VpcId:  
        Ref: myVPC
      Tags:
      - Key: Name
        Value: Aja-db-rt

  myPUBSubnetRouteTableAssociation:
    Type: AWS::EC2::SubnetRouteTableAssociation
    Properties:
      SubnetId:
        Ref: myPubSubnet
      RouteTableId:
        Ref: myPUBRouteTable

  myPvtSubnetRouteTableAssociation:
    Type: AWS::EC2::SubnetRouteTableAssociation
    Properties:
      SubnetId:
        Ref: myPvtSubnet
      RouteTableId:
        Ref: myPvtRouteTable

  MyIGWRoute:
    Type: AWS::EC2::Route
    Properties:
      RouteTableId: !Ref myPUBRouteTable
      DestinationCidrBlock: 0.0.0.0/0
      GatewayId: !Ref  myInternetGateway
```

Check if all the resources from the stack are created successfully.
1. Your VPC
2. Subnets
3. Route tables
4. Internet gateway

Create a NAT gateway and attach it to the Private Route table.
Allocate an Elastic IP to the gateway.

Note: YOu have to assign it to the Public server and attach it to the private route table.

2. Create EC2 Instances
Public EC2 Instance (Deployment Server):
Create a public EC2 instance (Amazon Linux 2 preferred) in the VPC and public subnet created by the stack.

Open ports: 22 (SSH), 3306 (MySQL), and 8080 (Tomcat) in the public security group.

Private EC2 Instance:
Launch a second EC2 instance in the private subnet of the same VPC.

Use a private security group that allows SSH access only from the public EC2 instance.

3. Connect the Public instance to the terminal

🔒 4. Jump Host Configuration
1. Copy your .pem from your local to server using: 
        scp -i .pem .pem host-server@publicIP

2. Go to your public server and check if the file is copied to the server using:
        ls

3. Change the permission of the .pem using:
       chmod 400 .pem

4. Connect the private server using jump host
       ssh -i "your-key.pem" ec2-user@<Private_EC2_Private_IP>

Now you are connected to the Private instance.


☕ 5. Deploy Java App Using Maven and Tomcat
On Public EC2 Instance:
a. Install Java, Git, Maven, Tomcat:

    sudo yum update -y
    sudo yum install openjdk-17-jre -y
    sudo yum install maven -y
    sudo yum install git -y

b. Clone the java code from your git repository to local repository.

    sudo git clone https://github.com/Ai-TechNov/aws-rds-java.git
    cd aws-rds-java
    

c. Install and configure Tomcat:

    wget https://downloads.apache.org/tomcat/tomcat-9/v9.0.86/bin/apache-tomcat-9.0.86.tar.gz
    tar -xvzf apache-tomcat-9.0.86.tar.gz
    sudo mv apache-tomcat-9.0.86 /opt/tomcat
    chmod +x /opt/tomcat/bin/*.sh

Create a RDS with the created VPC and private subnet
choose the availability Zone in which your private subnet is created.

a. Open the aws-rds-java and edit the file by changing endpoint with your RDS endpoint:
   
    sudo vi src/main/webapp/login.jsp
    sudo vi src/main/webapp/userRegistration.jsp

Edit your pokm.xml file by changing the java into 17 version and mysql into 8th version:

    <project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0
                             http://maven.apache.org/xsd/maven-4.0.0.xsd">

    <modelVersion>4.0.0</modelVersion>

    <groupId>com.javawebtutor</groupId>
    <artifactId>LoginWebApp</artifactId>
    <version>1.0-SNAPSHOT</version>
    <packaging>war</packaging>

    <name>LoginWebApp Maven Webapp</name>
    <url>http://maven.apache.org</url>

    <properties>
        <maven.compiler.source>17</maven.compiler.source>
        <maven.compiler.target>17</maven.compiler.target>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
    </properties>

    <dependencies>
        <!-- Servlet API provided by container like Tomcat -->
        <dependency>
            <groupId>javax.servlet</groupId>
            <artifactId>servlet-api</artifactId>
            <version>2.5</version>
            <scope>provided</scope>
        </dependency>

        <!-- MySQL JDBC driver -->
        <dependency>
            <groupId>mysql</groupId>
            <artifactId>mysql-connector-java</artifactId>
            <version>8.0.33</version>
        </dependency>

        <!-- Unit testing -->
        <dependency>
            <groupId>junit</groupId>
            <artifactId>junit</artifactId>
            <version>4.13.2</version>
            <scope>test</scope>
        </dependency>
    </dependencies>

    <build>
        <finalName>LoginWebApp</finalName>
        <plugins>
            <!-- Java 17 compiler support -->
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-compiler-plugin</artifactId>
                <version>3.11.0</version>
                <configuration>
                    <source>17</source>
                    <target>17</target>
                </configuration>
            </plugin>

            <!-- WAR packaging -->
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-war-plugin</artifactId>
                <version>3.3.2</version>
                <configuration>
                    <failOnMissingWebXml>false</failOnMissingWebXml>
                </configuration>
            </plugin>
        </plugins>
    </build>
</project>

Build the Java code:

    mvn package


A .war file has been generated in the target directory.

Deploy .war to Tomcat:
    cp target/*.war /opt/tomcat/webapps/

Start Tomcat:
    /opt/tomcat/bin/startup.sh

🛢️ 6. MySQL Setup on Private EC2
 
   a. SSH into private EC2 from public EC2:
      ssh -i "your-key.pem" ec2-user@<Private_EC2_Private_IP>

   b. Install MySQL Server:
      sudo yum update -y
      sudo yum install mysql -y

   c. Create mysql environment by connection with rds end point:
      mysql -h <hostname endpoint> -u admin -p

      enter password
      Your mysql environment is created.

   d. Create database, and schema:
      create database jwt;
      use jwt;
      CREATE TABLE USER (
        id INT(10) UNSIGNED NOT NULL AUTO_INCREMENT,
        first_name VARCHAR(45) NOT NULL,
        last_name VARCHAR(45) NOT NULL,
        email VARCHAR(45) NOT NULL,
        username VARCHAR(45) NOT NULL,
        password VARCHAR(45) NOT NULL,
        regdate DATE NOT NULL,
        PRIMARY KEY (id)
        ) ENGINE=InnoDB DEFAULT CHARSET=latin1;

    Now your database is created to store the data.

✅ Final Testing
Ensure you can access the app at:
http://<public-ec2-ip>:8080/LoginWebApp/

Test form submission and data getting saved into the DB by checking the USER table in the DB.

