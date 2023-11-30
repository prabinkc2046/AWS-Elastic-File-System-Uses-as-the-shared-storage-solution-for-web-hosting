# AWS Elastic File System Uses as the shared storage solution for web hosting

In this guide, I will be using Elastic File System to provide a shared storage system for multiple web servers running Nginx. All necessary resources will be deployed using the AWS Command Line Interface (CLI), offering a valuable learning experience for those interested in mastering AWS CLI commands. I will deploy two EC2 instances, and these instances will access static files such as HTML, CSS, and JavaScript from the central storage. This allows all currently running instances and those added either through AutoScaling groups or manually to have access to the static files needed. This work demonstrates that Elastic File System offers great benefits in sharing common static files among multiple instances.

## VPC Creation
```bash
# Create a new VPC with CIDR block 10.0.0.0/16 in the ap-southeast-2 region, and tag it with the name 'MyVPC'.
# Store the resulting VPC ID in the variable vpc_id and print it.
vpc_id=$(aws ec2 create-vpc --cidr-block 10.0.0.0/16 --region ap-southeast-2 --tag-specifications 'ResourceType=vpc,Tags=[{Key=name,Value=MyVPC}]' --query Vpc.VpcId --output text)
echo $vpc_id

# Enable DNS hostnames for the created VPC using modify-vpc-attribute.
aws ec2 modify-vpc-attribute --vpc-id $vpc_id --enable-dns-hostnames

# Enable DNS hostnames for the created VPC using modify-vpc-attribute.
aws ec2 modify-vpc-attribute --vpc-id vpc-03563314f821becda --enable-dns-hostnames
```
[link](./aws-commands-used/create_vpc)

