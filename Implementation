# AWS CloudFormation — Aurora Digital

## Business Use Case

Aurora Digital, an expanding online retailer specializing in luxury home goods, is
migrating its digital operations to AWS to handle increasing traffic and scale
automatically during major sales events without sacrificing performance or security.

**Why move from manual provisioning to CloudFormation?**

- **Automation** — Entire infrastructure deployed in a single command with zero manual clicks
- **Consistency** — Identical environments every deployment — eliminates "it worked last time" failures
- **Scalability** — Auto Scaling Group adjusts compute capacity automatically during peak traffic
- **Reliability** — Multi-AZ deployment ensures the site stays up if an entire AWS data center fails
- **Cost efficiency** — Pay-as-you-go scaling means no over-provisioned idle servers between sales events
- **Version control** — Infrastructure lives in a `.yml` file that can be reviewed, rolled back, and reused

## Architecture

## Tools & Services Used

- AWS CloudFormation
- AWS EC2 (Elastic Compute Cloud) — t2.micro
- Amazon Linux 2 (via SSM Parameter Store dynamic lookup)
- Apache HTTP Server (httpd)
- AWS EC2 Launch Template
- AWS Auto Scaling Group
- AWS Application Load Balancer (ALB)
- AWS EC2 Target Group
- AWS VPC, Subnets, Internet Gateway, Route Table
- AWS Security Groups

## Prerequisites

- AWS Account with CloudFormation, EC2, VPC, and ELB permissions
- An existing EC2 Key Pair in the target region
- AWS Console or AWS CLI to deploy the stack

---

## Implementation — Foundational Tier

The foundational tier proves the core concept: a CloudFormation template that
deploys a single EC2 instance with Apache installed automatically via UserData,
inside a Security Group that allows web traffic.

### Step 1: Parameters

Before any resources are created, CloudFormation collects two inputs at deploy time.

```yaml
Parameters:
  KeyName:
    Description: Name of SSH KeyPair
    Type: 'AWS::EC2::KeyPair::KeyName'
    ConstraintDescription: Provide the name of an existing SSH key pair
  LatestAmiId:
    Description: Latest Amazon Linux 2 AMI ID
    Type: 'AWS::SSM::Parameter::Value<AWS::EC2::Image::Id>'
    Default: /aws/service/ami-amazon-linux-latest/amzn2-ami-hvm-x86_64-gp2
```

| Parameter | Type | How Value Is Resolved |
|---|---|---|
| `KeyName` | `AWS::EC2::KeyPair::KeyName` | You supply it manually at deploy time |
| `LatestAmiId` | `AWS::SSM::Parameter::Value` | CloudFormation queries SSM Parameter Store and retrieves the current AMI ID automatically |

> **Two most important lines:**
> - `Type: 'AWS::EC2::KeyPair::KeyName'` — CloudFormation validates that the key pair you enter
>   actually exists in the account before the stack deploys. If the key doesn't exist, the stack
>   fails at validation, not mid-deployment
> - `Default: /aws/service/ami-amazon-linux-latest/amzn2-ami-hvm-x86_64-gp2` — this is an SSM
>   path, not an AMI ID. CloudFormation calls AWS's SSM service at deploy time and retrieves
>   whatever the current AMI ID is stored at that path. This ensures every deployment uses the
>   latest patched image automatically

### Step 2: Security Group

```yaml
InstanceSecurityGroup:
  Type: 'AWS::EC2::SecurityGroup'
  Properties:
    GroupName: MyDMZSecurityGroup
    GroupDescription: Allow HTTP only from ALB
    VpcId: !Ref MyVPC
    SecurityGroupIngress:
      - IpProtocol: tcp
        FromPort: 80
        ToPort: 80
        CidrIp: 0.0.0.0/0
```

| Port | Protocol | Source | Purpose |
|---|---|---|---|
| 80 | TCP | 0.0.0.0/0 | HTTP web traffic |

