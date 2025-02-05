# Secure Web Application Deployment on AWS with OpenVPN, VPC, EC2, Route 53, and CI/CD Pipeline

# Introduction

This project outlines the step-by-step process of deploying a secure web application on AWS. The setup includes configuring OpenVPN for secure access, building a VPC with public and private subnets, launching EC2 instances, setting up Route 53 for DNS management, creating an AWS Transit Gateway for network connectivity, deploying a dummy application, and automating the deployment process using AWS CodePipeline. The goal is to create a scalable, secure, and automated environment for hosting a web application.

# Step-by-Step Implementation

# Step 1: Set Up OpenVPN

Purpose: Secure connection to your AWS VPC.

1. Sign Up for OpenVPN

Visit the OpenVPN website

<img width="958" alt="vsa" src="https://github.com/user-attachments/assets/b2f5bc5e-5698-4ba5-a115-97cd8c79a62c" />


Go to OpenVPN.net and review the service offerings. Depending on your needs, you can:

OpenVPN Access Server (Commercial product, offers a web-based interface for easier management)

OpenVPN Community Edition (Fully open-source and free to use)

<img width="628" alt="bds" src="https://github.com/user-attachments/assets/b034160a-0371-4209-b961-4bd65f7cd18d" />


Choose the version you want to use

If you want a straightforward, managed solution, you may opt for a trial or license of OpenVPN Access Server.

If you prefer self-hosting and complete control, download the OpenVPN Community Edition.

Create or log in to your OpenVPN account

For Access Server, you’ll be prompted to create an account (if you don’t have one already).

For the Community Edition, you can proceed to download it directly.

2. Download and Install the OpenVPN Client

Access the Downloads page

<img width="656" alt="nkn" src="https://github.com/user-attachments/assets/52a03fd8-2009-4c0c-964f-6ad0c334b946" />

For Access Server: look for the Client Downloads section or the OpenVPN Connect client.

For Community Edition: you’ll need both server and client packages; they may be available in your operating system’s repository or on the OpenVPN download page.

Select the installer for your operating system

Windows: .exe installer

Linux: Usually installed from a package manager (e.g., sudo yum install openvpn on Red Hat or sudo apt-get install openvpn on Ubuntu).

Run the installer

Follow the on-screen instructions:

Enter a Stack name or leave it as is.

Note that the Activation Key is populated.

Select your VPC ID and Public Subnet ID.

Adjust the Instance Name and Instance Type if needed.

Select a Key Pair Name for SSH access.

Check the box to acknowledge that AWS CloudFormation might create IAM resources.

Click Create stack.

CREATE_IN_PROGRESS displays, and you can monitor the progress until CREATE_COMPLETE displays.

Once installed, you’ll typically see an OpenVPN GUI (on Windows) or an OpenVPN client icon (macOS/Linux) in your system tray or Applications folder.

3. Configure OpenVPN

Obtain configuration files

You’ll need .ovpn configuration files (or similar) from the OpenVPN server that corresponds to your AWS VPC environment.

<img width="710" alt="cds" src="https://github.com/user-attachments/assets/f0987f7a-077f-4cc7-ab79-e06b02822b88" />

If you’re setting up an Access Server in AWS, you can generate client profiles from the server’s web interface.

For the Community Edition, manually create server and client configs matching your AWS VPC network and security requirements.

Import the .ovpn configuration file

In the OpenVPN client GUI, look for an “Import” or “Add” option.

Select the .ovpn file you obtained from the server or your AWS environment.

Edit the configuration (if needed)

If you require specific DNS servers, routes, or pushing AWS VPC IP ranges, update the .ovpn file or the server config to reflect these details.

Make sure that the remote directive in the .ovpn file points to your AWS VPC’s public IP or DNS name.

4. Test the Connection

Start the VPN

In your OpenVPN client, click Connect using the imported configuration profile.

Enter any required username/password credentials (if configured).

Verify connectivity

Once connected, open a terminal or command prompt and run ping or curl to test resources inside your AWS VPC.

For example, ping 10.0.0.10 (if that’s an instance in your private VPC subnet).

Check routes

Verify that a route to your VPC subnets (e.g., 10.0.0.0/16) is established. You can run:

Linux: ip route show

Windows: route print

Look for the VPN gateway’s IP as the next hop to the VPC subnets.

# 2. Build VPC and EC2 Instances

2.1 Create a VPC

Access the AWS Console: Log in to the AWS Management Console.

Navigate to VPC Service: Go to Services > VPC.

Create VPC:

Click Your VPCs > Create VPC.

Name Tag: MyProjectVPC (or any descriptive name).

IPv4 CIDR Block: 10.0.0.0/16 (example).

Tenancy: Default.

Click Create VPC.

