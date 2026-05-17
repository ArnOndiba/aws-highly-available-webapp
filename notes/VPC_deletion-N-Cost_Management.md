# AWS Cost Management & Infrastructure Cleanup

## Objective

This session focused on:

- reducing AWS credit consumption
- identifying hidden billable resources
- understanding AWS infrastructure dependencies
- learning proper VPC teardown workflow through AWS CLI

The original highly available web application project is still ongoing and has I have yet implemented:

- Load Balancers
- Target Groups
- Listeners
- Web application deployment
- Jenkins integration

This session was specifically used to clean up unused infrastructure before continuing the project.


# 1. Realization About AWS Billing

Initially, I was only stopping EC2 instances, however, I discovered that:

- stopping EC2 instances alone does **NOT** stop all AWS charges
- networking resources can continue billing even when compute instances are off

## Important Discovery

Resources such as:

- NAT Gateways
- Elastic IPs
- Load Balancers
- EBS volumes

may continue consuming credits.

## Key Learning

Cloud engineering also involves:

- infrastructure lifecycle management
- cost optimization
- teardown procedures


# 2. Understanding VPC Deletion Behavior

I attempted to delete the VPC directly:

```bash
aws ec2 delete-vpc --vpc-id vpc-0004aa8e36a774718 --region us-east-1
```

Received:

* DependencyViolation 


## Key Learning

A VPC cannot be deleted while dependent resources still exist inside it.

AWS enforces dependency-based infrastructure cleanup.


# 3. Internet Gateway Investigation

I checked Internet Gateway attached to the VPC:

```bash
aws ec2 describe-internet-gateways --filters "Name=attachment.vpc-id,Values=vpc-0004aa8e36a774718" --region us-east-1
```

## Learned

* Internet Gateways attach at VPC level
* they provide public internet connectivity for public subnet traffic

I detached Internet Gateway:

```bash
aws ec2 detach-internet-gateway --internet-gateway-id igw-02b8fa9960ac2d65d --vpc-id vpc-0004aa8e36a774718 --region us-east-1
```

I deleted Internet Gateway:

```bash
aws ec2 delete-internet-gateway --internet-gateway-id igw-02b8fa9960ac2d65d --region us-east-1
```

# 4. NAT Gateway Cleanup

I investigated NAT Gateways:

```bash
aws ec2 describe-nat-gateways --filter "Name=vpc-id,Values=vpc-0004aa8e36a774718" --region us-east-1
```

## Important Discovery

NAT Gateways:

* remain billable even after EC2 shutdown
* are one of the biggest hidden AWS costs for beginners

I deleted NAT Gateways as part of cost optimization.

## Key Learning

Private subnet outbound internet access comes through:

Private Subnet → Route Table → NAT Gateway → Internet Gateway → Internet


# 5. Security Group Cleanup

I investigated remaining Security Groups:

```bash
aws ec2 describe-security-groups --filters "Name=vpc-id,Values=vpc-0004aa8e36a774718" --region us-east-1 --query "SecurityGroups[*].[GroupId,GroupName]" --output table
```

I deleted custom Security Groups:

```bash
aws ec2 delete-security-group --group-id sg-xxxxxxxx --region us-east-1
```

## Key Learning

The default Security Group:

* cannot usually be manually removed
* persists with the VPC


# 6. Route Table Investigation

I investigated Route Tables:

```bash
aws ec2 describe-route-tables --filters "Name=vpc-id,Values=vpc-0004aa8e36a774718" --region us-east-1 --query "RouteTables[*].[RouteTableId,Associations[0].Main]" --output table
```

## Important Discovery

AWS VPCs always contain:

* one MAIN route table

The main route table:

* cannot be deleted
* cannot be disassociated

I deleted custom route tables:

```bash
aws ec2 delete-route-table --route-table-id rtb-xxxxxxxx --region us-east-1
```

---

# 7. Subnet Cleanup

I investigated remaining subnets:

```bash
aws ec2 describe-subnets --filters "Name=vpc-id,Values=vpc-0004aa8e36a774718" --region us-east-1 --query "Subnets[*].[SubnetId,AvailabilityZone]" --output table
```