> **Security Note:** Opening port 80 to `0.0.0.0/0` is acceptable for the foundational tier to
> verify Apache is reachable. In the Complex tier this is locked down — instances only accept
> traffic from the ALB, not directly from the internet. Port 22 (SSH) is intentionally omitted
> from this template — SSH access is not required for a web server managed by CloudFormation.

### Step 3: UserData — Apache Installation

```yaml
UserData: !Base64 |
  #!/bin/bash
  yum update -y
  yum install -y httpd
  systemctl start httpd
  systemctl enable httpd
  echo "<h1>Aurora Digital</h1>" > /var/www/html/index.html
```

> **Two most important lines:**
> - `!Base64` — UserData must be Base64-encoded because AWS's instance metadata service, which
>   delivers the script to the instance at boot, only accepts Base64 format. CloudFormation
>   handles the encoding automatically — you write readable bash and the function converts it
> - `systemctl enable httpd` — ensures Apache restarts automatically if the instance reboots.
>   Without this, any reboot (including ASG instance replacement) would leave the instance
>   running but silently not serving traffic

---

## Implementation — Advanced Tier

The advanced tier introduces an Auto Scaling Group and distributes instances across
multiple Availability Zones for resilience. The ASG uses a Launch Template to define
the instance configuration.

### Step 1: Launch Template

The Launch Template replaces manually specifying instance settings at launch time.
All instance configuration lives in one reusable resource that the ASG references.

```yaml
InstanceLaunchTemplate:
  Type: 'AWS::EC2::LaunchTemplate'
  Properties:
    LaunchTemplateName: MyLaunchTemplate
    LaunchTemplateData:
      InstanceType: t2.micro
      ImageId: !Ref LatestAmiId
      KeyName: !Ref KeyName
      SecurityGroupIds:
        - !GetAtt InstanceSecurityGroup.GroupId
      UserData: !Base64 |
        #!/bin/bash
        yum update -y
        yum install -y httpd
        systemctl start httpd
        systemctl enable httpd
        echo "<h1>Aurora Digital</h1>" > /var/www/html/index.html
```

> **Two most important lines:**
> - `ImageId: !Ref LatestAmiId` — references the SSM-resolved AMI ID from the Parameters
>   section. Every instance the ASG launches will always use the latest Amazon Linux 2 image
> - `SecurityGroupIds: - !GetAtt InstanceSecurityGroup.GroupId` — when using a custom VPC you
>   must reference the Security Group by its ID, not its name. `!GetAtt` retrieves the GroupId
>   attribute of the security group resource

> **Cost Note:** `t2.micro` is free tier eligible — 750 hours/month. With a minimum of 2
> instances running simultaneously, you consume ~1,500 hours/month. Free tier only covers
> 750 hours for a single account, so running 2+ instances will incur a small charge
> (~$0.0116/hour per instance beyond free tier). Terminate the stack when not in use.

### Step 2: Auto Scaling Group

```yaml
MyAutoScalingGroup:
  Type: 'AWS::AutoScaling::AutoScalingGroup'
  DependsOn: [SubnetRouteTableAssociation1, SubnetRouteTableAssociation2, SubnetRouteTableAssociation3]
  Properties:
    AutoScalingGroupName: MyAutoScalingGroup
    LaunchTemplate:
      LaunchTemplateId: !Ref InstanceLaunchTemplate
      Version: !GetAtt InstanceLaunchTemplate.LatestVersionNumber
    MinSize: 2
    MaxSize: 5
    DesiredCapacity: 2
    VPCZoneIdentifier:
      - !Ref MySubnet1
      - !Ref MySubnet2
      - !Ref MySubnet3
    TargetGroupARNs:
      - !Ref MyTargetGroup
```

| Setting | Value | Reason |
|---|---|---|
| `MinSize` | 2 | Site never drops below 2 instances — single instance failure doesn't cause downtime |
| `MaxSize` | 5 | Caps spend during traffic spikes — prevents runaway scaling costs |
| `DesiredCapacity` | 2 | Normal operating state — 2 instances running at all times |
| `VPCZoneIdentifier` | All 3 subnets | Distributes instances across 3 AZs for multi-AZ resilience |

