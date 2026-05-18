## Connect With Me

- X: https://x.com/0ndiba
- LinkedIn: https://www.linkedin.com/in/arnold-ondiba/
- GitHub: https://github.com/ArnOndiba# AWS Cost Management & Infrastructure Cleanup

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


# 13. Systematic AWS VPC Teardown Workflow (Dependency-Based Cleanup)

After encountering multiple dependency violations during VPC deletion, a more systematic teardown workflow was developed.

This process minimizes repeated dependency errors and provides a structured cleanup sequence.

---

## Step 1 — Disable Auto Scaling Groups FIRST

Important discovery:

Auto Scaling Groups can automatically recreate EC2 instances even after manual termination.
During teardown, terminated EC2 instances kept reappearing because the Auto Scaling Group was maintaining desired capacity automatically.

This caused repeated:

- ENI dependencies
- Security Group dependencies
- subnet dependencies

until the ASG desired capacity was reduced to 0.


Before deleting infrastructure:
- Identify the Asg you want to delete/update:

```bash
aws autoscaling describe-auto-scaling-groups --region us-east-1
```

- Then:

```bash
aws autoscaling update-auto-scaling-group --auto-scaling-group-name <asg-name> --min-size 0 --max-size 0 --desired-capacity 0 --region us-east-1
```

Then verify instances terminate:

```bash
aws ec2 describe-instances --filters "Name=vpc-id,Values=<vpc-id>" --region us-east-1 --query "Reservations[*].Instances[*].[InstanceId,State.Name]" --output table
```


## Step 2 — Investigate Remaining Infrastructure

### Check EC2 Instances
 In my case since I ssh into the private Ec2 through the bastion host, which is not controlled by the ASG I had to check for any instances that would be running inside the subnet

```bash
aws ec2 describe-instances --filters "Name=vpc-id,Values=<vpc-id>" --region us-east-1 --query "Reservations[*].Instances[*].[InstanceId,State.Name,SubnetId]" --output table
```


### Check NAT Gateways

```bash
aws ec2 describe-nat-gateways --filter "Name=vpc-id,Values=<vpc-id>" --region us-east-1 --query "NatGateways[*].[NatGatewayId,State,SubnetId]" --output table
```

Delete NAT Gateway:

```bash
aws ec2 delete-nat-gateway --nat-gateway-id <nat-id> --region us-east-1
```

---

### Check Load Balancers

Application Load Balancers create hidden ENIs inside subnets.

These ENIs can:

- prevent subnet deletion
- prevent security group deletion
- prevent VPC deletion

until the Load Balancer is fully removed.

```bash
aws elbv2 describe-load-balancers --region us-east-1 --query "LoadBalancers[?VpcId=='<vpc-id>'].[LoadBalancerName,State.Code]" --output table
```

Delete Load Balancer:

```bash
aws elbv2 delete-load-balancer --load-balancer-arn <alb-arn> --region us-east-1
```


### Check Elastic IPs

```bash
aws ec2 describe-addresses --region us-east-1 --query "Addresses[*].[AllocationId,PublicIp,AssociationId,NetworkInterfaceId]" --output table
```

Release Elastic IP:

```bash 
aws ec2 release-address --allocation-id <allocation-id> --region us-east-1
```


## Step 3 — Investigate Hidden ENI Dependencies

One major discovery was that many AWS services create hidden ENIs.

Check ENIs:

```bash
aws ec2 describe-network-interfaces --filters "Name=vpc-id,Values=<vpc-id>" --region us-east-1 --query "NetworkInterfaces[*].[NetworkInterfaceId,Description,Status,Attachment.InstanceId,SubnetId]" --output table
```

ENIs may belong to:

- NAT Gateways
- Load Balancers
- EC2 instances
- VPC Endpoints

These ENIs must disappear before subnet deletion succeeds.


## Step 4 — Check VPC Endpoints

```bash
aws ec2 describe-vpc-endpoints --filters "Name=vpc-id,Values=<vpc-id>" --region us-east-1
```

Delete endpoint:

```bash
aws ec2 delete-vpc-endpoints --vpc-endpoint-ids <vpce-id> --region us-east-1
```

---

## Step 5 — Delete Custom Security Groups

Check Security Groups:

```bash
aws ec2 describe-security-groups --filters "Name=vpc-id,Values=<vpc-id>" --region us-east-1 --query "SecurityGroups[*].[GroupId,GroupName]" --output table
```

Delete Security Group:

```bash
aws ec2 delete-security-group --group-id <sg-id> --region us-east-1
```

# Important: *Default security groups remain until the VPC itself is deleted.*


## Step 6 — Delete Subnets

Check subnets:

```bash
aws ec2 describe-subnets --filters "Name=vpc-id,Values=<vpc-id>" --region us-east-1 --query "Subnets[*].[SubnetId,AvailabilityZone]" --output table
```

Delete subnet:

```bash
aws ec2 delete-subnet --subnet-id <subnet-id> --region us-east-1
```


## Step 7 — Delete Custom Route Tables

Check route tables:

```bash
aws ec2 describe-route-tables --filters "Name=vpc-id,Values=<vpc-id>" --region us-east-1 --query "RouteTables[*].[RouteTableId,Associations[0].Main]" --output table
```

Delete custom route table:

```bash
aws ec2 delete-route-table --route-table-id <rtb-id> --region us-east-1
```

# Important: *Main route tables cannot be deleted manually.*


## Step 8 — Detach & Delete Internet Gateway

Check Internet Gateway:

```bash
aws ec2 describe-internet-gateways --filters "Name=attachment.vpc-id,Values=<vpc-id>" --region us-east-1
```

Detach IGW:

```bash
aws ec2 detach-internet-gateway --internet-gateway-id <igw-id> --vpc-id <vpc-id> --region us-east-1
```

Delete IGW:

```bash
aws ec2 delete-internet-gateway --internet-gateway-id <igw-id> --region us-east-1
```


## Step 9 — Delete VPC

Final deletion:

```bash
aws ec2 delete-vpc --vpc-id <vpc-id> --region us-east-1
```



# Final Major Realization

AWS infrastructure deletion is dependency-based.
Successful teardown requires understanding relationships between:

- Auto Scaling Groups
- EC2 instances
- ENIs
- Security Groups
- NAT Gateways
- Route Tables
- Internet Gateways
- Load Balancers
- VPC Endpoints
- Subnets

This teardown process demonstrated that AWS infrastructure behaves as an interconnected system rather than isolated resources.
Even after compute instances are removed, networking resources and managed services may continue existing independently.
Understanding infrastructure relationships is critical for:

- cloud cost management
- infrastructure lifecycle management
- dependency troubleshooting
- teardown automation
- operational awareness



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

# Final Engineering Takeaway

This teardown process demonstrated that AWS infrastructure behaves as an interconnected system rather than isolated resources.
Even after compute instances are removed, networking resources and managed services may continue existing independently.
Understanding infrastructure relationships is critical for:

- cloud cost management
- troubleshooting
- automation
- operational reliability
- infrastructure lifecycle management