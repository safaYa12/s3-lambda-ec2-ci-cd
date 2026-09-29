# 🐍 Automated Neon Snake Game Deployment Pipeline on AWS

An automated CI/CD-style deployment pipeline for a web-based Neon Snake Game using **Amazon S3, AWS Lambda, Amazon EC2, AWS Systems Manager (SSM), IAM, and Nginx**.

The project demonstrates how AWS managed services can be combined to create a lightweight, event-driven deployment system without requiring SSH access or a traditional CI/CD server.

---

## 📌 Project Overview

The goal of this project is to automatically deploy a new version of a web application whenever a new application package is uploaded to an Amazon S3 bucket.

Instead of manually connecting to the EC2 server, downloading the application, extracting the files, and restarting the web server, the entire process is automated.

### Deployment flow

```text
Developer
    │
    │ Upload game.zip
    ▼
┌─────────────────────┐
│    Amazon S3        │
│  Artifact Bucket    │
└──────────┬──────────┘
           │
           │ ObjectCreated Event
           ▼
┌─────────────────────┐
│   AWS Lambda        │
│   SnakeDeployer     │
└──────────┬──────────┘
           │
           │ SSM SendCommand
           ▼
┌─────────────────────┐
│ AWS Systems Manager │
│     Run Command     │
└──────────┬──────────┘
           │
           ▼
┌─────────────────────┐
│ Amazon EC2          │
│ NeonSnakeServer     │
│                     │
│ Amazon Linux 2023   │
│ Nginx               │
└──────────┬──────────┘
           │
           │ Serve application
           ▼
      🌐 Web Browser
```

---

# 🎯 Project Objectives

This project was designed to demonstrate:

* Event-driven application deployment
* Amazon S3 event notifications
* AWS Lambda automation
* AWS Systems Manager Run Command
* IAM role separation
* EC2 application hosting
* Nginx web server configuration
* Automated artifact deployment
* SSH-free server management
* Basic CI/CD architecture using native AWS services

---

# 🛠️ AWS Services Used

| AWS Service             | Purpose                                       |
| ----------------------- | --------------------------------------------- |
| **Amazon S3**           | Stores application deployment artifacts       |
| **AWS Lambda**          | Processes S3 events and initiates deployments |
| **Amazon EC2**          | Hosts the Neon Snake Game                     |
| **AWS Systems Manager** | Executes deployment commands remotely         |
| **AWS IAM**             | Controls permissions between AWS services     |
| **Nginx**               | Serves the deployed web application           |

---

# 🏗️ Architecture

The deployment architecture is event-driven.

When a developer uploads `game.zip` to the artifact bucket:

1. S3 detects the new object.
2. S3 triggers the `SnakeDeployer` Lambda function.
3. Lambda extracts the bucket and object information from the event.
4. Lambda sends an SSM Run Command to the EC2 server.
5. The EC2 instance downloads the artifact from S3.
6. The existing website files are removed.
7. The new application files are extracted.
8. The files are moved into the Nginx document root.
9. Nginx is restarted.
10. The updated application becomes available through the EC2 public IP.

---

# 🔐 IAM Configuration

Two separate IAM roles were created to separate the permissions required by the EC2 server and Lambda function.

## 1. EC2 Server Role

### Role

```text
iam_role_snake_server
```

### Attached permissions

```text
AmazonSSMManagedInstanceCore
AmazonS3ReadOnlyAccess
```

### Purpose

The EC2 instance uses this role to:

* Register with AWS Systems Manager
* Receive SSM Run Command instructions
* Communicate with Systems Manager
* Read application artifacts from S3

The EC2 server does not require write access to the S3 bucket.

---

# 2. Lambda Deployer Role

### Role

```text
iam_role_snake_deployer
```

### Permissions used in the project

```text
AmazonSSMFullAccess
AmazonS3ReadOnlyAccess
AWSLambdaBasicExecutionRole
```

### Purpose

The Lambda function requires permissions to:

* Send commands through AWS Systems Manager
* Read relevant S3 resources
* Write execution logs to CloudWatch Logs

> **Security note:** For a production implementation, broad managed policies such as `AmazonSSMFullAccess` and `AmazonS3ReadOnlyAccess` should be replaced with narrowly scoped custom IAM policies following the principle of least privilege.

---

# 🪣 S3 Artifact Storage

An S3 bucket was created specifically for deployment artifacts.

Example:

```text
neon-snake-artifacts-10101
```

The bucket name follows the required naming convention:

```text
neon-snake-artifacts-<unique-value>
```

### Region

```text
us-east-1
```

The bucket stores the application package:

```text
game.zip
```

---

# 🖥️ EC2 Web Server

The application server was created using:

### Instance name

