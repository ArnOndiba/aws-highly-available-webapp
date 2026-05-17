## Connect With Me

- X: https://x.com/0ndiba
- LinkedIn: https://www.linkedin.com/in/arnold-ondiba/
- GitHub: https://github.com/ArnOndiba

# Aws-Highly-Available-Webapp
Hands-on AWS cloud infrastructure project demonstrating highly available web architecture using VPC, public/private subnets, bastion hosts, load balancers, Auto Scaling Groups, and EC2 instances.

# Project Goal
The goal of this project is to build and understand a highly available AWS web infrastructure using hands-on implementation rather than memorization.

This project focuses on:
- AWS networking fundamentals
- Public and private subnet architecture
- Bastion host access
- Load balancing
- Auto Scaling Groups
- Linux server management
- Infrastructure troubleshooting

The entire learning process is being documented through GitHub, LinkedIn, and X.

# Services Used

- Amazon VPC
- EC2
- Auto Scaling Groups
- NAT Gateway
- Internet Gateway
- Route Tables
- Security Groups
- Bastion Host Architecture
- AWS CLI
- Linux (Ubuntu)
- Git & GitHub

# Architecture Overview
Initial Architecture Sketch:
![AWS Architecture](architecture/VPC_Environment_Infastructure.jpeg)
This architecture demonstrates a highly available AWS environment using:

- Public and private subnets
- Bastion host access
- Load balancer distribution
- Auto Scaling Groups
- Private EC2 web servers
- Internet Gateway
- NAT Gateway
- Multi-AZ deployment

## 1.Screenshots

## VPC Architecture

![VPC Architecture](screenshots/networking/vpc-architecture.png)
This VPC is designed across two Availability Zones for high availability. It includes public subnets for internet-facing resources and private subnets for application servers. The Internet Gateway allows public subnet resources to communicate with the internet, while NAT Gateways allow private instances to access the internet for outbound traffic without exposing them directly.


## Bastion Host Configuration

![Bastion Host](screenshots/networking/bastion-host.png)
The bastion host is placed in a public subnet and acts as a secure entry point into the private network. Instead of exposing private EC2 instances directly to the internet, SSH access first goes through the bastion host, then connects internally to private instances using their private IP addresses.


## Auto Scaling Group Configuration

![ASG Configuration](screenshots/networking/asg-configuration.png)
The Auto Scaling Group manages the private EC2 web servers. It maintains the desired number of instances across private subnets and can automatically replace unhealthy or terminated instances. This helps improve availability and supports scalable infrastructure design.

## Secure SSH Workflow

The bastion host was used as a secure jump server to access EC2 instances located inside private subnets.
![ASG Configuration](screenshots/networking/bastion-access-workflow-1.png)
![ASG Configuration](screenshots/networking/bastion-access-workflow-2.png)

Steps performed:
- SSH from local machine into bastion host
- Securely copied PEM key into bastion host using SCP
- Adjusted Linux file permissions using `chmod 400`
- SSH from bastion host into private EC2 instances using private IP addresses

This validated:
- private subnet isolation
- internal VPC communication
- SSH authentication workflow
- bastion host architecture

## Jenkins Integration & Private Application Hosting

To extend the project further, Jenkins was deployed inside a private EC2 instance behind the Application Load Balancer.

This introduced additional concepts involving:

- backend application ports
- ALB listener routing
- target groups
- health checks
- Linux services
- private subnet application hosting

### Jenkins Deployment

Jenkins was installed on a private EC2 instance using:

```bash
sudo apt update
sudo apt install fontconfig openjdk-21-jre -y
sudo apt install jenkins -y
```

The Jenkins service was started and enabled using:

```bash
sudo systemctl start jenkins
sudo systemctl enable jenkins
```

Verified Jenkins status:

```bash
sudo systemctl status jenkins
```

### Key Realisation

Jenkins runs independently as a Linux systemd service and automatically occupies port `8080`.


## Application Load Balancer Listener & Target Group Configuration
![Jenkins Deployment](screenshots/networking/ALB-Listner-config.png)

I configured the Application Load Balancer to forward traffic from:


ALB Listener :80
↓
Target Group :8080
↓
Private Jenkins Server :8080


This allowed browser access through the ALB DNS without exposing the private EC2 instance directly to the internet.


### Health Check Configuration

![TargetGroup Configuration](screenshots/networking/Target-group-health_checks.png)

The Jenkins target group health check path was configured as: **/login**
This ensured the ALB could correctly validate Jenkins health status.

### Port Troubleshooting Discovery

Attempting to run: *python3 -m http.server 8080*

returned: *OSError: [Errno 98] Address already in use*


because Jenkins was already running on port *8080*

### Final Result

![Jenkins Deployment](screenshots/networking/Jenkins-Upload.png)
Successfully deployed and accessed Jenkins through the AWS Application Load Balancer while keeping the Jenkins EC2 instance inside a private subnet architecture.


# Lessons Learned

- Public and private subnets are mainly defined by route table behavior.
- Internet Gateways connect the VPC to the public internet.
- NAT Gateways allow outbound internet access for private EC2 instances.
- Private EC2 instances can still serve webpages through a Load Balancer.
- Bastion hosts provide secure SSH access into private infrastructure.
- Traffic direction (inbound vs outbound) is very important in AWS networking.
- Route tables determine where subnet traffic is sent.
- Load Balancers require healthy registered targets to serve traffic properly.
- Application Load Balancer listeners can forward traffic to different backend ports
- Jenkins runs independently as a Linux service
- Health checks are critical for load balancer routing
- Auto Scaling Group instances are ephemeral and can be replaced at any time
- Jenkins runs as a Linux service
- Infrastructure automation becomes important in scalable cloud environments