2.2 Create and Configure Subnets

Create Public Subnet:


Navigate to Subnets > Create Subnet.

Select your new VPC (MyProjectVPC).

Subnet name: MyPublicSubnet.

CIDR block: 10.0.1.0/24 (example).

Click Create Subnet.

Create Private Subnet:

Repeat the above step with:

Subnet name: MyPrivateSubnet.

CIDR block: 10.0.2.0/24.

Create an Internet Gateway (Required for public access):

Go to Internet Gateways > Create internet gateway.

Name it (e.g., MyIGW) and click Create internet gateway.

Select the newly created IGW and click Actions > Attach to VPC.

Attach it to MyProjectVPC.

Update Route Tables:

Navigate to Route Tables, find the route table associated with your Public Subnet.

Add a route:

Destination: 0.0.0.0/0.

Target: Your newly created Internet Gateway (MyIGW).

This ensures internet-bound traffic from the public subnet is routed to the Internet Gateway.

(Optional) For the private subnet, you could create a NAT Gateway if you want to allow outbound internet access from the private subnet. This is typically recommended, but 
not mandatory if you only need intranet connectivity.

2.3 Launch EC2 Instances

2.3.1 Public EC2 Instance

Navigate to EC2: Go to Services > EC2.

Launch Instance:

Click Launch instances.

Choose an AMI (e.g., Amazon Linux 2).

Choose an instance type (e.g., t2.micro).

Configure Instance Details:

Network: Select MyProjectVPC.

Subnet: Select MyPublicSubnet.

Auto-assign Public IP: Enable (to ensure the instance gets a public IP).

Add Storage: Accept defaults or configure as needed.

Configure Security Group:

Create a new security group (e.g., PublicSG).

Allow SSH (port 22) from your IP or a restricted range.

Allow HTTP (port 80) from 0.0.0.0/0 for public web access.

Review > Launch > select or create a key pair.

Click Launch Instances.

2.3.2 Private EC2 Instance

Launch another instance:

Same AMI and instance type.

Network: MyProjectVPC.

Subnet: MyPrivateSubnet.

Auto-assign Public IP: Disable.

Configure Security Group:

Create a new security group (e.g., PrivateSG).

Allow SSH (port 22) only from the PublicSG security group (i.e., reference the security group ID so that only the Public EC2 can SSH into the private EC2).
Proceed to launch.

Note: The private EC2 instance will not have direct public internet access unless you configure a NAT Gateway or other means. It is typically accessed via the public EC2 instance (bastion host) or via VPN.

# 3. Route 53 Configuration

3.1 Register a Domain (Optional if you already own a domain)

Navigate to Route 53: Go to Services > Route 53.

Registered Domains: Click Register Domain and follow the steps to register, e.g., myapp.com.

3.2 Create a Hosted Zone

Hosted Zones: Click Create Hosted Zone.

Domain Name: myapp.com.

Type: Public Hosted Zone.

Click Create.

Copy Name Servers:

In the hosted zone details, you’ll see Name Servers (NS records).

If you registered the domain in Route 53, these are automatically assigned.

If you registered elsewhere, update the domain’s DNS settings to use these Route 53 name servers.

3.3 Create DNS Records

Create an A Record:

Name: myapp.com.

Type: A (IPv4).

Value: Public IP of your public EC2 instance.

Routing Policy: Simple.

Click Create records.

Optional - Create a CNAME Record:

Name: www.myapp.com.

Value: myapp.com.

This redirects www.myapp.com to myapp.com.

# 4. VPC with AWS Transit Gateway (TGW)

If you have multiple VPCs or plan to connect an on-prem network, a Transit Gateway helps centralize network communication.

4.1 Create a Transit Gateway

Navigate to VPC: Go to Services > VPC > Transit Gateways.

Create Transit Gateway:

Name: MyTransitGateway.

Amazon side ASN: Accept default or provide a custom ASN (e.g., 64512).

Leave other defaults as is and click Create Transit Gateway.

4.2 Attach Your VPC to the TGW

Transit Gateway Attachments: Click Create Transit Gateway Attachment.

Resource type: VPC.

Transit Gateway ID: MyTransitGateway.

VPC ID: MyProjectVPC.

Subnets: Select at least one subnet from each Availability Zone you want to attach.

Click Create attachment.

4.3 Update VPC Route Tables

Route Tables: For each subnet you attached, go to its route table.

Add a route:

Destination: The CIDR of the networks you want to reach via TGW.

Target: MyTransitGateway.

Repeat for additional subnets or VPCs that you want to interconnect.

# 5. Deploy a (Slightly More Complex) Dummy Application on EC2
5.1 SSH into the Public EC2 Instance