```text
NeonSnakeServer
```

### Operating system

```text
Amazon Linux 2023
```

### Instance type

```text
t3.micro
```

### SSH

No key pair was required for administration because Systems Manager is used instead of SSH.

### IAM instance profile

```text
iam_role_snake_server
```

### Network

HTTP traffic was allowed from the Internet so that the game could be accessed through the EC2 public IPv4 address.

---

# ⚙️ EC2 User Data

The following bootstrap script was used during instance creation:

```bash
#!/bin/bash

dnf update -y
dnf install -y nginx unzip

systemctl start nginx
systemctl enable nginx

chown -R ec2-user:ec2-user /usr/share/nginx/html
chmod -R 755 /usr/share/nginx/html
```

This automatically:

* Updates the operating system
* Installs Nginx
* Installs `unzip`
* Starts Nginx
* Enables Nginx at boot
* Configures permissions for the web directory

---

# ☁️ AWS Systems Manager

Systems Manager is used as the deployment mechanism instead of SSH.

The EC2 instance was successfully registered as an SSM managed node.

Verification:

```bash
aws ssm describe-instance-information --region us-east-1
```

The instance returned:

```text
InstanceId: i-0791534196f22bf9c
PingStatus: Online
PlatformName: Amazon Linux
PlatformVersion: 2023
AgentVersion: 3.3.5226.0
```

The SSM Agent was also verified directly on the EC2 instance:

```bash
sudo systemctl status amazon-ssm-agent
```

Result:

```text
Active: active (running)
```

This confirmed that the EC2 instance was online and ready to receive SSM commands.

---

# ⚡ Lambda Deployment Function

A Lambda function named:

```text
SnakeDeployer
```

was created using Python.

The function receives the S3 event and extracts:

* Bucket name
* Object key

It then constructs an SSM deployment command.

## Lambda Code

```python
import boto3
import urllib.parse

ssm = boto3.client("ssm", region_name="us-east-1")

def lambda_handler(event, context):

    bucket = event["Records"][0]["s3"]["bucket"]["name"]

    key = urllib.parse.unquote_plus(
        event["Records"][0]["s3"]["object"]["key"]
    )

    commands = [
        f"aws s3 cp s3://{bucket}/{key} /tmp/game.zip",
        "sudo rm -rf /usr/share/nginx/html/*",
        "sudo mkdir -p /tmp/deploy_temp",
        "sudo unzip -o /tmp/game.zip -d /tmp/deploy_temp/",
        "sudo mv /tmp/deploy_temp/* /usr/share/nginx/html/",
        "sudo rm -rf /tmp/deploy_temp /tmp/game.zip",
        "sudo systemctl restart nginx"
    ]

    ssm.send_command(
        Targets=[
            {
                "Key": "tag:Name",
                "Values": ["NeonSnakeServer"]
            }
        ],
        DocumentName="AWS-RunShellScript",
        Parameters={
            "commands": commands
        }
    )

    return {
        "status": "Snake Deployed successfully"
    }
```

---

# 🔄 Deployment Process

The deployment process is completely event-driven.

## Step 1 — Developer uploads application

The developer uploads:

```text
game.zip
```

to:

```text
s3://neon-snake-artifacts-<UNIQUE>/
```

Example:

```bash
aws s3 cp game.zip s3://neon-snake-artifacts-10101/
```

---

## Step 2 — S3 generates an event

The S3 bucket is configured with an event notification for:

```text
All object create events
```

The event triggers:

```text
SnakeDeployer
```

---

## Step 3 — Lambda processes the event

Lambda extracts the S3 information:

```python
bucket = event["Records"][0]["s3"]["bucket"]["name"]
```

and:

```python
key = urllib.parse.unquote_plus(
    event["Records"][0]["s3"]["object"]["key"]
)
```

For example:

```text
Bucket:
neon-snake-artifacts-10101

Object:
game.zip
```

---

## Step 4 — Lambda sends an SSM command

Lambda sends the deployment instructions to the EC2 instance using:

```text
AWS-RunShellScript
```

The EC2 target is identified by its Name tag:

```text
NeonSnakeServer
```

This avoids hard-coding the instance ID in the Lambda function.

---

# 📦 Deployment Commands

The deployment commands executed on the EC2 server are:

```bash
aws s3 cp s3://<bucket>/<key> /tmp/game.zip
```

Download the newly uploaded artifact.

```bash
sudo rm -rf /usr/share/nginx/html/*
```

Remove the previous application version.

```bash
sudo mkdir -p /tmp/deploy_temp
```

Create a temporary extraction directory.

```bash
sudo unzip -o /tmp/game.zip -d /tmp/deploy_temp/
```

Extract the application.

