# Image Upload Service for Cloudflare R2

A lightweight, production-ready Go service for uploading images to Cloudflare R2 storage with automatic file validation, UUID-based naming, and optional HTTPS support.

## Features

- ✅ Upload images to Cloudflare R2 (S3-compatible)
- ✅ API Key authentication for secure access
- ✅ Health check endpoint
- ✅ File type validation (JPEG, PNG, WebP)
- ✅ Automatic UUID-based unique filenames
- ✅ 10MB file size limit
- ✅ JSON API responses
- ✅ Optional HTTPS/TLS support
- ✅ Environment variable validation
- ✅ Works behind reverse proxies

## Prerequisites

- Go 1.25.1 or higher
- Cloudflare R2 account with:
  - Account ID
  - Access Key & Secret Key
  - Bucket created
  - Public URL configured (optional: R2 custom domain or public bucket URL)

## Installation

1. Clone the repository:
```bash
git clone https://github.com/yourusername/image-upload-r2.git
cd image-upload-r2
```

2. Install dependencies:
```bash
go mod download
```

3. Configure environment variables:
```bash
cp .env.example .env
```

Edit `.env` with your credentials:
```env
PORT=8080

# API Key for authentication (required)
API_KEY=your-secret-api-key-here

# Optional: For direct HTTPS support
# TLS_CERT_FILE=/path/to/cert.pem
# TLS_KEY_FILE=/path/to/key.pem

R2_ACCOUNT_ID=your_account_id
R2_ACCESS_KEY=your_access_key
R2_SECRET_KEY=your_secret_key
R2_BUCKET_NAME=your_bucket_name
R2_PUBLIC_URL=https://your-cdn-url.com
```

## Usage

### Run the server

```bash
go run main.go
```

Or build and run:
```bash
go build -o image-upload
./image-upload
```

### API Endpoints

#### Health Check

**GET** `/`

**Headers:**
- `X-API-Key`: Your API key (required)

**Success Response (200):**
```json
{
  "success": true,
  "message": "successfully connect"
}
```

**Error Response (401):**
```json
{
  "status": 401,
  "message": "Unauthorized: Invalid or missing API key"
}
```

#### Upload Image

**POST** `/upload`

**Headers:**
- `X-API-Key`: Your API key (required)

**Request:**
- Content-Type: `multipart/form-data`
- Field name: `image`
- Accepted formats: `.jpg`, `.jpeg`, `.png`, `.webp`
- Max size: 10MB

**Success Response (200):**
```json
{
  "status": 200,
  "url": "https://your-cdn-url.com/uploads/uuid-filename.jpg",
  "message": "Image uploaded successfully"
}
```

**Error Response (400/401/500):**
```json
{
  "status": 400,
  "url": null,
  "message": "Invalid image type. Only jpeg, png, webp allowed"
}
```

## Testing with cURL

**Health check:**
```bash
curl -H "X-API-Key: your-secret-api-key-here" http://localhost:8080/
```

**Upload image:**
```bash
curl -X POST http://localhost:8080/upload \
  -H "X-API-Key: your-secret-api-key-here" \
  -F "image=@/path/to/your/image.jpg"
```

## Testing with Postman

1. Create a new request (GET for `/` or POST for `/upload`)
2. Set URL: `http://localhost:8080/` or `http://localhost:8080/upload`
3. Go to Headers → Add `X-API-Key` with your API key value
4. For upload: Go to Body → form-data → Add key `image` (type: File) → Select image
5. Send request

## HTTPS Configuration

### Option 1: HTTP (Default)
Leave `TLS_CERT_FILE` and `TLS_KEY_FILE` empty. Suitable for:
- Development
- Running behind reverse proxy (nginx, Cloudflare, AWS ALB)

### Option 2: Direct HTTPS
Set certificate paths in `.env`:
```env
TLS_CERT_FILE=/path/to/cert.pem
TLS_KEY_FILE=/path/to/key.pem
```

Generate self-signed certificate for testing:
```bash
openssl req -x509 -newkey rsa:4096 -keyout key.pem -out cert.pem -days 365 -nodes
```

## Cloudflare R2 Setup

