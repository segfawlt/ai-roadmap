# Final Exam: Deploy FastAPI Application to AWS Cloud

## Assignment Metadata

| Field | Description |
|----|----|
| **Assignment Name** | Deploy FastAPI Application to AWS Cloud |
| **Course** | 05 - Basic Cloud Essentials for Developer |
| **Project Name** | cloud-fastapi-deployment |
| **Estimated Time** | 480 minutes (8 hours) |
| **Framework** | FastAPI, Docker, AWS (ECR, EC2, S3, RDS/DynamoDB, IAM), GitHub Actions |

------------------------------------------------------------------------

## overview

In this project exam, you will take the FastAPI application you built in **Module 03 - Basic Building Monolith API with FastAPI** and deploy it to AWS Cloud. You will replace local services with AWS managed services and set up a CI/CD pipeline for automated deployments.

### Architecture Diagram

``` mermaid
        flowchart TD
    subgraph AWS["AWS Cloud"]
        GH["GitHub Actions"] --> ECR["ECR<br>(Images)"]
        ECR -- Pull Image --> Deploy
        subgraph VPC
            subgraph EC2["EC2 Instance"]
                App["FastAPI<br>(Docker)"]
            end
            App --> RDS["RDS<br>(PostgreSQL)"]
        end
        Deploy --> EC2
        App --> S3["S3<br>(Storage)"]
        IAM["IAM<br>(Access Control for all services)"]
    end
```

------------------------------------------------------------------------

## Learning Objectives

By completing this exam, you will be able to:

- **Create** IAM users and configure appropriate permissions for AWS services

- **Build** Docker images and push them to Amazon ECR

- **Deploy** containerized FastAPI applications on Amazon EC2

- **Configure** Amazon RDS (PostgreSQL) as the database backend

- **Implement** file storage using Amazon S3

- **Set up** CI/CD pipeline using GitHub Actions

- **Apply** security best practices with VPC and security groups

- **Utilize** environment variables and AWS credentials securely

------------------------------------------------------------------------

## Prerequisites

Before starting this exam, ensure you have:

Completed Module 03: Basic Building Monolith API with FastAPI

AWS Account with appropriate permissions

GitHub account

Docker installed locally

AWS CLI installed and configured

Basic understanding of cloud concepts from Module 05 lectures

------------------------------------------------------------------------

## Part 1: IAM Configuration (15%)

### Task 1.1: Create IAM User

Create a new IAM user for programmatic access with the following specifications:

| Specification      | Value               |
|--------------------|---------------------|
| **User Name**      | `fastapi-deployer`  |
| **Access Type**    | Programmatic access |
| **Console Access** | Optional            |

### Task 1.2: Create and Attach Policies

Create the following permissions for your IAM user:

#### Required Permissions

| Service | Actions Required |
|----|----|
| **ECR** | `ecr:GetAuthorizationToken`, `ecr:BatchCheckLayerAvailability`, `ecr:GetDownloadUrlForLayer`, `ecr:BatchGetImage`, `ecr:PutImage`, `ecr:InitiateLayerUpload`, `ecr:UploadLayerPart`, `ecr:CompleteLayerUpload` |
| **EC2** | `ec2:DescribeInstances`, `ec2:StartInstances`, `ec2:StopInstances` |
| **S3** | `s3:GetObject`, `s3:PutObject`, `s3:DeleteObject`, `s3:ListBucket` |
| **RDS** | `rds:DescribeDBInstances`, `rds:Connect` |

### Task 1.3: Configure AWS CLI

Configure your local AWS CLI with the created credentials:

``` bash
aws configure
# Enter your Access Key ID, Secret Access Key, Region (ap-southeast-1), Output format (json)
```

### Deliverables for Part 1

Screenshot of IAM user creation

Screenshot of attached policies

Verification of `aws`` ``sts`` ``get-caller-identity` output

------------------------------------------------------------------------

## Part 2: Database Setup with Amazon RDS (20%)

### Task 2.1: Create RDS PostgreSQL Instance

Create an RDS instance with the following specifications:

