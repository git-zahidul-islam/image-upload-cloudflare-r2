# Deploy to AWS - Image Upload Service

Complete guide to deploy the Image Upload Service on AWS using different methods.

## Table of Contents
- [AWS EC2 Deployment](#aws-ec2-deployment)
- [AWS Elastic Beanstalk](#aws-elastic-beanstalk)
- [AWS ECS (Docker)](#aws-ecs-docker)
- [AWS Lambda + API Gateway](#aws-lambda--api-gateway)

---

## AWS EC2 Deployment

### Prerequisites
- AWS Account
- EC2 instance (Ubuntu/Amazon Linux)
- Security Group allowing port 80/443

### Step 1: Launch EC2 Instance

1. Go to AWS Console → EC2 → Launch Instance
2. Choose **Ubuntu 22.04 LTS** or **Amazon Linux 2023**
3. Instance type: **t2.micro** (free tier) or **t3.small**
4. Configure Security Group:
   - SSH (22) - Your IP
   - HTTP (80) - 0.0.0.0/0
   - HTTPS (443) - 0.0.0.0/0
5. Create/select key pair
6. Launch instance

### Step 2: Connect to EC2

```bash
ssh -i your-key.pem ubuntu@your-ec2-public-ip
```

### Step 3: Install Go

```bash
# Update system
sudo apt update && sudo apt upgrade -y

# Install Go
wget https://go.dev/dl/go1.25.1.linux-amd64.tar.gz
sudo tar -C /usr/local -xzf go1.25.1.linux-amd64.tar.gz

# Add to PATH
echo 'export PATH=$PATH:/usr/local/go/bin' >> ~/.bashrc
source ~/.bashrc

# Verify
go version
```

### Step 4: Deploy Application

```bash
# Clone repository
git clone https://github.com/yourusername/image-upload-r2.git
cd image-upload-r2

# Install dependencies
go mod download

# Create .env file
nano .env
```

Add your configuration:
```env
PORT=8080
R2_ACCOUNT_ID=your_account_id
R2_ACCESS_KEY=your_access_key
R2_SECRET_KEY=your_secret_key
R2_BUCKET_NAME=your_bucket_name
R2_PUBLIC_URL=https://your-cdn-url.com
```

### Step 5: Build and Run

```bash
# Build
go build -o image-upload main.go

# Run in background
nohup ./image-upload > app.log 2>&1 &
```

### Step 6: Setup as System Service (Recommended)

Create service file:
```bash
sudo nano /etc/systemd/system/image-upload.service
```

Add:
```ini
[Unit]
Description=Image Upload Service
After=network.target

[Service]
Type=simple
User=ubuntu
WorkingDirectory=/home/ubuntu/image-upload-r2
ExecStart=/home/ubuntu/image-upload-r2/image-upload
Restart=always
RestartSec=5
Environment="PATH=/usr/local/go/bin:/usr/bin:/bin"

[Install]
WantedBy=multi-user.target
```

Enable and start:
```bash
sudo systemctl daemon-reload
sudo systemctl enable image-upload
sudo systemctl start image-upload
sudo systemctl status image-upload
```

### Step 7: Setup Nginx Reverse Proxy (Optional)

```bash
# Install Nginx
sudo apt install nginx -y

# Configure
sudo nano /etc/nginx/sites-available/image-upload
```

Add:
```nginx
server {
    listen 80;
    server_name your-domain.com;

    location / {
        proxy_pass http://localhost:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        
        # File upload settings
        client_max_body_size 10M;
    }
}
```

Enable:
```bash
sudo ln -s /etc/nginx/sites-available/image-upload /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl restart nginx
```

### Step 8: Setup SSL with Let's Encrypt

```bash
# Install Certbot
sudo apt install certbot python3-certbot-nginx -y

# Get certificate
sudo certbot --nginx -d your-domain.com

# Auto-renewal is configured automatically
```

---

## AWS Elastic Beanstalk

### Step 1: Install EB CLI

```bash
pip install awsebcli
```

### Step 2: Initialize Application

```bash
cd image-upload-r2
eb init -p go image-upload-service --region us-east-1
```

### Step 3: Create Buildfile

Create `Buildfile`:
```bash
make: go build -o application main.go
```

### Step 4: Create Procfile

Create `Procfile`:
```
web: ./application
```

### Step 5: Configure Environment Variables

```bash
eb create image-upload-env
eb setenv PORT=8080 \
  R2_ACCOUNT_ID=your_account_id \
  R2_ACCESS_KEY=your_access_key \
  R2_SECRET_KEY=your_secret_key \
  R2_BUCKET_NAME=your_bucket_name \
  R2_PUBLIC_URL=https://your-cdn-url.com
```

### Step 6: Deploy

```bash
eb deploy
eb open
```

---

## AWS ECS (Docker)

### Step 1: Create Dockerfile

Create `Dockerfile`:
```dockerfile
FROM golang:1.25-alpine AS builder
WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN go build -o image-upload main.go

FROM alpine:latest
RUN apk --no-cache add ca-certificates
WORKDIR /root/
COPY --from=builder /app/image-upload .
EXPOSE 8080
CMD ["./image-upload"]
```

### Step 2: Build and Push to ECR

```bash
# Create ECR repository
aws ecr create-repository --repository-name image-upload-service

# Login to ECR
aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin YOUR_ACCOUNT_ID.dkr.ecr.us-east-1.amazonaws.com

# Build and push
docker build -t image-upload-service .
docker tag image-upload-service:latest YOUR_ACCOUNT_ID.dkr.ecr.us-east-1.amazonaws.com/image-upload-service:latest
docker push YOUR_ACCOUNT_ID.dkr.ecr.us-east-1.amazonaws.com/image-upload-service:latest
```

### Step 3: Create ECS Task Definition

Create `task-definition.json`:
```json
{
  "family": "image-upload-task",
  "networkMode": "awsvpc",
  "requiresCompatibilities": ["FARGATE"],
  "cpu": "256",
  "memory": "512",
  "containerDefinitions": [
    {
      "name": "image-upload",
      "image": "YOUR_ACCOUNT_ID.dkr.ecr.us-east-1.amazonaws.com/image-upload-service:latest",
      "portMappings": [
        {
          "containerPort": 8080,
          "protocol": "tcp"
        }
      ],
      "environment": [
        {"name": "PORT", "value": "8080"},
        {"name": "R2_ACCOUNT_ID", "value": "your_account_id"},
        {"name": "R2_ACCESS_KEY", "value": "your_access_key"},
        {"name": "R2_SECRET_KEY", "value": "your_secret_key"},
        {"name": "R2_BUCKET_NAME", "value": "your_bucket_name"},
        {"name": "R2_PUBLIC_URL", "value": "https://your-cdn-url.com"}
      ]
    }
  ]
}
```

### Step 4: Deploy to ECS

```bash
# Register task definition
aws ecs register-task-definition --cli-input-json file://task-definition.json

# Create cluster
aws ecs create-cluster --cluster-name image-upload-cluster

# Create service with ALB (Application Load Balancer)
# Follow AWS Console wizard for ECS Service creation
```

---

## AWS Lambda + API Gateway

### Step 1: Install AWS Lambda Go Runtime

```bash
go get github.com/aws/aws-lambda-go/lambda
go get github.com/aws/aws-lambda-go/events
```

### Step 2: Create Lambda Handler

Create `lambda/main.go`:
```go
package main

import (
    "context"
    "encoding/base64"
    "github.com/aws/aws-lambda-go/events"
    "github.com/aws/aws-lambda-go/lambda"
    // ... your upload logic
)

func handler(ctx context.Context, request events.APIGatewayProxyRequest) (events.APIGatewayProxyResponse, error) {
    // Decode base64 image from request.Body
    // Upload to R2
    // Return response
    return events.APIGatewayProxyResponse{
        StatusCode: 200,
        Body:       `{"status": 200, "message": "Success"}`,
    }, nil
}

func main() {
    lambda.Start(handler)
}
```

### Step 3: Build for Lambda

```bash
GOOS=linux GOARCH=amd64 go build -o bootstrap lambda/main.go
zip function.zip bootstrap
```

### Step 4: Deploy to Lambda

```bash
# Create Lambda function
aws lambda create-function \
  --function-name image-upload \
  --runtime provided.al2 \
  --handler bootstrap \
  --zip-file fileb://function.zip \
  --role arn:aws:iam::YOUR_ACCOUNT_ID:role/lambda-execution-role \
  --environment Variables="{R2_ACCOUNT_ID=xxx,R2_ACCESS_KEY=xxx,R2_SECRET_KEY=xxx,R2_BUCKET_NAME=xxx,R2_PUBLIC_URL=xxx}"
```

### Step 5: Create API Gateway

1. Go to API Gateway Console
2. Create REST API
3. Create POST method → Integrate with Lambda
4. Deploy API
5. Get invoke URL

---

## Cost Comparison

| Method | Monthly Cost (Estimate) | Best For |
|--------|------------------------|----------|
| EC2 t2.micro | $8-10 | Small projects, learning |
| EC2 t3.small | $15-20 | Production, moderate traffic |
| Elastic Beanstalk | $15-25 | Easy deployment, auto-scaling |
| ECS Fargate | $12-30 | Containerized, scalable |
| Lambda | $0-5 | Sporadic usage, serverless |

---

## Monitoring & Logs

### CloudWatch Logs

```bash
# View logs
aws logs tail /aws/elasticbeanstalk/image-upload-env/var/log/eb-engine.log --follow
```

### Application Logs

Add to your Go code:
```go
import "log"

log.Println("Upload successful:", filename)
```

---

## Troubleshooting

**Service won't start:**
```bash
sudo journalctl -u image-upload -f
```

**Check if port is listening:**
```bash
sudo netstat -tulpn | grep 8080
```

**Test locally:**
```bash
curl -X POST http://localhost:8080/upload -F "image=@test.jpg"
```

---

## Security Best Practices

1. **Use AWS Secrets Manager** for credentials instead of environment variables
2. **Enable VPC** for ECS/Lambda
3. **Use IAM roles** instead of access keys when possible
4. **Enable CloudWatch monitoring**
5. **Set up AWS WAF** for DDoS protection
6. **Use HTTPS only** in production

---

## Support

For AWS-specific issues:
- AWS Documentation: https://docs.aws.amazon.com/
- AWS Support: https://console.aws.amazon.com/support/

For application issues:
- GitHub Issues: https://github.com/yourusername/image-upload-r2/issues