> **Two most important lines:**
> - `DependsOn: [SubnetRouteTableAssociation1, SubnetRouteTableAssociation2, SubnetRouteTableAssociation3]`
>   — forces CloudFormation to finish associating all subnets to the route table before launching
>   any instances. Without this, an instance could boot into a subnet with no internet route,
>   causing the UserData `yum install` to fail silently with no Apache installed
> - `Version: !GetAtt InstanceLaunchTemplate.LatestVersionNumber` — always uses the most recent
>   version of the Launch Template. If you update the template, new instances launched by the
>   ASG will automatically use the updated configuration

> **Security Note:** `VPCZoneIdentifier` must reference all three subnets to achieve true
> multi-AZ resilience. Listing only one subnet creates a single point of failure — if that
> Availability Zone experiences an outage, the ASG cannot launch replacements anywhere else
> and the site goes down entirely.

---

## Implementation — Complex Tier

The complex tier builds a fully isolated custom VPC from scratch with its own
networking stack, then places the ASG behind an Application Load Balancer.
Users reach the site exclusively through the ALB — direct access to EC2
instance public IPs is blocked at the security group level.

### Step 1: VPC and Networking

```yaml
MyVPC:
  Type: 'AWS::EC2::VPC'
  Properties:
    CidrBlock: 10.10.0.0/16
    EnableDnsSupport: true
    EnableDnsHostnames: true
```

```yaml
PublicRoute:
  Type: 'AWS::EC2::Route'
  Properties:
    RouteTableId: !Ref PublicRouteTable
    DestinationCidrBlock: 0.0.0.0/0
    GatewayId: !Ref InternetGateway
```

| Resource | Value | Purpose |
|---|---|---|
| VPC CIDR | `10.10.0.0/16` | Private IP space — up to 65,534 addresses |
| Subnet 1 | `10.10.1.0/24` — AZ index 0 | 256 addresses, first AZ |
| Subnet 2 | `10.10.2.0/24` — AZ index 1 | 256 addresses, second AZ |
| Subnet 3 | `10.10.3.0/24` — AZ index 2 | 256 addresses, third AZ |
| Internet Gateway | Attached to VPC | Required for any internet traffic |
| Route Table | `0.0.0.0/0 → IGW` | Sends all outbound traffic to the internet |
| Route Associations | All 3 subnets | Activates the route for each subnet |

> **Two most important lines:**
> - `DestinationCidrBlock: 0.0.0.0/0` — the catch-all route that sends any traffic not destined
>   for the local VPC to the Internet Gateway. Without this route, instances have no path to the
>   internet and UserData scripts cannot download packages
> - `AvailabilityZone: !Select [0, !GetAZs '']` — `!GetAZs ''` calls the AWS API at deploy time
>   and returns a live list of all AZs in the current region. `!Select [0, ...]` picks the first
>   one. This makes the template region-agnostic — it works in us-east-1, us-west-2, or any
>   other region without hardcoding AZ names

### Step 2: Security Groups — DMZ Pattern

Two security groups implement a DMZ architecture: the ALB faces the internet,
and the EC2 instances only accept traffic from the ALB.

```yaml
ALBSecurityGroup:
  SecurityGroupIngress:
    - IpProtocol: tcp
      FromPort: 80
      ToPort: 80
      CidrIp: 0.0.0.0/0       # Internet → ALB: open

InstanceSecurityGroup:
  SecurityGroupIngress:
    - IpProtocol: tcp
      FromPort: 80
      ToPort: 80
      SourceSecurityGroupId: !GetAtt ALBSecurityGroup.GroupId   # ALB → EC2 only
```

| Security Group | Allows Inbound From | Purpose |
|---|---|---|
| `MyALBSecurityGroup` | `0.0.0.0/0` port 80 | Public-facing door — accepts all HTTP traffic |
| `MyDMZSecurityGroup` | ALB Security Group ID only | Instances hidden behind ALB — no direct public access |

