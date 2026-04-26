# AWS CloudFormation — Aurora Digital
### CDA 03 | Infrastructure as Code | VPC · Auto Scaling · Application Load Balancer

---

## Scenario

Aurora Digital, a growing online retailer specializing in luxury home goods, needed cloud
infrastructure that could automatically scale during high-traffic sales events without manual
intervention. The company required a repeatable, version-controlled deployment process to
eliminate configuration drift between environments and reduce the risk of human error during
provisioning.

## Obstacle

The core challenge was building a fully isolated network from scratch — no VPC wizard shortcuts —
while simultaneously enforcing a security requirement that no customer could reach a web server
directly by IP. Every instance needed to sit behind a load balancer, with the infrastructure
capable of surviving an entire Availability Zone failure without taking the site down. All of
this had to be expressed in a single deployable configuration file.

## Action

Authored a CloudFormation template that provisioned a complete networking stack — custom VPC
(`10.10.0.0/16`), three public subnets across three Availability Zones, an Internet Gateway,
route table, and subnet associations — entirely through infrastructure as code. Implemented a
DMZ security pattern using two security groups: an ALB-facing group open to the internet, and
an instance-facing group that accepted traffic exclusively from the ALB's security group ID
rather than an IP range, preventing direct public access to EC2 instances. Configured an Auto
Scaling Group with a min of 2 and max of 5 `t2.micro` instances using a Launch Template with
dynamic AMI resolution via SSM Parameter Store, ensuring deployments always use the latest
patched Amazon Linux 2 image. Wired the ASG to an Application Load Balancer through a Target
Group with HTTP health checks, and used `DependsOn` to enforce that all subnet route associations
completed before any instance was allowed to launch.

## Result

The full infrastructure stack — VPC, subnets, IGW, routing, security groups, launch template,
ASG, target group, ALB, and listener — deployed successfully in a single CloudFormation stack
execution. Verified end-to-end traffic flow by navigating to the ALB DNS name in a browser and
confirming the Aurora Digital custom webpage was served. The ALB DNS name was surfaced
automatically via the stack Outputs section, eliminating the need to search the console
post-deployment.

## Troubleshoot

During template review, identified that `VPCZoneIdentifier` in the Auto Scaling Group referenced
only `MySubnet2` instead of all three subnets. While the stack deployed successfully and the ALB
spanned all three AZs correctly, the ASG could only launch instances into a single Availability
Zone — meaning an AZ failure during a sales event would take the site down entirely despite the
multi-AZ network architecture. Root cause was a scope mismatch between the ALB configuration
(which correctly referenced all three subnets) and the ASG configuration. Corrected by adding
`MySubnet1` and `MySubnet3` to `VPCZoneIdentifier`, aligning the ASG's placement with the ALB's
distribution.

---

## Tools & Services Used

- AWS CloudFormation
- AWS EC2 — t2.micro · Launch Template · Auto Scaling Group
- Amazon Linux 2 (SSM Parameter Store dynamic AMI resolution)
- Apache HTTP Server (httpd)
- AWS Application Load Balancer · Target Group · Listener
- AWS VPC · Subnets · Internet Gateway · Route Table · Security Groups

---

## Next Iterations

- [x] Foundational — EC2 instance with Apache via UserData and Security Group
- [x] Advanced — Auto Scaling Group across 3 AZs with Launch Template
- [x] Complex — Custom VPC, ALB, DMZ security pattern, full networking stack