| Specification            | Value                      |
|--------------------------|----------------------------|
| **Engine**               | PostgreSQL 15.x            |
| **Instance Class**       | db.t3.micro (Free Tier)    |
| **Storage**              | 20 GB gp2                  |
| **DB Instance ID**       | `fastapi-db`               |
| **Master Username**      | `postgres`                 |
| **Database Name**        | `fastapi_prod`             |
| **VPC**                  | Default VPC                |
| **Public Accessibility** | Yes (for development only) |
| **Security Group**       | Allow PostgreSQL (5432)    |

### Task 2.2: Update FastAPI Database Configuration

Modify your FastAPI application’s `db.py` to use environment variables:

``` python
import os
from sqlalchemy.ext.asyncio import create_async_engine, AsyncSession
from sqlalchemy.orm import sessionmaker

# Use environment variables for database connection
DATABASE_URL = os.getenv(
    "DATABASE_URL",
    "postgresql+asyncpg://postgres:password@localhost:5432/fastapi_prod"
)

engine = create_async_engine(DATABASE_URL, echo=True)

AsyncSessionLocal = sessionmaker(
    engine, expire_on_commit=False, class_=AsyncSession
)

async def get_db_connection():
    db = AsyncSessionLocal()
    try:
        yield db
    finally:
        await db.close()
```

### Task 2.3: Test Database Connection

Verify connectivity to RDS from your local machine:

``` bash
# Using psql
psql -h <rds-endpoint> -U postgres -d fastapi_prod

# Or using Python
python -c "import asyncio; from db import engine; asyncio.run(engine.connect())"
```

### Deliverables for Part 2

Screenshot of RDS instance details

Updated `db.py` with environment variable support

Screenshot of successful database connection

------------------------------------------------------------------------

## Part 3: File Storage with Amazon S3 (15%)

### Task 3.1: Create S3 Bucket

Create an S3 bucket for storing application files:

| Specification           | Value                         |
|-------------------------|-------------------------------|
| **Bucket Name**         | `fastapi-app-files-<your-id>` |
| **Region**              | ap-southeast-1                |
| **Block Public Access** | Enabled                       |
| **Versioning**          | Enabled (optional)            |

### Task 3.2: Implement S3 File Upload in FastAPI

Add a new endpoint for file uploads using boto3:

``` python
# s3_service.py
import boto3
from botocore.exceptions import ClientError
import os

class S3Service:
    def __init__(self):
        self.s3_client = boto3.client(
            "s3",
            region_name=os.getenv("AWS_REGION", "ap-southeast-1"),
            aws_access_key_id=os.getenv("AWS_ACCESS_KEY_ID"),
            aws_secret_access_key=os.getenv("AWS_SECRET_ACCESS_KEY"),
        )
        self.bucket_name = os.getenv("S3_BUCKET_NAME")

    async def upload_file(self, file_path: str, object_name: str) -> bool:
        try:
            self.s3_client.upload_file(file_path, self.bucket_name, object_name)
            return True
        except ClientError as e:
            print(f"Upload failed: {e}")
            return False

    async def download_file(self, object_name: str, file_path: str) -> bool:
        try:
            self.s3_client.download_file(self.bucket_name, object_name, file_path)
            return True
        except ClientError as e:
            print(f"Download failed: {e}")
            return False

    def get_presigned_url(self, object_name: str, expiration: int = 3600) -> str:
        try:
            url = self.s3_client.generate_presigned_url(
                'get_object',
                Params={'Bucket': self.bucket_name, 'Key': object_name},
                ExpiresIn=expiration
            )
            return url
        except ClientError:
            return None
```

### Task 3.3: Create File Upload Endpoint

``` python
# In main.py
from fastapi import UploadFile, File
from s3_service import S3Service
import tempfile

s3_service = S3Service()

@app.post("/files/upload")
async def upload_file(file: UploadFile = File(...)):
    # Save temporarily
    with tempfile.NamedTemporaryFile(delete=False) as tmp:
        content = await file.read()
        tmp.write(content)
        tmp_path = tmp.name

    # Upload to S3
    object_name = f"uploads/{file.filename}"
    success = await s3_service.upload_file(tmp_path, object_name)

    if success:
        url = s3_service.get_presigned_url(object_name)
        return {"message": "File uploaded", "url": url}
    return {"error": "Upload failed"}
```

### Deliverables for Part 3

Screenshot of S3 bucket configuration

Implementation of `s3_service.py`

File upload endpoint in FastAPI