> **Security Note:** `SourceSecurityGroupId: !GetAtt ALBSecurityGroup.GroupId` is the most
> important security decision in this template. Instead of trusting an IP range — which can
> change or be spoofed — the instance security group trusts a specific AWS resource identity.
> Only traffic that has passed through the ALB's security group is allowed to reach the
> instances. Even though instances have public IPs (from `MapPublicIpOnLaunch: true`),
> no browser or attacker can reach them directly on port 80.

### Step 3: Application Load Balancer

The ALB requires three resources working together: the load balancer itself,
a Target Group to register instances into, and a Listener to define the routing rule.

```yaml
MyTargetGroup:
  Type: 'AWS::ElasticLoadBalancingV2::TargetGroup'
  Properties:
    Port: 80
    Protocol: HTTP
    VpcId: !Ref MyVPC
    TargetType: instance
    HealthCheckProtocol: HTTP
    HealthCheckPath: /

MyALB:
  Type: 'AWS::ElasticLoadBalancingV2::LoadBalancer'
  Properties:
    Scheme: internet-facing
    SecurityGroups:
      - !GetAtt ALBSecurityGroup.GroupId
    Subnets:
      - !Ref MySubnet1
      - !Ref MySubnet2
      - !Ref MySubnet3

MyALBListener:
  Type: 'AWS::ElasticLoadBalancingV2::Listener'
  Properties:
    LoadBalancerArn: !Ref MyALB
    Port: 80
    Protocol: HTTP
    DefaultActions:
      - Type: forward
        TargetGroupArn: !Ref MyTargetGroup
```

| Resource | Role |
|---|---|
| Target Group | Maintains the list of healthy EC2 instances. ALB only routes to instances that pass health checks |
| Load Balancer | The public-facing entry point. Spans all 3 subnets for multi-AZ availability |
| Listener | The rule engine — listens on port 80 and forwards all traffic to the Target Group |

> **Two most important lines:**
> - `HealthCheckPath: /` — the ALB sends a GET request to `/` on each instance every 30 seconds.
>   If an instance stops responding, the ALB stops sending traffic to it and the ASG replaces it.
>   This is what makes the architecture self-healing
> - `Scheme: internet-facing` — makes the ALB reachable from the public internet. The alternative
>   is `internal`, which is only reachable from within the VPC. This must be `internet-facing`
>   for customers to reach the Aurora Digital site

> **Cost Note:** Application Load Balancers are **not free tier eligible**. Cost is
> roughly **$16/month** at minimal traffic.
> Delete the CloudFormation stack when the lab is complete to stop ALB charges immediately.

### Step 4: Stack Outputs

```yaml
Outputs:
  ALBDNSName:
    Description: DNS name of the Application Load Balancer
    Value: !GetAtt MyALB.DNSName
```

The ALB DNS name is randomly generated by AWS at deploy time. The Outputs section
surfaces it directly on the CloudFormation stack page the moment deployment completes.

**Verify the site:**
Navigate to `http://[ALBDNSName]` in an incognito browser window.

Confirmed: `<h1>Aurora Digital</h1>`

> **Important:** Use `http://` not `https://`. Only port 80 was opened on the ALB security
> group. Navigating to `https://` (port 443) will time out since no HTTPS listener was
> configured. This is an intentional scope decision for this project.

---

## Full Template