```bash
sudo mv /tmp/deploy_temp/* /usr/share/nginx/html/
```

Move the new application into the Nginx document root.

```bash
sudo rm -rf /tmp/deploy_temp /tmp/game.zip
```

Clean up temporary files.

```bash
sudo systemctl restart nginx
```

Restart Nginx and serve the newly deployed application.

---

# 🧪 Testing the Deployment

After configuring the pipeline, the deployment was tested by uploading the application artifact.

```bash
aws s3 cp game.zip s3://neon-snake-artifacts-<UNIQUE>/
```

The upload triggered the following chain:

```text
S3 Upload
   ↓
S3 Event
   ↓
Lambda
   ↓
SSM SendCommand
   ↓
EC2
   ↓
Download game.zip
   ↓
Extract files
   ↓
Update Nginx
   ↓
Restart Nginx
```

---

# 🔎 Verifying SSM

The SSM command can be checked through:

```text
AWS Systems Manager
        ↓
Run Command
        ↓
Command history
```

The deployment command should eventually show:

```text
Status: Success
```

It can also be tested through the AWS CLI.

Example:

```bash
aws ssm send-command \
    --instance-ids i-0791534196f22bf9c \
    --document-name "AWS-RunShellScript" \
    --parameters 'commands=["hostname","uptime"]' \
    --region us-east-1
```

A successful test returned a command ID:

```text
7e2b89b6-c4e0-438e-bd94-cdda22b8b9ac
```

The command can then be inspected using:

```bash
aws ssm get-command-invocation \
    --command-id 7e2b89b6-c4e0-438e-bd94-cdda22b8b9ac \
    --instance-id i-0791534196f22bf9c \
    --region us-east-1
```

---

# 🐛 Troubleshooting

During development, the Lambda initially returned:

```text
InvalidInstanceId:
Instances not in a valid state for account
```

The EC2 instance itself was investigated.

### SSM registration

```text
InstanceId: i-0791534196f22bf9c
PingStatus: Online
```

### SSM Agent

```text
amazon-ssm-agent.service
Active: active (running)
```

### Direct SSM test

A direct `send-command` from the AWS CLI successfully targeted the instance:

```text
CommandId:
7e2b89b6-c4e0-438e-bd94-cdda22b8b9ac
```

This established that:

```text
EC2 → SSM
```

was functioning correctly.

The investigation therefore focused on the Lambda environment, particularly:

* Lambda Region
* AWS account
* Lambda execution role
* `ssm:SendCommand` permissions

The Lambda SSM client was explicitly configured for the correct Region:

```python
ssm = boto3.client("ssm", region_name="us-east-1")
```

This is an important lesson when working with regional AWS services: **the Lambda execution environment and the target EC2/SSM resources must be addressed in the correct AWS Region and account.**

---

# 🌐 Accessing the Game

Once the deployment completed successfully:

1. Open the EC2 console.
2. Select `NeonSnakeServer`.
3. Copy the instance's public IPv4 address.
4. Open the address in a browser.

Example:

```text
http://<EC2-PUBLIC-IP>
```

Nginx serves the newly deployed Neon Snake Game.

---

# 🔐 Security Considerations

This project demonstrates several useful security concepts.

## No SSH dependency

The deployment process does not require:

```text
SSH
Port 22
SSH private keys
Manual server access
```

Systems Manager provides the remote command mechanism.

---

## Separate IAM roles

The architecture separates permissions between:

```text
Lambda Deployer
        │
        └── Sends deployment commands

EC2 Server
        │
        └── Receives commands + reads artifacts
```

This is preferable to using one shared role for all components.

---

## Least privilege

The lab uses AWS managed policies for simplicity.

For a production environment, permissions should be narrowed.

For example, Lambda should ideally only be able to:

```text
ssm:SendCommand
```

against the required SSM document and target resources.

Similarly, the EC2 instance should ideally have read access only to the specific S3 bucket/prefix containing deployment artifacts rather than unrestricted S3 read access.

---

# ⚠️ Production Improvements

This project is intentionally lightweight and educational. A production deployment pipeline would require additional controls.

Potential improvements include:

### 1. Restrict S3 permissions

Instead of:

```text
AmazonS3ReadOnlyAccess
```

create a policy allowing access only to:

```text
arn:aws:s3:::neon-snake-artifacts-<UNIQUE>/*
```

---

### 2. Restrict Lambda SSM permissions

Replace:

```text
AmazonSSMFullAccess
```

with a narrowly scoped policy allowing only the required SSM actions.

---

### 3. Artifact validation

Before deployment:

* Validate the ZIP structure
* Verify the artifact source
* Validate checksums
* Reject unexpected files

---

### 4. Safer deployment strategy