Screenshot of successful file upload to S3

------------------------------------------------------------------------

## Part 4: Containerization with Amazon ECR (20%)

### Task 4.1: Create ECR Repository

Create an ECR repository for your FastAPI application:

``` bash
aws ecr create-repository \
    --repository-name fastapi-app \
    --region ap-southeast-1
```

### Task 4.2: Create Dockerfile

Create a production-ready Dockerfile:

``` dockerfile
# Dockerfile
FROM python:3.11-slim

WORKDIR /app

# Install dependencies
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Copy application code
COPY . .

# Expose port
EXPOSE 8000

# Run with uvicorn
CMD ["uvicorn", "main:app", "--host", "0.0.0.0", "--port", "8000"]
```

### Task 4.3: Create docker-compose.yml

``` yaml
# docker-compose.yml
version: "3.8"

services:
  fastapi:
    build: .
    ports:
      - "8000:8000"
    environment:
      - DATABASE_URL=${DATABASE_URL}
      - AWS_ACCESS_KEY_ID=${AWS_ACCESS_KEY_ID}
      - AWS_SECRET_ACCESS_KEY=${AWS_SECRET_ACCESS_KEY}
      - AWS_REGION=${AWS_REGION}
      - S3_BUCKET_NAME=${S3_BUCKET_NAME}
    restart: unless-stopped
```

### Task 4.4: Build and Push to ECR

``` bash
# Authenticate Docker with ECR
aws ecr get-login-password --region ap-southeast-1 | \
    docker login --username AWS --password-stdin <account-id>.dkr.ecr.ap-southeast-1.amazonaws.com

# Build image
docker build -t fastapi-app .

# Tag image
docker tag fastapi-app:latest <account-id>.dkr.ecr.ap-southeast-1.amazonaws.com/fastapi-app:latest

# Push image
docker push <account-id>.dkr.ecr.ap-southeast-1.amazonaws.com/fastapi-app:latest
```

### Deliverables for Part 4

Screenshot of ECR repository

Dockerfile

docker-compose.yml

Screenshot of pushed image in ECR

------------------------------------------------------------------------

## Part 5: Deploy to Amazon EC2 (20%)

### Task 5.1: Launch EC2 Instance

Launch an EC2 instance with the following specifications:

| Specification      | Value                                        |
|--------------------|----------------------------------------------|
| **AMI**            | Amazon Linux 2023 or Ubuntu 22.04            |
| **Instance Type**  | t2.micro (Free Tier)                         |
| **Key Pair**       | Create or use existing                       |
| **Security Group** | Allow SSH (22), HTTP (80), Custom TCP (8000) |
| **Storage**        | 8 GB gp3                                     |

### Task 5.2: Configure EC2 Instance

SSH into your EC2 instance and install required software:

``` bash
# Connect to EC2
ssh -i your-key.pem ec2-user@<ec2-public-ip>

# Install Docker (Amazon Linux 2023)
sudo yum update -y
sudo yum install -y docker
sudo systemctl start docker
sudo systemctl enable docker
sudo usermod -aG docker ec2-user

# Install AWS CLI (if not pre-installed)
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip
sudo ./aws/install

# Configure AWS credentials
aws configure
```

### Task 5.3: Deploy Application on EC2

``` bash
# Authenticate with ECR
aws ecr get-login-password --region ap-southeast-1 | \
    docker login --username AWS --password-stdin <account-id>.dkr.ecr.ap-southeast-1.amazonaws.com

# Pull image
docker pull <account-id>.dkr.ecr.ap-southeast-1.amazonaws.com/fastapi-app:latest

# Run container
docker run -d \
    --name fastapi-app \
    -p 8000:8000 \
    -e DATABASE_URL="postgresql+asyncpg://postgres:password@<rds-endpoint>:5432/fastapi_prod" \
    -e AWS_ACCESS_KEY_ID="<your-access-key>" \
    -e AWS_SECRET_ACCESS_KEY="<your-secret-key>" \
    -e AWS_REGION="ap-southeast-1" \
    -e S3_BUCKET_NAME="fastapi-app-files-<your-id>" \
    <account-id>.dkr.ecr.ap-southeast-1.amazonaws.com/fastapi-app:latest
```

### Task 5.4: Verify Deployment

Test your deployed application:

