# Deploy to AWS

## Option 1: AWS EC2

### 1. Launch EC2 Instance
- AMI: Ubuntu 22.04 LTS
- Instance Type: t2.micro (free tier)
- Security Group: Allow ports 22, 80, 443

### 2. Connect and Setup
```bash
ssh -i your-key.pem ubuntu@your-ec2-ip

# Install Go
wget https://go.dev/dl/go1.25.1.linux-amd64.tar.gz
sudo tar -C /usr/local -xzf go1.25.1.linux-amd64.tar.gz
echo 'export PATH=$PATH:/usr/local/go/bin' >> ~/.bashrc
source ~/.bashrc

# Clone and build
git clone <your-repo-url>
cd image-upload-r2
go build -o image-upload
```

### 3. Configure Environment
```bash
nano .env
```
Add your credentials:
```env
PORT=8080
API_KEY=your-secret-key
R2_ACCOUNT_ID=your-account-id
R2_ACCESS_KEY=your-access-key
R2_SECRET_KEY=your-secret-key
R2_BUCKET_NAME=your-bucket
R2_PUBLIC_URL=https://your-cdn-url.com
```

### 4. Create Systemd Service
```bash
sudo nano /etc/systemd/system/image-upload.service
```

```ini
[Unit]
Description=Image Upload Service
After=network.target

[Service]
Type=simple
User=ubuntu
WorkingDirectory=/home/ubuntu/image-upload-r2
EnvironmentFile=/home/ubuntu/image-upload-r2/.env
ExecStart=/home/ubuntu/image-upload-r2/image-upload
Restart=always

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable image-upload
sudo systemctl start image-upload
sudo systemctl status image-upload
```

### 5. Setup Nginx
```bash
sudo apt update
sudo apt install nginx -y

sudo nano /etc/nginx/sites-available/image-upload
```

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
        client_max_body_size 10M;
    }
}
```

```bash
sudo ln -s /etc/nginx/sites-available/image-upload /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl restart nginx
```

### 6. SSL with Certbot
```bash
sudo apt install certbot python3-certbot-nginx -y
sudo certbot --nginx -d your-domain.com
```

---

## Option 2: AWS Elastic Beanstalk

### 1. Install EB CLI
```bash
pip install awsebcli
```

### 2. Initialize
```bash
eb init -p go image-upload-r2
```

### 3. Create Environment
```bash
eb create production
```

### 4. Set Environment Variables
```bash
eb setenv API_KEY=your-key \
  R2_ACCOUNT_ID=your-id \
  R2_ACCESS_KEY=your-access \
  R2_SECRET_KEY=your-secret \
  R2_BUCKET_NAME=your-bucket \
  R2_PUBLIC_URL=https://your-cdn.com
```

### 5. Deploy
```bash
eb deploy
```

---

## Option 3: AWS ECS (Docker)

### 1. Create Dockerfile
```dockerfile
FROM golang:1.25-alpine AS builder
WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN go build -o image-upload

FROM alpine:latest
RUN apk --no-cache add ca-certificates
WORKDIR /root/
COPY --from=builder /app/image-upload .
EXPOSE 8080
CMD ["./image-upload"]
```

### 2. Build and Push to ECR
```bash
aws ecr create-repository --repository-name image-upload-r2

aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin <account-id>.dkr.ecr.us-east-1.amazonaws.com

docker build -t image-upload-r2 .
docker tag image-upload-r2:latest <account-id>.dkr.ecr.us-east-1.amazonaws.com/image-upload-r2:latest
docker push <account-id>.dkr.ecr.us-east-1.amazonaws.com/image-upload-r2:latest
```

### 3. Create ECS Task Definition
```json
{
  "family": "image-upload-task",
  "containerDefinitions": [
    {
      "name": "image-upload",
      "image": "<account-id>.dkr.ecr.us-east-1.amazonaws.com/image-upload-r2:latest",
      "portMappings": [
        {
          "containerPort": 8080,
          "protocol": "tcp"
        }
      ],
      "environment": [
        {"name": "PORT", "value": "8080"},
        {"name": "API_KEY", "value": "your-key"},
        {"name": "R2_ACCOUNT_ID", "value": "your-id"},
        {"name": "R2_ACCESS_KEY", "value": "your-access"},
        {"name": "R2_SECRET_KEY", "value": "your-secret"},
        {"name": "R2_BUCKET_NAME", "value": "your-bucket"},
        {"name": "R2_PUBLIC_URL", "value": "https://your-cdn.com"}
      ]
    }
  ]
}
```

### 4. Create ECS Service
```bash
aws ecs create-cluster --cluster-name image-upload-cluster
aws ecs create-service --cluster image-upload-cluster --service-name image-upload-service --task-definition image-upload-task --desired-count 1
```

---

## Option 4: AWS Lambda (Serverless)

### 1. Install AWS SAM CLI
```bash
pip install aws-sam-cli
```

### 2. Create template.yaml
```yaml
AWSTemplateFormatVersion: '2010-09-09'
Transform: AWS::Serverless-2016-10-31

Globals:
  Function:
    Timeout: 30
    Environment:
      Variables:
        API_KEY: your-key
        R2_ACCOUNT_ID: your-id
        R2_ACCESS_KEY: your-access
        R2_SECRET_KEY: your-secret
        R2_BUCKET_NAME: your-bucket
        R2_PUBLIC_URL: https://your-cdn.com

Resources:
  ImageUploadFunction:
    Type: AWS::Serverless::Function
    Properties:
      CodeUri: .
      Handler: bootstrap
      Runtime: provided.al2
      Events:
        HealthCheck:
          Type: Api
          Properties:
            Path: /
            Method: get
        Upload:
          Type: Api
          Properties:
            Path: /upload
            Method: post
```

### 3. Build and Deploy
```bash
GOOS=linux GOARCH=amd64 go build -o bootstrap main.go
sam build
sam deploy --guided
```

---

## Testing

```bash
# Health check
curl -H "X-API-Key: your-key" https://your-domain.com/

# Upload
curl -X POST https://your-domain.com/upload \
  -H "X-API-Key: your-key" \
  -F "image=@test.jpg"
```

---

## Cost Estimation

- **EC2 t2.micro**: ~$8-10/month (free tier: 750 hours/month)
- **Elastic Beanstalk**: ~$10-15/month
- **ECS Fargate**: ~$15-30/month
- **Lambda**: Pay per request (~$0.20 per 1M requests)

---

## Security Best Practices

- Use AWS Secrets Manager for API keys
- Enable CloudWatch logging
- Use IAM roles instead of access keys
- Enable AWS WAF for DDoS protection
- Use Application Load Balancer with SSL
- Enable VPC for network isolation