1. Go to [Cloudflare Dashboard](https://dash.cloudflare.com/) → R2
2. Create a bucket
3. Generate API tokens (Manage R2 API Tokens)
4. Configure public access:
   - Enable public bucket access, OR
   - Set up custom domain for R2 bucket

## Deployment

### Deploy to VPS (Ubuntu/Debian)

1. **Install Go:**
```bash
wget https://go.dev/dl/go1.25.1.linux-amd64.tar.gz
sudo tar -C /usr/local -xzf go1.25.1.linux-amd64.tar.gz
export PATH=$PATH:/usr/local/go/bin
```

2. **Clone and build:**
```bash
git clone <your-repo-url>
cd image-upload-r2
go build -o image-upload
```

3. **Configure environment:**
```bash
cp .env.example .env
nano .env  # Edit with your credentials
```

4. **Run with systemd:**

Create `/etc/systemd/system/image-upload.service`:
```ini
[Unit]
Description=Image Upload Service
After=network.target

[Service]
Type=simple
User=www-data
WorkingDirectory=/path/to/image-upload-r2
EnvironmentFile=/path/to/image-upload-r2/.env
ExecStart=/path/to/image-upload-r2/image-upload
Restart=always

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

### Deploy with Docker

Create `Dockerfile`:
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

Build and run:
```bash
docker build -t image-upload-r2 .
docker run -d -p 8080:8080 --env-file .env --name image-upload image-upload-r2
```

### Deploy with Docker Compose

Create `docker-compose.yml`:
```yaml
version: '3.8'
services:
  image-upload:
    build: .
    ports:
      - "8080:8080"
    env_file:
      - .env
    restart: unless-stopped
```

Run:
```bash
docker-compose up -d
```

### Behind Nginx Reverse Proxy

Create `/etc/nginx/sites-available/image-upload`:
```nginx
server {
    listen 80;
    server_name yourdomain.com;

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

Enable and reload:
```bash
sudo ln -s /etc/nginx/sites-available/image-upload /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl reload nginx
```

### SSL with Certbot (Let's Encrypt)

```bash
sudo apt install certbot python3-certbot-nginx
sudo certbot --nginx -d yourdomain.com
```

### Deploy to Railway

1. Install Railway CLI:
```bash
npm i -g @railway/cli
```

2. Login and deploy:
```bash
railway login
railway init
railway up
```

3. Add environment variables in Railway dashboard

### Deploy to Render

1. Create `render.yaml`:
```yaml
services:
  - type: web
    name: image-upload-r2
    env: go
    buildCommand: go build -o image-upload
    startCommand: ./image-upload
    envVars:
      - key: PORT
        value: 8080
      - key: API_KEY
        sync: false
      - key: R2_ACCOUNT_ID
        sync: false
      - key: R2_ACCESS_KEY
        sync: false
      - key: R2_SECRET_KEY
        sync: false
      - key: R2_BUCKET_NAME
        sync: false
      - key: R2_PUBLIC_URL
        sync: false
```

2. Push to GitHub and connect to Render

### Deploy to Fly.io

1. Install Fly CLI:
```bash
curl -L https://fly.io/install.sh | sh
```

2. Deploy:
```bash
fly launch
fly secrets set API_KEY=your-key
fly secrets set R2_ACCOUNT_ID=your-id
fly secrets set R2_ACCESS_KEY=your-key
fly secrets set R2_SECRET_KEY=your-secret
fly secrets set R2_BUCKET_NAME=your-bucket
fly secrets set R2_PUBLIC_URL=your-url
fly deploy
```

## Project Structure

```
.
├── main.go          # Main application code
├── go.mod           # Go module dependencies
├── go.sum           # Dependency checksums
├── .env             # Environment configuration (not committed)
└── README.md        # This file
```

## Environment Variables

| Variable | Required | Description |
|----------|----------|-------------|
| `PORT` | No | Server port (default: 8080) |
| `API_KEY` | Yes | API key for authentication |
| `TLS_CERT_FILE` | No | Path to TLS certificate for HTTPS |
| `TLS_KEY_FILE` | No | Path to TLS private key for HTTPS |
| `R2_ACCOUNT_ID` | Yes | Cloudflare account ID |
| `R2_ACCESS_KEY` | Yes | R2 access key |
| `R2_SECRET_KEY` | Yes | R2 secret key |
| `R2_BUCKET_NAME` | Yes | R2 bucket name |
| `R2_PUBLIC_URL` | Yes | Public URL for uploaded files |

## Security Considerations

- Never commit `.env` file to version control
- Use strong, randomly generated API keys (min 32 characters)
- Always use HTTPS in production
- Consider rate limiting for public deployments
- Validate file content (not just extension) for production use
- Set appropriate CORS headers if needed
- Use IAM roles instead of static credentials when possible
- Rotate API keys regularly
- Store API keys in secrets manager for production

## Contributing

Contributions are welcome! Please feel free to submit a Pull Request.

1. Fork the repository
2. Create your feature branch (`git checkout -b feature/amazing-feature`)
3. Commit your changes (`git commit -m 'Add amazing feature'`)
4. Push to the branch (`git push origin feature/amazing-feature`)
5. Open a Pull Request

## License

This project is licensed under the MIT License - see the [LICENSE](LICENSE) file for details.

## Support

If you encounter any issues or have questions:
- Open an issue on GitHub
- Check Cloudflare R2 documentation: https://developers.cloudflare.com/r2/

## Acknowledgments

- Built with [AWS SDK for Go v2](https://github.com/aws/aws-sdk-go-v2)
- Uses [Cloudflare R2](https://www.cloudflare.com/products/r2/) for storage
- UUID generation by [google/uuid](https://github.com/google/uuid)