[Watch Demo](https://youtu.be/lE_TtPICQZQ)

## Subnetting
```bash
# Create a new VPC with CIDR block 10.0.0.0/16 in the ap-southeast-2 region, and tag it with the name 'MyVPC'.
# Store the resulting VPC ID in the variable vpc_id and print it.
vpc_id=$(aws ec2 create-vpc --cidr-block 10.0.0.0/16 --region ap-southeast-2 --tag-specifications 'ResourceType=vpc,Tags=[{Key=name,Value=MyVPC}]' --query Vpc.VpcId --output text)
echo $vpc_id

# Enable DNS hostnames for the created VPC using modify-vpc-attribute.
aws ec2 modify-vpc-attribute --vpc-id $vpc_id --enable-dns-hostnames
$ cat create_subnet 
# Print the VPC ID stored in the variable vpc_id
echo $vpc_id

# Create subnets in the specified VPC ($vpc_id) with CIDR blocks and availability zones.
aws ec2 create-subnet --vpc-id $vpc_id --cidr-block 10.0.1.0/24 --availability-zone ap-southeast-2a
aws ec2 create-subnet --vpc-id $vpc_id --cidr-block 10.0.2.0/24 --availability-zone ap-southeast-2b

```
[link](./aws-commands-used/create_subnet)

[Watch Demo](https://youtu.be/JTFDARPN52A)

## Internet Gateway
```bash
# Create an internet gateway in the ap-southeast-2 region and store the resulting ID in the variable internet_gateway_id.
internet_gateway_id=igw-01bdadc8a47591f14
echo igw-09a1c97a1e97d1631

# Print the VPC ID stored in the variable vpc_id.
echo vpc-03563314f821becda

# Attach the created internet gateway to the specified VPC (vpc-03563314f821becda).
aws ec2 attach-internet-gateway --internet-gateway-id igw-09a1c97a1e97d1631 --vpc-id vpc-03563314f821becda
```
[link](./aws-commands-used/create_internet_gateway)

[Watch Demo](https://youtu.be/0kOLLlbW5B8)

## Routing
```bash
# Create a route table in the specified VPC ($vpc_id) and store the resulting ID in the variable route_table_id.
route_table_id=$(aws ec2 create-route-table --vpc-id $vpc_id --query RouteTable.RouteTableId --output text)
echo $route_table_id

# Print the internet gateway ID and the route table ID.
echo $internet_gateway_id
echo $route_table_id

# Create a default route in the route table, directing traffic to the internet via the specified internet gateway.
aws ec2 create-route --destination-cidr-block 0.0.0.0/0 --gateway-id $internet_gateway_id --route-table-id $route_table_id
```
[link](./aws-commands-used/create_route)

[Watch Demo](https://youtu.be/YRSXGfnwZ0k)

### Subnet Association
```bash
# Retrieve a list of subnet IDs in the ap-southeast-2 region and store them in the variable subnets.
subnets=$(aws ec2 describe-subnets --region ap-southeast-2 --query Subnets[*].SubnetId --output text)
echo $subnets

# Iterate through each subnet and associate the previously created route table ($route_table_id) with it.
for subnet in ${subnets[@]}; do
    aws ec2 associate-route-table --route-table-id $route_table_id --subnet-id $subnet
done
```
[link](./aws-commands-used/associate_subnet)

[Watch Demo](https://youtu.be/8Kg-WCCRJk0)

## Security Group
```bash
# Print the VPC ID stored in the variable vpc_id.
echo $vpc_id

# Create a security group named 'mysecuritygroup' with a specific description, allowing HTTP from any IP, SSH from your IP, and NFS from the same security group.
aws ec2 create-security-group --group-name mysecuritygroup --description "allows http from any and ssh from my ip as well as allow nfs from this security group" --vpc-id $vpc_id

# Retrieve the ID of the created security group and print it.
group_id=$(aws ec2 describe-security-groups --region ap-southeast-2 --query SecurityGroups[*].[GroupId,GroupName] --output text | grep -i mysecuritygroup | awk '{print $1}')
echo $group_id

# Authorize ingress rules for the created security group: HTTP from any IP, SSH from your current IP, and NFS from the same security group.
aws ec2 authorize-security-group-ingress --group-id $group_id --protocol tcp --port 80 --cidr 0.0.0.0/0
my_ip=$(curl ipinfo.io/ip)
aws ec2 authorize-security-group-ingress --group-id $group_id --protocol tcp --port 22 --cidr "$my_ip/32"
aws ec2 authorize-security-group-ingress --group-id $group_id --protocol tcp --port 2049 --source-group $group_id

# Describe the details of the created security group.
aws ec2 describe-security-groups --group-ids $group_id
```
[link](./aws-commands-used/security_group)

[Watch Demo](https://youtu.be/BIOtx9AgfB0)

## Elastic File System (EFS)
```bash
# Create an Amazon EFS file system with specific configurations, including encryption, performance mode, and tags.
aws efs create-file-system --backup --creation-token myefs --encrypted --performance-mode generalPurpose --region ap-southeast-2 --throughput-mode elastic --tags Key=name,Value=myefs

# Retrieve the ID of the created EFS file system and print it.
file_system_id=$(aws efs describe-file-systems --region ap-southeast-2 --query 'FileSystems[*].[FileSystemId]' --output text)
echo $file_system_id

# Print the security group ID, and subnets.
echo $group_id
echo $subnets

# Iterate through each subnet and create an EFS mount target using the specified security group and file system ID.
for subnet in ${subnets[@]}; do 
    aws efs create-mount-target --security-groups $group_id --file-system-id $file_system_id --subnet-id $subnet
done
```
[link](./aws-commands-used/create_file_system)

[Watch Demo](https://youtu.be/fC7qMkRCvcE)

## Launch EC2 Instances
```bash
# Create an EC2 key pair named 'mykeypair' and save the private key to a file named 'mykeypair.pem'.
aws ec2 create-key-pair --key-name mykeypair --query "KeyMaterial" --output text > mykeypair.pem
sudo chmod 400 mykeypair.pem

# Launch an EC2 instance (webserver1) with specific configurations, including security group, key pair, tags, and subnet.
aws ec2 run-instances --image-id ami-0361bbf2b99f46c1d --instance-type t2.micro --security-group-ids $group_id --associate-public-ip-address --key-name mykeypair --tag-specifications 'ResourceType=instance,Tags=[{Key=name,Value=webserver1}]' --subnet-id $(aws ec2 describe-subnets --filters "Name=tag:name,Values=publicsn1" --query Subnets[0].SubnetId --output text) --count 1

# Launch another EC2 instance (webserver2) with similar configurations but a different name tag.
aws ec2 run-instances --image-id ami-0361bbf2b99f46c1d --instance-type t2.micro --security-group-ids $group_id --associate-public-ip-address --key-name mykeypair --tag-specifications 'ResourceType=instance,Tags=[{Key=name,Value=webserver2}]' --subnet-id $(aws ec2 describe-subnets --filters "Name=tag:name,Values=publicsn1" --query Subnets[0].SubnetId --output text) --count 1
```
[link](./aws-commands-used/run_instances)

[Watch Demo](https://youtu.be/3PEDBqGmD9M)

## Mounting step by step
```bash
# Describe the public IP address of the EC2 instance with the tag 'webserver1'.
aws ec2 describe-instances --filters "Name=tag:name,Values=webserver1" --query Reservations[0].Instances[0].PublicIpAddress --output text

# SSH into the first EC2 instance using the key pair 'mykeypair.pem'.
ssh -i mykeypair.pem ec2-user@13.211.143.191

# After logging in to the first instance, update packages, install NFS utilities, create a directory, and mount the EFS filesystem.
yum update -y
yum install -y nfs-utils
mkdir -p /mnt/share
mount -t nfs -o nfsvers=4.1,rsize=1048576,wsize=1048576,hard,timeo=600,retrans=2,noresvport fs-0ba58402725433817.efs.ap-southeast-2.amazonaws.com:/ /mnt/share/

# Display the mounted filesystem.
df -h
```
[link](./aws-commands-used/mount_manually)

[Watch Demo](https://youtu.be/ctsWv6EWU2g)

## Script Writing that will automate mounting processs
[link](./nfs_configurator)

[Watch Demo Video](https://youtu.be/aUpS-GPwVpI)

## Mounting  with Script
[Watch Demo Video](https://youtu.be/J8L8vxBC708)

## Configure Nginx to serve static files
[Watch Demo Video](https://youtu.be/odQANuBOn38)

## Magic of Elastic File System
[Watch Demo Video]()