```yaml
AWSTemplateFormatVersion: 2010-09-09

Description: Template to create an EC2 instance and enable Apache Web Server

Parameters:
  KeyName:
    Description: Name of SSH KeyPair
    Type: 'AWS::EC2::KeyPair::KeyName'
    ConstraintDescription: Provide the name of an existing SSH key pair
  LatestAmiId:
    Description: Latest Amazon Linux 2 AMI ID
    Type: 'AWS::SSM::Parameter::Value<AWS::EC2::Image::Id>'
    Default: /aws/service/ami-amazon-linux-latest/amzn2-ami-hvm-x86_64-gp2

Resources:
  MyVPC:
    Type: 'AWS::EC2::VPC'
    Properties:
      CidrBlock: 10.10.0.0/16
      EnableDnsSupport: true
      EnableDnsHostnames: true
      Tags:
        - Key: Name
          Value: MyVPC
  MySubnet1:
    Type: 'AWS::EC2::Subnet'
    Properties:
      CidrBlock: 10.10.1.0/24
      VpcId: !Ref MyVPC
      AvailabilityZone: !Select [0, !GetAZs '']
      MapPublicIpOnLaunch: true
      Tags:
        - Key: Name
          Value: MySubnet1
  MySubnet2:
    Type: 'AWS::EC2::Subnet'
    Properties:
      CidrBlock: 10.10.2.0/24
      VpcId: !Ref MyVPC
      AvailabilityZone: !Select [1, !GetAZs '']
      MapPublicIpOnLaunch: true
      Tags:
        - Key: Name
          Value: MySubnet2
  MySubnet3:
    Type: 'AWS::EC2::Subnet'
    Properties:
      CidrBlock: 10.10.3.0/24
      VpcId: !Ref MyVPC
      AvailabilityZone: !Select [2, !GetAZs '']
      MapPublicIpOnLaunch: true
      Tags:
        - Key: Name
          Value: MySubnet3
  InternetGateway:
    Type: 'AWS::EC2::InternetGateway'
    Properties:
      Tags:
        - Key: Name
          Value: MyInternetGateway
  AttachGateway:
    Type: 'AWS::EC2::VPCGatewayAttachment'
    Properties:
      VpcId: !Ref MyVPC
      InternetGatewayId: !Ref InternetGateway
  PublicRouteTable:
    Type: 'AWS::EC2::RouteTable'
    Properties:
      VpcId: !Ref MyVPC
      Tags:
        - Key: Name
          Value: MyPublicRouteTable
  PublicRoute:
    Type: 'AWS::EC2::Route'
    Properties:
      RouteTableId: !Ref PublicRouteTable
      DestinationCidrBlock: 0.0.0.0/0
      GatewayId: !Ref InternetGateway
  SubnetRouteTableAssociation1:
    Type: 'AWS::EC2::SubnetRouteTableAssociation'
    Properties:
      SubnetId: !Ref MySubnet1
      RouteTableId: !Ref PublicRouteTable
  SubnetRouteTableAssociation2:
    Type: 'AWS::EC2::SubnetRouteTableAssociation'
    Properties:
      SubnetId: !Ref MySubnet2
      RouteTableId: !Ref PublicRouteTable
  SubnetRouteTableAssociation3:
    Type: 'AWS::EC2::SubnetRouteTableAssociation'
    Properties:
      SubnetId: !Ref MySubnet3
      RouteTableId: !Ref PublicRouteTable
  ALBSecurityGroup:
    Type: 'AWS::EC2::SecurityGroup'
    Properties:
      GroupName: MyALBSecurityGroup
      GroupDescription: Allow HTTP from internet to ALB
      VpcId: !Ref MyVPC
      SecurityGroupIngress:
        - IpProtocol: tcp
          FromPort: 80
          ToPort: 80
          CidrIp: 0.0.0.0/0
  InstanceSecurityGroup:
    Type: 'AWS::EC2::SecurityGroup'
    Properties:
      GroupName: MyDMZSecurityGroup
      GroupDescription: Allow HTTP only from ALB
      VpcId: !Ref MyVPC
      SecurityGroupIngress:
        - IpProtocol: tcp
          FromPort: 80
          ToPort: 80
          SourceSecurityGroupId: !GetAtt ALBSecurityGroup.GroupId
  InstanceLaunchTemplate:
    Type: 'AWS::EC2::LaunchTemplate'
    Properties:
      LaunchTemplateName: MyLaunchTemplate
      LaunchTemplateData:
        InstanceType: t2.micro
        ImageId: !Ref LatestAmiId
        KeyName: !Ref KeyName
        SecurityGroupIds:
          - !GetAtt InstanceSecurityGroup.GroupId
        UserData: !Base64 |
          #!/bin/bash
          yum update -y
          yum install -y httpd
          systemctl start httpd
          systemctl enable httpd
          echo "<h1>Aurora Digital</h1>" > /var/www/html/index.html
  MyTargetGroup:
    Type: 'AWS::ElasticLoadBalancingV2::TargetGroup'
    Properties:
      Name: MyTargetGroup
      Port: 80
      Protocol: HTTP
      VpcId: !Ref MyVPC
      TargetType: instance
      HealthCheckProtocol: HTTP
      HealthCheckPath: /
  MyALB:
    Type: 'AWS::ElasticLoadBalancingV2::LoadBalancer'
    Properties:
      Name: MyALB
      Scheme: internet-facing
      SecurityGroups:
        - !GetAtt ALBSecurityGroup.GroupId
      Subnets:
        - !Ref MySubnet1
        - !Ref MySubnet2
        - !Ref MySubnet3
  MyALBListener:
    Type: 'AWS::ElasticLoadBalancingV2::Listener'
    Properties:
      LoadBalancerArn: !Ref MyALB
      Port: 80
      Protocol: HTTP
      DefaultActions:
        - Type: forward
          TargetGroupArn: !Ref MyTargetGroup
  MyAutoScalingGroup:
    Type: 'AWS::AutoScaling::AutoScalingGroup'
    DependsOn: [SubnetRouteTableAssociation1, SubnetRouteTableAssociation2, SubnetRouteTableAssociation3]
    Properties:
      AutoScalingGroupName: MyAutoScalingGroup
      LaunchTemplate:
        LaunchTemplateId: !Ref InstanceLaunchTemplate
        Version: !GetAtt InstanceLaunchTemplate.LatestVersionNumber
      MinSize: 2
      MaxSize: 5
      DesiredCapacity: 2
      VPCZoneIdentifier:
        - !Ref MySubnet1
        - !Ref MySubnet2
        - !Ref MySubnet3
      TargetGroupARNs:
        - !Ref MyTargetGroup

Outputs:
  ALBDNSName:
    Description: DNS name of the Application Load Balancer
    Value: !GetAtt MyALB.DNSName
```