Obtain Your Public IP: From the EC2 console, note the public IP or DNS of the instance in MyPublicSubnet.

SSH Command (Linux/macOS):

ssh -i /path/to/mykey.pem ec2-user@<PUBLIC-EC2-IP>

Update the System:

sudo yum update -y

5.2 Install a Web Server (Apache)

Install Apache:
sudo yum install httpd -y

Start and Enable Apache:

sudo systemctl start httpd
sudo systemctl enable httpd

5.3 Create a More Detailed HTML Page

Edit the index.html File:

sudo nano /var/www/html/index.html

Paste the Following Content:

<!DOCTYPE html>
<html>
<head>
    <meta charset="UTF-8">
    <title>My Enhanced Web App</title>
    <style>
        body {
            font-family: Arial, sans-serif;
            margin: 0;
            padding: 0;
            background-color: #f4f4f4;
        }
        header, footer {
            background-color: #333;
            color: #fff;
            padding: 1rem;
            text-align: center;
        }
        nav {
            background-color: #444;
            color: #fff;
            padding: 0.5rem;
            text-align: center;
        }
        nav a {
            color: #fff;
            margin: 0 1rem;
            text-decoration: none;
        }
        main {
            padding: 2rem;
            background-color: #fff;
            margin: 1rem auto;
            max-width: 600px;
        }
        h1 {
            color: #333;
        }
        p {
            line-height: 1.6;
        }
    </style>
</head>
<body>
    <header>
        <h1>Welcome to My Enhanced Web App</h1>
    </header>
    <nav>
        <a href="#">Home</a>
        <a href="#">About</a>
        <a href="#">Contact</a>
    </nav>
    <main>
        <h2>About This App</h2>
        <p>
            This is a more detailed example of a static web application hosted on an Amazon EC2 instance. 
            Here you can add text describing your project, features, or any relevant information.
        </p>
        <h2>Getting Started</h2>
        <p>
            1. Connect to Zscaler VPN if required. <br>
            2. Access this page through the public domain or IP. <br>
            3. Enjoy the content securely.
        </p>
        <h2>Security & Scaling</h2>
        <p>
            The application is secured behind an Application Load Balancer with SSL, 
            and it's deployed in a highly available environment. 
            Future enhancements include auto-scaling and continuous integration for seamless updates.
        </p>
    </main>
    <footer>
        <p>© 2025 My Enhanced Web App - All Rights Reserved</p>
    </footer>
</body>
</html>

Save and Exit (Ctrl + O, Enter, Ctrl + X for nano).

5.4 Test the Application

Open a Browser: Go to http://<PUBLIC-EC2-IP> or http://myapp.com if DNS is configured.

You should see the enhanced HTML page.

6. VPN Authentication in the Private Subnet

6.1 Restrict Access to the Private EC2

The PrivateSG security group should only allow SSH connections from the PublicSG or from the Zscaler VPN IP range (if you have a direct route).

6.2 Connect via Public EC2 (Bastion Host) or Zscaler VPN

Via Bastion (Public EC2):

SSH into your public EC2.

From there, SSH into the private EC2:

ssh ec2-user@<PRIVATE-EC2-LOCAL-IP>


Via OpenVPN:

Make sure your OpenVPN is connected and set to route traffic to the VPC IP ranges (10.0.2.0/24).

SSH directly to the private IP if your VPN tunnel is correctly configured.

# 7. Load Balancer with SSL Certificate

7.1 Create an Application Load Balancer (ALB)

Navigate to EC2: In the AWS Console, go to EC2 > Load Balancers.

Create Load Balancer:

Select Application Load Balancer.

Name: MyAppALB.

Scheme: Internet-facing (for public access).

IP address type: IPv4.

Listeners: Add HTTP (80) and later add HTTPS (443).

VPC: Select MyProjectVPC.

Subnets: Select two or more public subnets in different Availability Zones (e.g., MyPublicSubnet in two different AZs if you created them).

7.2 Configure Security Groups and Target Group

Security Group for ALB:

Allows inbound HTTP (80) and HTTPS (443) from the public internet.

Outbound to the EC2 instances on port 80 if needed (or whichever port your app uses).

Create a Target Group:

Target Type: Instances.

Protocol: HTTP.

Port: 80.

VPC: MyProjectVPC.

After creation, register your public EC2 instance(s) as targets (or your private ones if you want them behind an ALB).

7.3 Request and Attach an SSL Certificate (HTTPS)

Request a Certificate in AWS Certificate Manager (ACM):

Go to ACM > Request a certificate > Request a public certificate.

Enter your domain (e.g., myapp.com) and any subdomains you need (e.g., www.myapp.com).

Complete DNS or email validation as required.

Configure HTTPS Listener:

In the ALB Listeners tab, add an HTTPS (443) listener.

