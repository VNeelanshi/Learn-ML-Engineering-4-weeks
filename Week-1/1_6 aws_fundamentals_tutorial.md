# AWS Fundamentals for ML Engineers
## IAM · EC2 & Networking · S3 · ECR & ECS · Deployment Practice · CLI & boto3

> **Level:** Beginner-friendly (no prior AWS experience assumed)
> **Goal:** By the end you'll have your ML serving container running on ECS, with proper IAM security, S3 storage, and the ability to manage everything from both the console and Python.

---

## Setup Checklist

Before you start:

```bash
# Install the AWS CLI (version 2)
# macOS
brew install awscli

# Ubuntu/Debian
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip && sudo ./aws/install

# Verify
aws --version   # should show aws-cli/2.x.x

# Install boto3 (for Hour 6)
pip install boto3
```

You'll also need:
- A **credit/debit card** for AWS account creation (the free tier covers everything in this tutorial)
- The **Docker image** from the CI/CD tutorial (or any container you want to deploy)
- A terminal and a text editor

### A Note on Costs

Everything in this tutorial fits within the **AWS Free Tier** (available for 12 months after account creation). That means:
- 750 hours/month of `t2.micro` EC2 instances
- 5 GB of S3 storage
- 500 MB/month of ECR storage

If you forget to clean up resources after the exercises, you *will* get charged. Each exercise section includes a cleanup step — don't skip it.

---

---

# Hour 1 — AWS Account & IAM

## Why This Matters

Everything in AWS starts with identity: *who are you, and what are you allowed to do?* Get this wrong, and you either lock yourself out of your own resources or leave the door wide open for anyone. IAM (Identity and Access Management) is the first thing you configure — before you launch a single server or create a single bucket.

---

## 1.1 Creating Your AWS Account