---

## Security Findings & Next Steps

| Finding | Risk | Remediation |
|---|---|---|
| HTTP only — no HTTPS | High — customer traffic is unencrypted | Add ACM SSL certificate + HTTPS listener on ALB (port 443) |
| No scaling policy defined | Medium — ASG scales manually only | Add CloudWatch CPU alarms tied to scale-in/scale-out policies |
| No WAF on ALB | Medium — vulnerable to web attacks | Attach AWS WAF to the ALB (not free-tier eligible) |
| No access logs on ALB | Low — no traffic visibility | Enable ALB access logs to S3 for monitoring and forensics |
| `MapPublicIpOnLaunch: true` | Low — instances have public IPs even though they're blocked | Move instances to private subnets with NAT Gateway (cost tradeoff) |

## What I Learned

- CloudFormation deploys infrastructure as code — the entire stack is defined in one `.yml`
  file that can be version-controlled, reviewed, and redeployed identically every time
- The `DependsOn` attribute controls deployment order — without it, CloudFormation may
  launch instances before their subnets have internet routes, causing silent UserData failures
- The DMZ security pattern uses two security groups: one facing the internet (ALB) and one
  that only accepts traffic from the first (EC2 instances). This prevents direct public access
  to instances even when they have public IPs
- `!GetAZs ''` calls the AWS API live at deploy time — it is not referencing hardcoded AZ names
  from the template. This makes the template portable across any AWS region
- `SourceSecurityGroupId` trusts a resource identity, not an IP range — a fundamentally stronger
  security posture than CIDR-based rules which can be spoofed or become stale
- An ALB requires three separate resources: the load balancer, a target group, and a listener.
  All three must exist and reference each other for traffic to flow
- CloudFormation rolls back all resources automatically on failure — nothing is left running
  to incur costs while debugging

## Next Iterations

- [x] Foundational — EC2 instance with Apache via UserData and Security Group
- [x] Advanced — Auto Scaling Group across 3 AZs with Launch Template
- [x] Complex — Custom VPC, ALB, DMZ security pattern, full networking stack
