# Image Upload Service for Cloudflare R2

A lightweight, production-ready Go service for uploading images to Cloudflare R2 storage with automatic file validation, UUID-based naming, and optional HTTPS support.

## Features

- ✅ Upload images to Cloudflare R2 (S3-compatible)
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

### API Endpoint

**POST** `/upload`

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

**Error Response (400/500):**
```json
{
  "status": 400,
  "url": null,
  "message": "Invalid image type. Only jpeg, png, webp allowed"
}
```

## Testing with cURL

```bash
curl -X POST http://localhost:8080/upload \
  -F "image=@/path/to/your/image.jpg"
```

## Testing with Postman

1. Create a new POST request to `http://localhost:8080/upload`
2. Go to Body → form-data
3. Add key `image`, change type to `File`
4. Select an image file
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

### Docker (Optional)

Create `Dockerfile`:
```dockerfile
FROM golang:1.25-alpine
WORKDIR /app
COPY . .
RUN go build -o server
EXPOSE 8080
CMD ["./server"]
```

Build and run:
```bash
docker build -t image-upload-r2 .
docker run -p 8080:8080 --env-file .env image-upload-r2
```

### Behind Nginx

```nginx
server {
    listen 443 ssl;
    server_name yourdomain.com;

    ssl_certificate /path/to/cert.pem;
    ssl_certificate_key /path/to/key.pem;

    location / {
        proxy_pass http://localhost:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
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
| `TLS_CERT_FILE` | No | Path to TLS certificate for HTTPS |
| `TLS_KEY_FILE` | No | Path to TLS private key for HTTPS |
| `R2_ACCOUNT_ID` | Yes | Cloudflare account ID |
| `R2_ACCESS_KEY` | Yes | R2 access key |
| `R2_SECRET_KEY` | Yes | R2 secret key |
| `R2_BUCKET_NAME` | Yes | R2 bucket name |
| `R2_PUBLIC_URL` | Yes | Public URL for uploaded files |

## Security Considerations

- Never commit `.env` file to version control
- Use HTTPS in production
- Consider rate limiting for public deployments
- Validate file content (not just extension) for production use
- Set appropriate CORS headers if needed
- Use IAM roles instead of static credentials when possible

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