``` bash
# Test health endpoint
curl http://<ec2-public-ip>:8000/

# Test API endpoints
curl http://<ec2-public-ip>:8000/users/
curl -X POST http://<ec2-public-ip>:8000/users/ \
    -H "Content-Type: application/json" \
    -d '{"name": "Test User", "email": "[email protected]"}'
```

### Deliverables for Part 5

Screenshot of running EC2 instance

Screenshot of security group configuration

Screenshot of running Docker container (`docker`` ``ps`)

Screenshot of successful API response from EC2

------------------------------------------------------------------------

## Part 6: CI/CD Pipeline with GitHub Actions (10%)

### Task 6.1: Create GitHub Actions Workflow

Create `.github/workflows/deploy.yml`:

``` yaml
name: Deploy to AWS

on:
  push:
    branches:
      - main

env:
  AWS_REGION: ap-southeast-1
  ECR_REPOSITORY: fastapi-app

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest

    steps:
      - name: Checkout code
        uses: actions/checkout@v4

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ${{ env.AWS_REGION }}

      - name: Login to Amazon ECR
        run: |
          aws ecr get-login-password --region ${{ env.AWS_REGION }} | \
            docker login --username AWS --password-stdin ${{ secrets.AWS_ACCOUNT_ID }}.dkr.ecr.${{ env.AWS_REGION }}.amazonaws.com

      - name: Build and push Docker image
        run: |
          docker build -t ${{ env.ECR_REPOSITORY }} .
          docker tag ${{ env.ECR_REPOSITORY }}:latest ${{ secrets.AWS_ACCOUNT_ID }}.dkr.ecr.${{ env.AWS_REGION }}.amazonaws.com/${{ env.ECR_REPOSITORY }}:latest
          docker tag ${{ env.ECR_REPOSITORY }}:latest ${{ secrets.AWS_ACCOUNT_ID }}.dkr.ecr.${{ env.AWS_REGION }}.amazonaws.com/${{ env.ECR_REPOSITORY }}:${{ github.sha }}
          docker push ${{ secrets.AWS_ACCOUNT_ID }}.dkr.ecr.${{ env.AWS_REGION }}.amazonaws.com/${{ env.ECR_REPOSITORY }}:latest
          docker push ${{ secrets.AWS_ACCOUNT_ID }}.dkr.ecr.${{ env.AWS_REGION }}.amazonaws.com/${{ env.ECR_REPOSITORY }}:${{ github.sha }}

      - name: Deploy to EC2
        uses: appleboy/[email protected]
        with:
          host: ${{ secrets.EC2_HOST }}
          username: ec2-user
          key: ${{ secrets.EC2_SSH_KEY }}
          script: |
            aws ecr get-login-password --region ${{ env.AWS_REGION }} | \
              docker login --username AWS --password-stdin ${{ secrets.AWS_ACCOUNT_ID }}.dkr.ecr.${{ env.AWS_REGION }}.amazonaws.com

            docker pull ${{ secrets.AWS_ACCOUNT_ID }}.dkr.ecr.${{ env.AWS_REGION }}.amazonaws.com/${{ env.ECR_REPOSITORY }}:latest

            docker stop fastapi-app || true
            docker rm fastapi-app || true

            docker run -d \
              --name fastapi-app \
              -p 8000:8000 \
              -e DATABASE_URL="${{ secrets.DATABASE_URL }}" \
              -e AWS_ACCESS_KEY_ID="${{ secrets.AWS_ACCESS_KEY_ID }}" \
              -e AWS_SECRET_ACCESS_KEY="${{ secrets.AWS_SECRET_ACCESS_KEY }}" \
              -e AWS_REGION="${{ env.AWS_REGION }}" \
              -e S3_BUCKET_NAME="${{ secrets.S3_BUCKET_NAME }}" \
              ${{ secrets.AWS_ACCOUNT_ID }}.dkr.ecr.${{ env.AWS_REGION }}.amazonaws.com/${{ env.ECR_REPOSITORY }}:latest

            docker image prune -f
```

### Task 6.2: Configure GitHub Secrets

Add the following secrets to your GitHub repository:

