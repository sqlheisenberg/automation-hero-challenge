# Autmation Hero Test

Challenge:

Please provide an automated solution which:
1. Creates EC2 instance in AWS
2. Starts Nginx server on that EC2 with “Hello world” html
3. Creates AWS EFS and mounts it to the above EC2 instance

You can implement either 1 + 2 or 1 + 3, it's up to you!

## Solution
**To resolve this chalenge I will implement 1 and 2.**

### Decision 1: Infrastructure as Code using CloudFormation templates
For provisioning infrastructure I will use CloudFormation templates written in yaml notation.

### Decision 2: Separating resources into different templates
To ensure future improvements of CloudFormation templates, their easier management, easier integration into CI/CD workflows I decided to separate network, security and compute resources into three templates.

### Decision 3: Installing and setting up NGINX
To complete this chalange as fast is possible I decided to use UserData inside [web-server-ec2.yaml](/web-server-ec2.yaml) to install and setup nginx web server. 
For production environment this is not the best approach of installing web server and I would rather use some other tools like Ansible, AWS Systems Manager etc. 

### Decision 4: Create resources inside custom VPC
To follow AWS best practices I decided to create custom VPC in which I will spin up NGINX web server. 
Since all resources created later depends on VPC, when we are provisioning our infrastrucutre we should execute VPC cloud formation template first. After that check Output tab inside AWS Console and use values from there as input values for [web-server-security-group.yaml](/web-server-security-group.yaml]) template. 

### Decision 5: Security Group template  
I decided to use separate template for security group, that will ensure higher visibility in case of Pull Request reviews so that we can easily see which inbound rules are added. Since our web server is available thorugh public IP address we only allow HTTP and HTTPS traffic to our web server EC2 instance. Opening SSH port would be considered as potential security risk. 
Since output values from [web-server-security-group.yaml](/web-server-security-group.yaml) template are used for NGINX web server EC2 instance creation please check Output tab for values.


### Decision 6: Default AMI image for web-server.yaml CF temlate  
I decided to use Amazon Linux AMI for our base image. As you can notice inside [web-server.yaml](/web-server.yaml) CF template we have default AMI images mappings. You can pull image id's using following command from your terminal.
```
for region in $(aws ec2 describe-regions --query "Regions[].RegionName" --output text --region eu-central-1 --profile default); do ami=$(aws --region $region ec2 describe-images --filters "Name=name,Values=amzn2-ami-hvm-2.0.20211223.0-x86_64-gp2" --query "Images[0].ImageId" --output "text" --profile default); printf "'$region':\n  AMI: '$ami'\n"; done
```

NOTE: If you are using named awscli profiles update parameter `--profile` with your awscli profile name.
Latest Amazon Linux AMI relase notes can be found [here](https://aws.amazon.com/amazon-linux-2/release-notes/)

## Templates Execution  

### 1. vpc.yaml  
First create VPC infrastrucute by executing [vpc.yaml](/vpc.yaml) CloudFormation template.
Naming convention for VPC stack name is: automation-hero-vpc. 
You can use another stack name if you want.  

**Parameters:**  
Required parameters: CIDR block for our VPC.  

### 2. web-server-security-group.yaml  
Once when we have VPC created, execute [web-server-security-group.yaml](/web-server-security-group.yaml) CloudFormation template to create security group for our NGINX web server EC2 instance.
Naming convention for VPC stack name is: nginx-web-server-security-group. 
You can use another stack name if you want.  

**Parameters:**  
Name of VPC stack
Default value: automation-hero-vpc

### 3. web-server-ec2.yaml  
Last step is to create EC2 instance for our NGINX web server. To do that execute [web-server-ec2.yaml](/web-server-ec2.yaml) CloudFormation template. 
Naming convention for VPC stack name is: nginx-web-server

**Parameters:**  
Parent VPC stack
Default value: automation-hero-vpc

Parent Security Group Stack

Network Parameters  
Subnet  
Default value: PublicASubnet

EC2 Parameters

Name
Default value: nignx-web-server

InstanceType
Default value: t3.nano

RootVolumeSize
Default value: 8

Once when you got [web-server-ec2.yaml](/web-server-ec2.yaml) template executed open Output tab and copy/paste public ip address in your browser, you should see "Hello world" page.