Default action: Forward to your target group.

SSL Certificate: Select the certificate you requested from ACM.

7.4 Update DNS

In Route 53, change the A record to be an Alias to your ALB’s DNS name, so myapp.com points to the ALB rather than the EC2 instance.

# 8. Create AWS Transit Gateway (If Needed Again)

This is essentially the same process described in Step 4. If you want to connect additional VPCs or on-prem networks:

Create or Use Existing TGW: MyTransitGateway.

Attach Additional VPCs: Follow the steps in Section 4.

Configure Route Tables: Update each VPC’s route table to direct traffic to/from the transit gateway.

# 9. AWS CodePipeline (Continuous Integration & Deployment)

9.1 Set Up Your Source Control (GitHub or CodeCommit)

Create a GitHub Repository: e.g., aws-secure-web-app.

Clone Locally:

git clone https://github.com/<YourUsername>/aws-secure-web-app.git

Add Your Application Code:

Place your index.html, any scripts, or infrastructure-as-code templates (like CloudFormation templates) in the repo.

Commit and Push:

git add .
git commit -m "Initial commit with web app code"
git push origin main

9.2 Create a CodePipeline

Navigate to CodePipeline: Go to Services > CodePipeline.

Create Pipeline:

Pipeline name: MyApp-Pipeline.

Service Role: Create a new or use an existing role.

Click Next.

9.3 Add Source Stage

Source Provider: Choose GitHub (Version 2) or AWS CodeCommit.

Connect your GitHub account if prompted.

Select Repository: aws-secure-web-app.

Select Branch: main.

Output Artifact: SourceArtifact.

Click Next.

9.4 Add Build Stage (AWS CodeBuild)

Build Provider: AWS CodeBuild.

Region: Same region as your pipeline.

Create a New Build Project:

Project name: MyApp-CodeBuild.

Environment: Choose managed image (e.g., Amazon Linux 2).

Buildspec file: If you have a buildspec.yml in your repo, reference it. Example:

version: 0.2
phases:
  install:
    commands:
      - echo "Installing dependencies..."
  build:
    commands:
      - echo "Building the application..."
  post_build:
    commands:
      - echo "Build complete"
artifacts:
  files:
    - index.html
    - '**/*'
Output Artifact: BuildArtifact.

9.5 Add Deploy Stage

You have multiple options for deployment:

AWS CodeDeploy (if your app runs on EC2 or ECS).

S3-based deployment (if it’s a static site).

Manual scripts to copy files to the EC2 instance.

Example Using CodeDeploy to EC2:

Deploy Provider: AWS CodeDeploy.

Application Name: Create a new or select existing CodeDeploy application.

Deployment Group: A group containing your EC2 instance(s).

AppSpec File: In your repo, define an appspec.yml that tells CodeDeploy how to deploy to your instance. Example:

version: 0.0
os: linux
files:
  - source: /
    destination: /var/www/html/
hooks:
  AfterInstall:
    - location: scripts/restart_httpd.sh
      timeout: 300
      runas: root
      
Scripts: You can have a scripts/restart_httpd.sh that restarts Apache:

#!/bin/bash
systemctl restart httpd

Permissions: Ensure the CodeDeploy service role and the IAM role on the EC2 instances allow the necessary actions (e.g., s3:GetObject, codedeploy:*).

9.6 Review and Create Pipeline

Create pipeline and watch it run. On subsequent code pushes, CodePipeline will automatically detect changes, trigger a new build, and deploy your updated files.


# 10. (Optional) Further Hardening and Monitoring

AWS WAF: Add a Web Application Firewall for additional security on your ALB.

CloudWatch Logs and Metrics: Monitor your instance logs and set alarms for CPU usage, memory, etc.

Amazon GuardDuty: Intelligent threat detection to protect your AWS accounts and workloads.

Conclusion

By following the expanded steps above, you have:

OpenVPN set up for secure remote access.

VPC with Public and Private Subnets, properly routed through an Internet Gateway for public access, and optionally a NAT Gateway for private outbound traffic.

EC2 Instances in both public (web-accessible) and private (restricted) subnets.

Route 53 configured for user-friendly domain names.

An optional Transit Gateway for interconnecting multiple VPCs or on-premises networks.

A slightly more complex dummy application running on Apache.

An Application Load Balancer with SSL for secure HTTPS traffic.

AWS CodePipeline for continuous integration and deployment, automating the build and deploy process whenever changes are pushed to your repository.
With these components in place, you have a scalable, secure, and automated environment. Continue to refine your infrastructure with best practices like Infrastructure as Code (CloudFormation/Terraform), advanced load balancing rules, autoscaling groups, and deeper security monitoring for a production-grade setup.

Happy Coding!