Go to [aws.amazon.com](https://aws.amazon.com) and click **Create an AWS Account**. The account you create is the **root account** — it has unrestricted access to everything. Think of it as the master key to a building: you use it to set things up, then put it in a safe and use regular keys day-to-day.

After creating the account:

1. **Enable MFA (Multi-Factor Authentication)** on the root account immediately:
   - Console → IAM → Security credentials → Assign MFA device
   - Use an authenticator app (Google Authenticator, Authy, etc.)
2. **Do not use the root account for daily work** — create an IAM user instead (next section)

> ⚠️ **Gotcha:** If your root account gets compromised and doesn't have MFA, an attacker can spin up hundreds of expensive GPU instances in minutes. AWS bills are *your* responsibility. Enable MFA before doing anything else.

---

## 1.2 IAM — The Core Concepts

IAM has four building blocks:

| Concept | What It Is | Real-World Analogy |
|---|---|---|
| **User** | A person or service that needs access | An employee with a badge |
| **Group** | A collection of users who share permissions | A department (all engineers get the same access) |
| **Role** | A set of permissions that can be *assumed* temporarily | A contractor pass — you pick it up when needed |
| **Policy** | A JSON document that defines what's allowed or denied | The rules printed on the badge |

The relationship: policies are *attached to* users, groups, or roles.

---

## 1.3 Creating Your First IAM User

The root account should only be used for billing and account-level settings. For everything else, create an IAM user:

1. Console → **IAM** → **Users** → **Create user**
2. Username: `your-name-admin`
3. Check **Provide user access to the AWS Management Console**
4. Set a password
5. Click **Next: Permissions**
6. Choose **Attach policies directly** → search for and select `AdministratorAccess`
7. Click **Create user**

Now log out of the root account and log in as this IAM user. This is your daily-driver account.

> ⚠️ **Gotcha:** `AdministratorAccess` gives full permissions — it's fine for a personal learning account, but in a team setting you'd never give this to everyone. We'll cover least-privilege policies next.

---

## 1.4 Least-Privilege Policies

The principle of least privilege: give every user or service *only* the permissions they need, nothing more.

AWS policies are JSON documents. Here's one that allows reading S3 buckets but nothing else:

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "s3:GetObject",
                "s3:ListBucket"
            ],
            "Resource": [
                "arn:aws:s3:::my-ml-bucket",
                "arn:aws:s3:::my-ml-bucket/*"
            ]
        }
    ]
}
```

Breaking this down:

- **Version** — always `"2012-10-17"` (this is the policy language version, not your version)
- **Effect** — `Allow` or `Deny`
- **Action** — the specific API calls permitted (e.g. `s3:GetObject` = download a file)
- **Resource** — which specific resources this applies to (using ARNs — Amazon Resource Names)

The two `Resource` lines are important: the first (`my-ml-bucket`) covers bucket-level operations like `ListBucket`, and the second (`my-ml-bucket/*`) covers object-level operations like `GetObject` on files inside the bucket.

> ⚠️ **Gotcha:** `"Resource": "*"` means "every resource in the account." This is the opposite of least-privilege. Always scope to specific resources. Even in a learning environment, practicing this habit early prevents dangerous muscle memory.

---

## 1.5 IAM Roles — Permissions Without Passwords

Users have long-lived credentials (username + password, or access keys). Roles are different — they provide *temporary* credentials that expire automatically.

You'll use roles for:
- **EC2 instances** that need to access S3 (the instance "assumes" a role)
- **ECS tasks** that need to pull images or write logs
- **CI/CD pipelines** that deploy to AWS

The key idea: instead of putting AWS access keys into your code or environment variables (dangerous — they can leak), you assign a role to the service, and AWS handles the credentials automatically.

```
Without a role:    EC2 instance → hardcoded access key → S3
With a role:       EC2 instance → assumes role → temporary creds → S3
                                  (managed by AWS, rotated automatically)
```

---

## ✏️ Hour 1 Exercise

**Goal:** Secure your AWS account and create a least-privilege user.

1. Create your AWS account (if you haven't already).
2. Enable MFA on the root account.
3. Create an IAM user named `your-name-admin` with `AdministratorAccess`. Log in as this user.
4. Create a second IAM user named `s3-reader` with a custom policy that only allows `s3:GetObject` and `s3:ListBucket` on a bucket called `my-test-bucket-YOUR-UNIQUE-ID` (bucket names are globally unique — append random characters).
5. In IAM → **Access keys**, generate access keys for `s3-reader`. Save them somewhere safe — you'll use them in Hour 6.

**Stretch:** Create an IAM group called `ml-engineers`, attach a policy that allows EC2 and S3 access, and add your admin user to it. Then remove the direct `AdministratorAccess` from the user — their permissions should now come from the group.

---

---

# Hour 2 — EC2 & Networking

## Why This Matters

EC2 (Elastic Compute Cloud) is a virtual machine you rent by the hour. It's the foundation of most AWS deployments — and the first place you'll run your ML model if you're not using containers. But a server is useless if it's not networked correctly. This hour teaches you both: how to launch a machine, and how to make sure the right traffic reaches it while everything else is blocked.

---

## 2.1 What an EC2 Instance Actually Is

When you "launch an EC2 instance," you're renting a virtual machine with a specific combination of CPU, memory, storage, and network capacity. AWS offers dozens of instance types organized by family:

| Family | Optimized For | Example | ML Use Case |
|---|---|---|---|
| `t2`, `t3` | General purpose (burstable) | `t2.micro` | Dev, testing, light API serving |
| `c5`, `c6` | Compute (high CPU) | `c5.xlarge` | CPU-heavy inference |
| `r5`, `r6` | Memory | `r5.large` | Large models loaded into RAM |
| `g4`, `p3` | GPU | `g4dn.xlarge` | GPU inference, training |

For this tutorial, `t2.micro` is free-tier eligible and sufficient.

---

## 2.2 VPCs — Your Private Network

A **VPC** (Virtual Private Cloud) is your own isolated network inside AWS. Every AWS account comes with a **default VPC** in each region — it works out of the box, which is why beginners can launch instances without thinking about networking. But understanding the structure matters as soon as you need to control access.

A VPC contains:

```
VPC (10.0.0.0/16)              ← your private network (65,536 IPs)
├── Public subnet (10.0.1.0/24)   ← connected to the internet via an Internet Gateway
├── Private subnet (10.0.2.0/24)  ← no direct internet access
└── Internet Gateway               ← the bridge between your VPC and the internet
```

- **Public subnet** — instances here can have a public IP and be reached from the internet. Your API server goes here.
- **Private subnet** — instances here can only be reached from within the VPC. Your database goes here.

For this tutorial, the default VPC works fine. You'll customize VPCs when you need network isolation (e.g., separating staging from production).

---

## 2.3 Security Groups — Your Firewall

A **security group** is a set of rules that controls what traffic can reach your instance (inbound) and what traffic it can send (outbound).

Think of it as a bouncer at the door:

- **Inbound rules** — who's allowed *in*? (e.g., "allow SSH from my IP only")
- **Outbound rules** — who can the instance talk *to*? (default: everything)

```
Inbound rules for a typical API server:
┌──────────┬────────┬─────────────┐
│ Protocol │  Port  │   Source    │
├──────────┼────────┼─────────────┤
│ SSH      │   22   │ My IP only  │
│ HTTP     │   80   │ Anywhere    │
│ HTTPS    │  443   │ Anywhere    │
│ Custom   │  8000  │ Anywhere    │  ← for uvicorn
└──────────┴────────┴─────────────┘
```

> ⚠️ **Gotcha:** The default security group allows all *outbound* traffic but **no inbound** traffic. If you launch an instance and can't SSH into it, the first thing to check is whether port 22 is open in the security group — it almost certainly isn't.

---

## 2.4 Key Pairs — How You Log In

AWS doesn't use passwords for SSH. Instead, you create a **key pair**: AWS stores the public key, you download the private key (a `.pem` file), and you use it to SSH in.

```bash
# When you create a key pair in the console, you download my-key.pem
# Set the right permissions (SSH refuses to use a key with open permissions)
chmod 400 my-key.pem

# Connect to your instance
ssh -i my-key.pem ec2-user@<public-ip>
```

> ⚠️ **Gotcha:** If you lose the `.pem` file, you cannot SSH into the instance. AWS does not store the private key — there is no "forgot password" option. Keep the file safe and backed up.

---

## 2.5 Launching an Instance Step by Step

1. Console → **EC2** → **Launch instance**
2. **Name:** `ml-api-test`
3. **AMI (Amazon Machine Image):** Amazon Linux 2023 (free-tier eligible)
4. **Instance type:** `t2.micro`
5. **Key pair:** Create a new one (download the `.pem` file)
6. **Network settings:** Click Edit
   - VPC: default
   - Auto-assign public IP: **Enable**
   - Security group: Create new → add rules for SSH (port 22, your IP) and Custom TCP (port 8000, anywhere)
7. **Launch instance**

Wait 1–2 minutes for the instance to start, then:

```bash
ssh -i my-key.pem ec2-user@<public-ip-from-console>

# Once inside:
sudo yum update -y
sudo yum install python3-pip -y
pip3 install fastapi uvicorn
```

> ⚠️ **Gotcha:** The public IP changes every time you stop and start the instance. If you need a permanent IP, allocate an **Elastic IP** (free while attached to a running instance, charged if unused — another cleanup item to watch for).

---

## ✏️ Hour 2 Exercise

**Goal:** Launch an EC2 instance and run your API on it.

1. Launch a `t2.micro` instance with the settings above.
2. SSH in and install your dependencies.
3. Copy your `main.py` to the instance (using `scp`):
   ```bash
   scp -i my-key.pem main.py ec2-user@<public-ip>:~/
   ```
4. Start your API: `uvicorn main:app --host 0.0.0.0 --port 8000`
5. Open `http://<public-ip>:8000/docs` in your browser — you should see the Swagger UI.
6. **Clean up:** Stop or terminate the instance when done (Console → EC2 → Instances → right-click → Terminate). Verify it's gone.

**Stretch:** Create a *second* security group that only allows inbound traffic on port 8000 from your IP (not "anywhere"). Attach it to your instance and verify that the API is still reachable from your machine but would be blocked from other IPs.

---

---

# Hour 3 — S3

## Why This Matters

S3 (Simple Storage Service) is where you store everything that isn't running code: model files, training datasets, logs, configuration, artifacts. It's the most-used AWS service, period. For ML workflows, S3 is typically the glue — your training pipeline writes model files to S3, your serving API reads them from S3, and your CI/CD pipeline stores artifacts there.

---

## 3.1 Buckets and Objects

S3 has two concepts:

- **Bucket** — a container (like a top-level folder). Bucket names are *globally unique across all AWS accounts*.
- **Object** — a file inside a bucket. Identified by a **key** (the full path, like `models/v2/model.pkl`).

There are no real "folders" in S3 — `models/v2/model.pkl` is a single flat key that happens to contain slashes. The console shows a folder-like view, but under the hood it's all flat.

```bash
# Upload a file
aws s3 cp model.pkl s3://my-ml-bucket/models/v1/model.pkl

# Download it
aws s3 cp s3://my-ml-bucket/models/v1/model.pkl ./local-model.pkl

# List bucket contents
aws s3 ls s3://my-ml-bucket/models/

# Sync a directory
aws s3 sync ./local-models/ s3://my-ml-bucket/models/
```

> ⚠️ **Gotcha:** Bucket names are globally unique and permanent. `my-bucket` might already be taken by someone else in another AWS account. Use a unique prefix like your organization name: `acme-ml-models-prod`.

---

## 3.2 Versioning

Without versioning, uploading a file with the same key *overwrites* the old one permanently. With versioning enabled, every overwrite creates a new version, and you can retrieve or restore any previous version.

```bash
# Enable versioning on a bucket
aws s3api put-bucket-versioning \
    --bucket my-ml-bucket \
    --versioning-configuration Status=Enabled
```

Why this matters for ML: you retrain your model and upload `model.pkl` — but the new version performs worse. With versioning, you roll back by copying the previous version:

```bash
# List versions of a file
aws s3api list-object-versions \
    --bucket my-ml-bucket \
    --prefix models/model.pkl

# Download a specific version
aws s3api get-object \
    --bucket my-ml-bucket \
    --key models/model.pkl \
    --version-id "abc123" \
    restored-model.pkl
```

> ⚠️ **Gotcha:** Versioning keeps *all* old versions, and you pay storage for all of them. A 500 MB model overwritten 10 times means 5 GB stored. Combine versioning with lifecycle policies (next section) to auto-delete old versions after a retention period.

---

## 3.3 Lifecycle Policies

Lifecycle policies automate storage management — move old data to cheaper storage classes or delete it on a schedule.

```json
{
    "Rules": [
        {
            "ID": "archive-old-models",
            "Status": "Enabled",
            "Filter": { "Prefix": "models/" },
            "NoncurrentVersionTransitions": [
                {
                    "NoncurrentDays": 30,
                    "StorageClass": "GLACIER"
                }
            ],
            "NoncurrentVersionExpiration": {
                "NoncurrentDays": 90
            }
        }
    ]
}
```

This says: for files under `models/`, move old versions to Glacier (very cheap, slow retrieval) after 30 days, and delete them entirely after 90 days.

```bash
aws s3api put-bucket-lifecycle-configuration \
    --bucket my-ml-bucket \
    --lifecycle-configuration file://lifecycle.json
```

---

## 3.4 Presigned URLs — Temporary Access Without Credentials

A presigned URL lets anyone download (or upload) a specific file for a limited time, without needing AWS credentials.

Use cases: letting a frontend download a model file, sharing a dataset with a collaborator, allowing an upload from a browser.

```bash
# Generate a download URL valid for 1 hour (3600 seconds)
aws s3 presign s3://my-ml-bucket/models/model.pkl --expires-in 3600
# Output: https://my-ml-bucket.s3.amazonaws.com/models/model.pkl?X-Amz-...
```

Or in Python:

```python
import boto3

s3 = boto3.client("s3")
url = s3.generate_presigned_url(
    "get_object",
    Params={"Bucket": "my-ml-bucket", "Key": "models/model.pkl"},
    ExpiresIn=3600
)
print(url)   # share this URL — it expires in 1 hour
```

> ⚠️ **Gotcha:** Presigned URLs inherit the permissions of *whoever generated them*. If the generating user's permissions are revoked (or the access key is deleted), the URL stops working immediately — even before the expiration time. In production, generate presigned URLs from a dedicated service role rather than a personal user.

---

## 3.5 Bucket Policies vs. IAM Policies

You now know two ways to control S3 access:

| Method | Attached To | Best For |
|---|---|---|
| **IAM Policy** | A user, group, or role | "This user can access these buckets" |
| **Bucket Policy** | The bucket itself | "This bucket is accessible by these accounts/services" |

Both can grant or deny access. When both exist, AWS evaluates all policies together — an explicit `Deny` in *either* always wins.

> ⚠️ **Gotcha:** Public S3 buckets have been the source of massive data breaches (Capital One, US military, countless startups). AWS now blocks public access by default, but be very careful if you ever turn off "Block Public Access" settings. For ML data: there is almost never a reason for a bucket to be public.

---

## ✏️ Hour 3 Exercise

**Goal:** Set up an S3 bucket for your ML model workflow.

1. Create a bucket: `aws s3 mb s3://YOUR-UNIQUE-NAME-ml-models`
2. Enable versioning on it.
3. Upload a file (your model or any test file): `aws s3 cp test-file.txt s3://YOUR-BUCKET/models/v1/test-file.txt`
4. Upload it again (same key) to create a second version.
5. List versions and confirm you see two: `aws s3api list-object-versions --bucket YOUR-BUCKET --prefix models/`
6. Generate a presigned URL for the file and open it in a browser — it should download.
7. **Clean up:** Empty and delete the bucket:
   ```bash
   aws s3 rm s3://YOUR-BUCKET --recursive
   aws s3 rb s3://YOUR-BUCKET
   ```

**Stretch:** Create a lifecycle policy that moves non-current versions to Glacier after 7 days and deletes them after 30 days. Apply it to your bucket with the `put-bucket-lifecycle-configuration` command.

---

---

# Hour 4 — ECR & ECS Basics

## Why This Matters

You have a Docker image of your ML serving API. Running it on a raw EC2 instance works (you did it in Hour 2), but it doesn't scale — you'd have to manually SSH in, pull images, restart processes, and handle failures yourself.

**ECS** (Elastic Container Service) manages containers for you: it starts them, restarts them if they crash, scales them up and down, and load-balances traffic. **ECR** (Elastic Container Registry) is a private Docker Hub hosted inside your AWS account — the place ECS pulls images from.

```
Your laptop → docker push → ECR (stores the image)
                                    ↓
ECS reads task definition → pulls image from ECR → runs containers on EC2 or Fargate
```

---

## 4.1 ECR — Your Private Docker Registry

### Creating a repository

```bash
aws ecr create-repository \
    --repository-name ml-serving-api \
    --region us-east-1
```

This creates a repository — like a folder for your Docker images. The full image URI will be:
`123456789012.dkr.ecr.us-east-1.amazonaws.com/ml-serving-api`

### Logging in to ECR

Before you can push, Docker needs to authenticate with ECR:

```bash
aws ecr get-login-password --region us-east-1 | \
    docker login --username AWS --password-stdin \
    123456789012.dkr.ecr.us-east-1.amazonaws.com
```

Replace `123456789012` with your actual AWS account ID (find it in the console top-right under your account name).

### Pushing an image

```bash
# Tag your local image with the ECR URI
docker tag ml-serving-api:latest \
    123456789012.dkr.ecr.us-east-1.amazonaws.com/ml-serving-api:latest

# Push
docker push 123456789012.dkr.ecr.us-east-1.amazonaws.com/ml-serving-api:latest
```

> ⚠️ **Gotcha:** ECR login tokens expire after **12 hours**. If a push fails with "no basic auth credentials" and you logged in yesterday, just run the `get-login-password` command again.

---

## 4.2 ECS Core Concepts

ECS has four layers:

```
Cluster                          ← a logical group of services
  └── Service                    ← manages running copies of a task (desired count, restarts)
        └── Task                 ← one running instance of your container(s)
              └── Task Definition  ← the blueprint: image, CPU, memory, ports, env vars
```

### Launch types

ECS offers two ways to run containers:

| Launch Type | What It Means | When to Use |
|---|---|---|
| **Fargate** | AWS manages the servers — you never see an EC2 instance | Simpler, no server management, good starting point |
| **EC2** | You manage the servers (EC2 instances) that run containers | More control, GPU access, potentially cheaper at scale |

For this tutorial, we'll use **Fargate** — it's simpler and doesn't require managing instances.

---

## 4.3 Creating a Task Definition

A task definition is the recipe for your container — what image to use, how much CPU and memory it gets, what ports to expose, and what environment variables to inject.

```json
{
    "family": "ml-serving-task",
    "networkMode": "awsvpc",
    "requiresCompatibilities": ["FARGATE"],
    "cpu": "256",
    "memory": "512",
    "executionRoleArn": "arn:aws:iam::123456789012:role/ecsTaskExecutionRole",
    "containerDefinitions": [
        {
            "name": "ml-api",
            "image": "123456789012.dkr.ecr.us-east-1.amazonaws.com/ml-serving-api:latest",
            "portMappings": [
                {
                    "containerPort": 8000,
                    "protocol": "tcp"
                }
            ],
            "environment": [
                {
                    "name": "MODEL_PATH",
                    "value": "/app/models/model.pkl"
                }
            ],
            "logConfiguration": {
                "logDriver": "awslogs",
                "options": {
                    "awslogs-group": "/ecs/ml-serving",
                    "awslogs-region": "us-east-1",
                    "awslogs-stream-prefix": "ecs"
                }
            }
        }
    ]
}
```

Key fields to understand:
- **cpu/memory** — Fargate has fixed valid combinations (256 CPU / 512 MB is the smallest)
- **executionRoleArn** — the IAM role ECS uses to pull images from ECR and write logs
- **logConfiguration** — sends container stdout/stderr to CloudWatch Logs so you can debug without SSH

Register it:

```bash
aws ecs register-task-definition --cli-input-json file://task-definition.json
```

> ⚠️ **Gotcha:** The `ecsTaskExecutionRole` doesn't exist by default — you need to create it. The quickest way: Console → IAM → Create role → select "Elastic Container Service Task" → attach `AmazonECSTaskExecutionRolePolicy`. If this role is missing, ECS can't pull your image from ECR and the task fails silently.

---

## 4.4 Creating a Cluster and Service

```bash
# Create a cluster (Fargate doesn't need EC2 instances — this is just a logical grouping)
aws ecs create-cluster --cluster-name ml-cluster

# Create a service that keeps one copy of your task running
aws ecs create-service \
    --cluster ml-cluster \
    --service-name ml-api-service \
    --task-definition ml-serving-task \
    --desired-count 1 \
    --launch-type FARGATE \
    --network-configuration "awsvpcConfiguration={subnets=[subnet-abc123],securityGroups=[sg-abc123],assignPublicIp=ENABLED}"
```

Replace `subnet-abc123` and `sg-abc123` with your default VPC's subnet and security group IDs. Find them:

```bash
# List default subnets
aws ec2 describe-subnets --filters "Name=default-for-az,Values=true" \
    --query "Subnets[0].SubnetId" --output text

# List default security group
aws ec2 describe-security-groups --filters "Name=group-name,Values=default" \
    --query "SecurityGroups[0].GroupId" --output text
```

> ⚠️ **Gotcha:** `assignPublicIp=ENABLED` is required for Fargate tasks in a public subnet to reach ECR and pull the image. Without it, the task hangs at "PROVISIONING" and eventually times out with no useful error message. This is one of the most frustrating ECS debugging experiences for beginners.

---

## 4.5 How the Pieces Connect

```
You push image → ECR (stores it)
                    ↓
Task Definition (points to ECR image, sets CPU/memory/ports)
                    ↓
Service (uses task def, runs on cluster, maintains desired count)
                    ↓
Cluster (logical home for services)
                    ↓
Fargate (runs the actual container — no EC2 management)
                    ↓
Your API is live at the task's public IP on port 8000
```

To find the public IP of your running task:

```bash
# Get the task ARN
aws ecs list-tasks --cluster ml-cluster --service-name ml-api-service

# Get the network interface
aws ecs describe-tasks --cluster ml-cluster --tasks <task-arn> \
    --query "tasks[0].attachments[0].details[?name=='networkInterfaceId'].value" \
    --output text

# Get the public IP from the network interface
aws ec2 describe-network-interfaces --network-interface-ids <eni-id> \
    --query "NetworkInterfaces[0].Association.PublicIp" --output text
```

---

## ✏️ Hour 4 Exercise

**Goal:** Push an image to ECR and understand the task definition structure.

1. Create an ECR repository: `aws ecr create-repository --repository-name ml-serving-api`
2. Log in to ECR (use the `get-login-password` command from 4.1).
3. Tag and push your Docker image.
4. Verify it's there: `aws ecr list-images --repository-name ml-serving-api`
5. Create the `ecsTaskExecutionRole` in IAM (if it doesn't exist).
6. Write a `task-definition.json` file using the template above, replacing the account ID and region. Register it with `aws ecs register-task-definition`.

**Stretch:** Read through the task definition JSON and change `cpu` to `512` and `memory` to `1024`. Register it as a new revision and note how ECS versions task definitions (revision numbers increment automatically).

---

---

# Hour 5 — Practice: Deploy Your Container on ECS

## Why This Matters

Hours 1–4 taught each piece in isolation. This hour you wire them together end-to-end: push your container to ECR, run it on ECS, hit the live API, and tear it down — the full deployment loop you'll repeat for every release.

---

## 5.1 Pre-Flight Checklist

Before starting, confirm you have:

- [ ] Docker image built and working locally (`docker run -p 8000:8000 ml-serving-api` → hit `localhost:8000/docs`)
- [ ] ECR repository created (from Hour 4)
- [ ] `ecsTaskExecutionRole` IAM role created (from Hour 4)
- [ ] AWS CLI configured and working (`aws sts get-caller-identity` → shows your account)
- [ ] Default VPC subnet and security group IDs noted (from Hour 4)

---

## 5.2 The Full Deployment — Step by Step

### Step 1: Push to ECR

```bash
# Get your account ID
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
REGION=us-east-1

# Log in
aws ecr get-login-password --region $REGION | \
    docker login --username AWS --password-stdin $ACCOUNT_ID.dkr.ecr.$REGION.amazonaws.com

# Tag and push
docker tag ml-serving-api:latest $ACCOUNT_ID.dkr.ecr.$REGION.amazonaws.com/ml-serving-api:latest
docker push $ACCOUNT_ID.dkr.ecr.$REGION.amazonaws.com/ml-serving-api:latest
```

### Step 2: Ensure the security group allows port 8000

```bash
# Get default security group ID
SG_ID=$(aws ec2 describe-security-groups --filters "Name=group-name,Values=default" \
    --query "SecurityGroups[0].GroupId" --output text)

# Add inbound rule for port 8000
aws ec2 authorize-security-group-ingress \
    --group-id $SG_ID \
    --protocol tcp \
    --port 8000 \
    --cidr 0.0.0.0/0
```

### Step 3: Create the cluster

```bash
aws ecs create-cluster --cluster-name ml-cluster
```

### Step 4: Register the task definition

Save your `task-definition.json` (from Hour 4, with your account ID filled in), then:

```bash
aws ecs register-task-definition --cli-input-json file://task-definition.json
```

### Step 5: Create the service

```bash
SUBNET_ID=$(aws ec2 describe-subnets --filters "Name=default-for-az,Values=true" \
    --query "Subnets[0].SubnetId" --output text)

aws ecs create-service \
    --cluster ml-cluster \
    --service-name ml-api-service \
    --task-definition ml-serving-task \
    --desired-count 1 \
    --launch-type FARGATE \
    --network-configuration \
        "awsvpcConfiguration={subnets=[$SUBNET_ID],securityGroups=[$SG_ID],assignPublicIp=ENABLED}"
```

### Step 6: Wait and find the public IP

```bash
# Wait for task to start (check every 15 seconds)
echo "Waiting for task to start..."
sleep 30

# Get the task ARN
TASK_ARN=$(aws ecs list-tasks --cluster ml-cluster --service-name ml-api-service \
    --query "taskArns[0]" --output text)

# Get network interface ID
ENI_ID=$(aws ecs describe-tasks --cluster ml-cluster --tasks $TASK_ARN \
    --query "tasks[0].attachments[0].details[?name=='networkInterfaceId'].value" \
    --output text)

# Get public IP
PUBLIC_IP=$(aws ec2 describe-network-interfaces --network-interface-ids $ENI_ID \
    --query "NetworkInterfaces[0].Association.PublicIp" --output text)

echo "Your API is live at: http://$PUBLIC_IP:8000/docs"
```

### Step 7: Test it

```bash
curl http://$PUBLIC_IP:8000/predict?text=hello
# Should return your prediction JSON
```

---

## 5.3 Debugging When Things Go Wrong

ECS deployments fail silently more often than they should. Here's a debugging checklist:

| Symptom | Likely Cause | How to Check |
|---|---|---|
| Task stuck at PROVISIONING | `assignPublicIp` not ENABLED, or no internet gateway | Check service events in console |
| Task starts then immediately stops | App crashes on startup (import error, missing file) | Check CloudWatch Logs: `/ecs/ml-serving` |
| Task running but API unreachable | Security group doesn't allow port 8000 | Check SG inbound rules |
| "CannotPullContainerError" | ECR login expired, or `ecsTaskExecutionRole` missing ECR permissions | Check task stopped reason in console |

The most useful debugging command:

```bash
aws ecs describe-tasks --cluster ml-cluster --tasks $TASK_ARN \
    --query "tasks[0].stoppedReason"
```

---

## 5.4 Cleanup — Don't Skip This

ECS services and running tasks cost money. Tear everything down:

```bash
# Delete the service (this stops the task too)
aws ecs update-service --cluster ml-cluster --service ml-api-service --desired-count 0
aws ecs delete-service --cluster ml-cluster --service ml-api-service --force

# Delete the cluster
aws ecs delete-cluster --cluster ml-cluster

# (Optional) Delete the ECR repository and its images
aws ecr delete-repository --repository-name ml-serving-api --force
```

> ⚠️ **Gotcha:** `delete-service` without `--force` won't work if the service has running tasks. And setting `desired-count` to 0 first ensures the task stops before you delete the service — skipping this can leave orphaned tasks running (and billing you).

---

## ✏️ Hour 5 Exercise

**Goal:** Complete the full deployment loop.

1. Follow steps 1–7 above to deploy your container to ECS.
2. Hit your API from a browser and from `curl`.
3. Deliberately break something (change the image tag to a version that doesn't exist, or remove the security group rule for port 8000) and practice using the debugging checklist.
4. Fix the issue and redeploy.
5. **Clean up everything** using the commands in section 5.4. Verify in the console that no tasks, services, clusters, or ECR images remain.

**Stretch:** Write a `deploy.sh` script that automates steps 1–6 into a single command. Include the variable lookups for `ACCOUNT_ID`, `SUBNET_ID`, and `SG_ID` so it works on any machine with the right AWS credentials.

---

---

# Hour 6 — AWS CLI & boto3

## Why This Matters

The AWS console is fine for learning, but every click-based workflow is unrepeatable, unscriptable, and unauditable. The **CLI** lets you manage AWS from your terminal. **boto3** (the AWS SDK for Python) lets you manage AWS from your code. Together, they're how real automation happens — and you've already been using the CLI throughout this tutorial. This hour formalizes what you've learned and takes you deeper.

---

## 6.1 Configuring CLI Profiles

So far you've been using a default profile. In practice, you'll have multiple AWS accounts or roles — a personal account, a work staging account, a work production account. Profiles let you switch between them.

```bash
# Configure the default profile
aws configure
# AWS Access Key ID: <your key>
# AWS Secret Access Key: <your secret>
# Default region name: us-east-1
# Default output format: json

# Add a second profile
aws configure --profile staging
# (enter the staging account's credentials)

# Use a specific profile
aws s3 ls --profile staging

# Or set it for the entire terminal session
export AWS_PROFILE=staging
aws s3 ls   # now uses staging automatically
```

Credentials are stored in `~/.aws/credentials`, config in `~/.aws/config`. Both are plain text — protect these files.

> ⚠️ **Gotcha:** If `AWS_PROFILE` is set as an environment variable, it overrides the default profile for *every* command in that terminal. If you switch terminals and forget this, you might deploy to the wrong account. Run `aws sts get-caller-identity` to always confirm which account you're operating in before running destructive commands.

---

## 6.2 CLI Patterns You'll Use Constantly

### Filtering output with `--query`

AWS commands return large JSON blobs. The `--query` flag uses JMESPath syntax to extract what you need:

```bash
# Get just instance IDs
aws ec2 describe-instances \
    --query "Reservations[].Instances[].InstanceId" --output text

# Get running instances only
aws ec2 describe-instances \
    --filters "Name=instance-state-name,Values=running" \
    --query "Reservations[].Instances[].[InstanceId, PublicIpAddress]" \
    --output table
```

### Waiting for operations to complete

Many AWS operations are asynchronous. The CLI has built-in waiters:

```bash
# Launch an instance and wait for it to be running
aws ec2 run-instances --image-id ami-xxx --instance-type t2.micro ...
aws ec2 wait instance-running --instance-ids i-1234567890abcdef0
echo "Instance is ready"

# Wait for an ECS service to stabilize
aws ecs wait services-stable --cluster ml-cluster --services ml-api-service
```

### Dry runs

Some commands support `--dry-run` to test permissions without actually doing anything:

```bash
aws ec2 run-instances --dry-run --image-id ami-xxx --instance-type t2.micro
# Returns success → you have permission
# Returns error → tells you which permission is missing
```

---

## 6.3 boto3 — AWS in Python

boto3 is the Python SDK for AWS. It mirrors the CLI almost exactly but lets you integrate AWS into your scripts, APIs, and data pipelines.

### Setup and clients vs. resources

```python
import boto3

# Client — low-level, maps 1:1 to AWS API calls
s3_client = boto3.client("s3")

# Resource — higher-level, more Pythonic (not available for all services)
s3_resource = boto3.resource("s3")
```

Use a **client** when you need precise control or the resource interface doesn't cover the operation. Use a **resource** for common tasks where readability matters.

### S3 operations

```python
import boto3

s3 = boto3.client("s3")

# Upload a file
s3.upload_file("local-model.pkl", "my-ml-bucket", "models/v1/model.pkl")

# Download a file
s3.download_file("my-ml-bucket", "models/v1/model.pkl", "downloaded-model.pkl")

# List objects in a bucket
response = s3.list_objects_v2(Bucket="my-ml-bucket", Prefix="models/")
for obj in response.get("Contents", []):
    print(f"{obj['Key']}  ({obj['Size']} bytes)")

# Generate a presigned URL
url = s3.generate_presigned_url(
    "get_object",
    Params={"Bucket": "my-ml-bucket", "Key": "models/v1/model.pkl"},
    ExpiresIn=3600
)
```

### EC2 operations

```python
ec2 = boto3.client("ec2")

# List running instances
response = ec2.describe_instances(
    Filters=[{"Name": "instance-state-name", "Values": ["running"]}]
)
for reservation in response["Reservations"]:
    for instance in reservation["Instances"]:
        print(f"{instance['InstanceId']}: {instance.get('PublicIpAddress', 'no public IP')}")

# Stop an instance
ec2.stop_instances(InstanceIds=["i-1234567890abcdef0"])
```

### ECS operations

```python
ecs = boto3.client("ecs")

# List services in a cluster
response = ecs.list_services(cluster="ml-cluster")
print(response["serviceArns"])

# Update service to force new deployment (pull latest image)
ecs.update_service(
    cluster="ml-cluster",
    service="ml-api-service",
    forceNewDeployment=True
)
```

> ⚠️ **Gotcha:** boto3 uses the same credentials as the CLI (from `~/.aws/credentials` or environment variables). If `AWS_PROFILE` is set, boto3 uses it. To use a specific profile in code: `boto3.Session(profile_name="staging").client("s3")`.

---

## 6.4 Error Handling in boto3

AWS calls can fail for many reasons — permissions, rate limits, resources not found. Always handle errors:

```python
from botocore.exceptions import ClientError

s3 = boto3.client("s3")

try:
    s3.download_file("my-bucket", "models/model.pkl", "model.pkl")
except ClientError as e:
    error_code = e.response["Error"]["Code"]
    if error_code == "404" or error_code == "NoSuchKey":
        print("File not found in S3")
    elif error_code == "403":
        print("Access denied — check IAM permissions")
    else:
        print(f"AWS error: {error_code} — {e.response['Error']['Message']}")
```

> ⚠️ **Gotcha:** boto3 error codes are *strings*, not integers. `"404"` not `404`. And the code isn't always the HTTP status — S3 uses `"NoSuchKey"` instead of `"404"` for missing objects. Check the specific service's documentation for the error codes it returns.

---

## 6.5 Putting It Together — A Deployment Script

Here's a Python script that automates the deployment loop from Hour 5:

```python
#!/usr/bin/env python3
"""Deploy the ML serving API to ECS."""
import subprocess
import boto3
import json
import time

REGION = "us-east-1"
REPO_NAME = "ml-serving-api"
CLUSTER = "ml-cluster"
SERVICE = "ml-api-service"

sts = boto3.client("sts")
ecs = boto3.client("ecs")
ecr = boto3.client("ecr")

account_id = sts.get_caller_identity()["Account"]
image_uri = f"{account_id}.dkr.ecr.{REGION}.amazonaws.com/{REPO_NAME}:latest"

# Step 1: Log in to ECR and push
print("Logging in to ECR...")
token_cmd = f"aws ecr get-login-password --region {REGION}"
login_cmd = f"docker login --username AWS --password-stdin {account_id}.dkr.ecr.{REGION}.amazonaws.com"
subprocess.run(f"{token_cmd} | {login_cmd}", shell=True, check=True)

print("Tagging and pushing image...")
subprocess.run(["docker", "tag", "ml-serving-api:latest", image_uri], check=True)
subprocess.run(["docker", "push", image_uri], check=True)

# Step 2: Force ECS to pull the new image
print("Triggering new ECS deployment...")
ecs.update_service(
    cluster=CLUSTER,
    service=SERVICE,
    forceNewDeployment=True
)

# Step 3: Wait for stability
print("Waiting for service to stabilize...")
waiter = ecs.get_waiter("services_stable")
waiter.wait(cluster=CLUSTER, services=[SERVICE])

print("Deployment complete!")
```

---

## ✏️ Hour 6 Exercise

**Goal:** Replace your manual workflow with scripts.

1. Configure at least two CLI profiles (even if they point to the same account): `default` and `staging`.
2. Run `aws sts get-caller-identity --profile staging` to confirm the profile works.
3. Write a Python script that:
   - Lists all S3 buckets in the account
   - Lists all objects in a specific bucket
   - Uploads a file and generates a presigned URL for it
   - Handles the `NoSuchBucket` error gracefully
4. Run the script and verify the presigned URL works in a browser.

**Stretch:** Extend the deployment script from section 6.5 to also check if the ECR repository and ECS cluster exist first, and create them if they don't — making the script fully idempotent (safe to run multiple times).

---

---

# Gotchas Summary Table

| # | Gotcha | Where It Bites | Fix |
|---|---|---|---|
| 1 | Root account without MFA | Hr 1 | Enable MFA immediately after account creation |
| 2 | `"Resource": "*"` in IAM policies | Hr 1 | Always scope to specific ARNs |
| 3 | Security group blocks all inbound by default | Hr 2 | Explicitly open required ports (22, 8000, etc.) |
| 4 | `.pem` key file lost = no SSH access | Hr 2 | Back up key files; there's no recovery |
| 5 | EC2 public IP changes on stop/start | Hr 2 | Use an Elastic IP for permanent addresses |
| 6 | S3 bucket names are globally unique | Hr 3 | Use org-specific prefixes to avoid collisions |
| 7 | Versioning stores all versions (costs add up) | Hr 3 | Combine with lifecycle policies to auto-expire old versions |
| 8 | Presigned URL inherits generator's permissions | Hr 3 | Generate from a service role, not a personal user |
| 9 | Public S3 buckets cause data breaches | Hr 3 | Never disable "Block Public Access" unless you fully understand the implications |
| 10 | ECR login token expires after 12 hours | Hr 4 | Re-run `get-login-password` before each push |
| 11 | `ecsTaskExecutionRole` doesn't exist by default | Hr 4 | Create it in IAM with `AmazonECSTaskExecutionRolePolicy` |
| 12 | Fargate task stuck at PROVISIONING | Hr 4, 5 | Ensure `assignPublicIp=ENABLED` in network config |
| 13 | Forgetting to clean up ECS resources | Hr 5 | Set desired-count to 0, delete service, delete cluster |
| 14 | `AWS_PROFILE` env var overrides default | Hr 6 | Always run `aws sts get-caller-identity` before destructive commands |
| 15 | boto3 error codes are strings, not ints | Hr 6 | Check for `"NoSuchKey"` not `404`; read the service docs |

---

---

# Quick Reference Card

## AWS CLI Essentials

```bash
# Check who you're logged in as
aws sts get-caller-identity

# Switch profiles
export AWS_PROFILE=staging

# Common output formats
--output json    # default, full detail
--output table   # human-readable
--output text    # script-friendly, tab-separated
```

## S3

```bash
aws s3 cp local.txt s3://bucket/key           # upload
aws s3 cp s3://bucket/key local.txt            # download
aws s3 sync ./dir s3://bucket/prefix/          # sync directory
aws s3 ls s3://bucket/prefix/                  # list
aws s3 rm s3://bucket --recursive              # empty bucket
aws s3 rb s3://bucket                          # delete bucket
aws s3 presign s3://bucket/key --expires-in 3600  # presigned URL
```

## EC2

```bash
aws ec2 run-instances --image-id ami-xxx --instance-type t2.micro --key-name my-key
aws ec2 describe-instances --filters "Name=instance-state-name,Values=running"
aws ec2 stop-instances --instance-ids i-xxx
aws ec2 terminate-instances --instance-ids i-xxx
ssh -i my-key.pem ec2-user@<public-ip>
```

## ECR

```bash
aws ecr create-repository --repository-name my-repo
aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin <acct>.dkr.ecr.us-east-1.amazonaws.com
docker tag local-image:latest <acct>.dkr.ecr.us-east-1.amazonaws.com/my-repo:latest
docker push <acct>.dkr.ecr.us-east-1.amazonaws.com/my-repo:latest
```

## ECS

```bash
aws ecs create-cluster --cluster-name my-cluster
aws ecs register-task-definition --cli-input-json file://task-def.json
aws ecs create-service --cluster my-cluster --service-name my-svc --task-definition my-task --desired-count 1 --launch-type FARGATE --network-configuration "awsvpcConfiguration={...}"
aws ecs update-service --cluster my-cluster --service my-svc --force-new-deployment
aws ecs wait services-stable --cluster my-cluster --services my-svc
```

## boto3 Pattern

```python
import boto3
from botocore.exceptions import ClientError

client = boto3.client("s3")  # or "ec2", "ecs", "ecr"

try:
    response = client.some_operation(Param="value")
except ClientError as e:
    code = e.response["Error"]["Code"]
    msg = e.response["Error"]["Message"]
    print(f"AWS error {code}: {msg}")

# Use a named profile
session = boto3.Session(profile_name="staging")
client = session.client("s3")
```

## Free-Tier Limits (12-Month)

| Service | Free Allowance |
|---|---|
| EC2 | 750 hrs/month of `t2.micro` |
| S3 | 5 GB storage, 20k GET, 2k PUT |
| ECR | 500 MB storage |
| ECS (Fargate) | Not free-tier — charged per vCPU/memory/hour |
| Data transfer | 100 GB/month out |

## Cleanup Checklist

```
[ ] EC2: terminate instances, release Elastic IPs
[ ] S3: empty buckets, delete buckets
[ ] ECR: delete repositories
[ ] ECS: set desired-count to 0, delete services, delete clusters
[ ] IAM: delete test users, remove unused access keys
```