I deleted unused subnets:

```bash 
aws ec2 delete-subnet --subnet-id subnet-xxxxxxxx --region us-east-1
```

## Key Learning

Subnets cannot be deleted while:

* ENIs
* NAT Gateways
* Load Balancers
* EC2 instances

still exist inside them.



# 8. Network Interface Investigation

I investigated hidden ENI dependencies:

```bash 
aws ec2 describe-network-interfaces --filters "Name=vpc-id,Values=vpc-0004aa8e36a774718" --region us-east-1 --query "NetworkInterfaces[*].[NetworkInterfaceId,Status,Description,SubnetId]" --output table
```

## Key Learning

Many AWS resources create hidden ENIs:

* NAT Gateways
* Load Balancers
* EC2 instances

which can block VPC deletion.


# 9. VPC Endpoint Dependency Discovery

While troubleshooting VPC deletion failures, an additional hidden dependency was discovered:VPC Endpoint

I investigated VPC Endpoints:

```bash
aws ec2 describe-vpc-endpoints --filters "Name=vpc-id,Values=vpc-0e3929127bbd35b70" --region us-east-1
```

Discovered an active S3 Gateway Endpoint:vpce-02c52893809c16132


### Important Discovery

VPC Endpoints:

* create dependencies inside a VPC
* can block VPC deletion
* may persist even after subnets and gateways are removed

Deleted the VPC Endpoint:

```bash
aws ec2 delete-vpc-endpoints --vpc-endpoint-ids vpce-02c52893809c16132 --region us-east-1
```

After removing the endpoint, the VPC deletion process completed successfully.


# 10. Final Infrastructure Teardown Understanding

By the end of the session, a much clearer understanding was developed regarding:

* AWS infrastructure dependencies
* teardown order
* hidden billing resources
* networking relationships
* lifecycle management

# 10. NAT Gateway Cost Optimization Discovery

While continuing the AWS project, it was discovered that stopping EC2 instances alone does not stop all AWS charges.

Major billing sources investigated:

- NAT Gateways
- Load Balancers
- Elastic IPs
- VPC Endpoints

Important realization:

```text
NAT Gateways continue billing independently
of EC2 instance state.
```

Deleted unused NAT Gateways:

```bash
aws ec2 delete-nat-gateway --nat-gateway-id <nat-id> --region us-east-1
```

### Key Learning

Cloud cost optimisation requires reviewing:
- networking resources
- managed services
- hidden dependencies

not only compute infrastructure.


# 11. Infrastructure Rebuild & NAT Recreation

Recreated NAT Gateway infrastructure for continued private subnet testing.

Architecture understanding:

Public Subnet
├── Internet Gateway
├── NAT Gateway
├── Application Load Balancer

Private Subnet
├── Jenkins EC2


Important realization:


NAT Gateway provides outbound internet access for private subnet instances.

This enabled:
- package installations
- Jenkins downloads
- system updates
- external repository access

while maintaining private subnet isolation.


# 12. Final Infrastructure & Lifecycle Understanding

By the end of the project, several major AWS infrastructure concepts became significantly clearer:

- infrastructure dependency hierarchy
- ALB traffic forwarding
- backend application ports
- Linux services
- ASG instance ephemerality
- infrastructure teardown workflows
- networking cost optimisation
- public vs private architecture

Major realization:

Cloud engineering is not only about deployment, but also:
- troubleshooting
- lifecycle management
- scalability
- automation
- cost optimisation
- dependency management



# Major Lessons Learned During This Session

* AWS infrastructure dependencies are layered and interconnected
* Stopping EC2 instances does not eliminate all AWS costs
* NAT Gateways can continue billing silently
* AWS infrastructure deletion follows dependency order
* Route tables determine subnet traffic behavior
* Internet Gateways operate at VPC level
* Security groups, ENIs, and subnets can block VPC deletion
* Main route tables cannot be deleted
* VPC Endpoints can silently block VPC deletion
* Cloud cost management is an important engineering skill
* AWS CLI provides deeper visibility into infrastructure relationships
* Successful infrastructure teardown requires systematic investigation