| Secret Name             | Description                    |
|-------------------------|--------------------------------|
| `AWS_ACCESS_KEY_ID`     | IAM user access key            |
| `AWS_SECRET_ACCESS_KEY` | IAM user secret key            |
| `AWS_ACCOUNT_ID`        | Your AWS account ID            |
| `EC2_HOST`              | EC2 instance public IP         |
| `EC2_SSH_KEY`           | Private key for EC2 SSH access |
| `DATABASE_URL`          | Full RDS connection string     |
| `S3_BUCKET_NAME`        | S3 bucket name                 |

### Deliverables for Part 6

GitHub Actions workflow file

Screenshot of GitHub secrets configuration

Screenshot of successful workflow run

Screenshot of automated deployment to EC2

------------------------------------------------------------------------

## Submission Requirements

### Required Deliverables

| Deliverable | Description |
|----|----|
| **Source Code** | Complete FastAPI application with AWS integration |
| **Dockerfile** | Production-ready Dockerfile |
| **docker-compose.yml** | Docker Compose configuration |
| **GitHub Actions Workflow** | CI/CD pipeline configuration |
| **README.md** | Setup and deployment instructions |
| **Screenshots** | All required screenshots from each part |

### README.md Requirements

Your README.md must include:

1.  **Project overview** - Brief description of the application

2.  **Architecture Diagram** - Visual representation of AWS services used

3.  **Prerequisites** - Required tools and accounts

4.  **Local Development Setup** - How to run locally

5.  **AWS Resources** - List of AWS resources created

6.  **Environment Variables** - All required environment variables

7.  **Deployment Instructions** - Step-by-step deployment guide

8.  **API Documentation** - Available endpoints and usage

### Submission Checklist

All source code committed to GitHub repository

Application successfully deployed to EC2

All API endpoints working correctly

CI/CD pipeline triggered and completed successfully

README.md complete with all sections

All screenshots included

------------------------------------------------------------------------

## Grading Rubric

| Part | Weight | Criteria |
|----|----|----|
| **Part 1: IAM Configuration** | 15% | Correct IAM setup, appropriate permissions |
| **Part 2: Database (RDS)** | 20% | RDS created, connected, environment variables used |
| **Part 3: Storage (S3)** | 15% | S3 bucket, file upload/download working |
| **Part 4: Containerization** | 20% | Dockerfile, ECR push, image versioning |
| **Part 5: Deployment (EC2)** | 20% | EC2 running, application accessible, security groups |
| **Part 6: CI/CD Pipeline** | 10% | GitHub Actions working, automated deployment |

### Grading Scale

| Grade | Score Range | Description                                    |
|-------|-------------|------------------------------------------------|
| A     | 90-100%     | All requirements met, excellent implementation |
| B     | 80-89%      | Most requirements met, minor issues            |
| C     | 70-79%      | Core requirements met, some features missing   |
| D     | 60-69%      | Partial implementation, significant issues     |
| F     | Below 60%   | Major requirements not met                     |

------------------------------------------------------------------------

## Bonus Challenges (Optional)

### Bonus 1: HTTPS with Load Balancer (+5%)

- Set up Application Load Balancer with SSL certificate

- Configure Route 53 for custom domain

### Bonus 2: Environment-based Configuration (+5%)

- Implement separate staging and production environments

- Use AWS Systems Manager Parameter Store for secrets

### Bonus 3: Monitoring and Logging (+5%)

- Set up CloudWatch for application logs

- Create CloudWatch alarms for critical metrics

### Bonus 4: Database Migrations in CI/CD (+5%)

- Integrate Alembic migrations into GitHub Actions workflow

- Implement rollback strategy

------------------------------------------------------------------------

## Resources

- [AWS Documentation](https://docs.aws.amazon.com/)

- [FastAPI Documentation](https://fastapi.tiangolo.com/)

- [Docker Documentation](https://docs.docker.com/)

- [GitHub Actions Documentation](https://docs.github.com/en/actions)

- Module 05 Knowledge Base

------------------------------------------------------------------------

## Notes

- Never commit AWS credentials to version control

- Use IAM roles with least-privilege permissions

- Keep RDS in private subnet for production

- Enable VPC flow logs for network monitoring

- Regularly rotate access keys

<!-- -->

- Use Free Tier eligible resources when possible

- Set up billing alerts to monitor costs

- Stop EC2 instances when not in use

- Delete unused resources after the exam