The current deployment removes the existing website before installing the new version:

```bash
rm -rf /usr/share/nginx/html/*
```

A production system could instead use:

```text
Versioned deployment directories
        ↓
Validation
        ↓
Atomic symlink switch
        ↓
Rollback if required
```

This prevents downtime if a deployment fails halfway through.

---

### 5. CloudWatch monitoring

Lambda logs should be monitored through CloudWatch.

Additional monitoring could include:

* SSM command failures
* Lambda failures
* S3 upload events
* Deployment duration
* Nginx health
* EC2 status

---

### 6. HTTPS

The current demonstration uses HTTP.

A production deployment should use:

```text
HTTPS
```

with a valid TLS certificate, potentially using:

```text
Application Load Balancer
        +
AWS Certificate Manager
```

---

### 7. Versioned artifacts

Instead of always deploying:

```text
game.zip
```

the pipeline could use versioned artifacts:

```text
game-v1.0.0.zip
game-v1.0.1.zip
game-v1.0.2.zip
```

This makes rollback and auditing easier.

---

# 🧠 What I Learned

This project provided hands-on experience with several AWS concepts.

### Amazon S3

Learned how object storage can act as an artifact repository and how object creation events can initiate automation.

### AWS Lambda

Learned how Lambda can act as serverless orchestration logic between AWS services.

### AWS Systems Manager

Learned how SSM Run Command can replace traditional SSH-based server administration.

### IAM

Learned how different AWS resources require different IAM roles and permissions.

### Amazon EC2

Learned how to configure an Amazon Linux server and prepare it to host a web application.

### Nginx

Learned how to install and configure Nginx as a web server and update its document root automatically.

### Event-driven architecture

The project demonstrated how an application deployment can be triggered by an event rather than a manually executed process.

---

# 🚀 Final Architecture

The completed project can be summarized as:

```text
                 Developer
                     │
                     │
                     │ Upload game.zip
                     ▼
             ┌────────────────┐
             │   Amazon S3    │
             │ Artifact Store │
             └───────┬────────┘
                     │
                     │ ObjectCreated
                     ▼
             ┌────────────────┐
             │ AWS Lambda     │
             │ SnakeDeployer  │
             └───────┬────────┘
                     │
                     │ SSM SendCommand
                     ▼
             ┌────────────────┐
             │ AWS Systems    │
             │ Manager (SSM)  │
             └───────┬────────┘
                     │
                     │ Run Shell Script
                     ▼
        ┌──────────────────────────┐
        │       EC2 Server         │
        │     NeonSnakeServer      │
        │                          │
        │   Amazon Linux 2023      │
        │   SSM Agent              │
        │   Nginx                  │
        │                          │
        │   /usr/share/nginx/html │
        └────────────┬─────────────┘
                     │
                     │ HTTP
                     ▼
                Web Browser
                     │
                     ▼
                🐍 Neon Snake
```

---

# 📁 Suggested GitHub Repository Structure

A clean repository structure for this project would be:

```text
automated-neon-snake-deployment/
│
├── README.md
│
├── lambda/
│   └── snake_deployer.py
│
├── ec2/
│   └── user-data.sh
│
├── iam/
│   ├── lambda-policy.json
│   └── ec2-policy.json
│
├── deployment/
│   └── deploy-commands.sh
│
├── screenshots/
│   ├── s3-bucket.png
│   ├── ec2-instance.png
│   ├── iam-roles.png
│   ├── lambda-function.png
│   ├── ssm-command-success.png
│   └── snake-game.png
│
└── game/
    └── game.zip
```

> **Recommendation:** Don't commit AWS access keys, secret keys, private SSH keys, or other credentials to the repository. Also consider whether the game ZIP itself should be committed depending on its licensing and size.

---

# 🏁 Conclusion

This project successfully demonstrates a lightweight, event-driven deployment pipeline using native AWS services.

A developer only needs to upload a new application artifact to S3. The rest of the deployment process happens automatically:

```text
Upload
  ↓
S3 Event
  ↓
Lambda
  ↓
SSM
  ↓
EC2
  ↓
Nginx
  ↓
Updated Application
```

The architecture eliminates the need for manual SSH-based deployment and demonstrates how AWS serverless services can be combined with EC2 to create an automated deployment workflow.

## Key Technologies

```text
AWS S3
AWS Lambda
AWS Systems Manager
Amazon EC2
AWS IAM
Nginx
Amazon Linux 2023
Python
Bash
```

**Project:** Automated Neon Snake Game Deployment Pipeline
**Environment:** AWS `us-east-1`
**Architecture:** Event-driven / serverless-assisted CI/CD
**Deployment method:** S3 → Lambda → SSM → EC2 → Nginx
