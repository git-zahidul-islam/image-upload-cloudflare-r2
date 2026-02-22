# Deploy to Render - Image Upload Service

Complete guide to deploy the Image Upload Service on Render.com with automatic deployments from GitHub.

## Table of Contents
- [Quick Deploy (Web Service)](#quick-deploy-web-service)
- [Docker Deployment](#docker-deployment)
- [Environment Variables Setup](#environment-variables-setup)
- [Custom Domain](#custom-domain)
- [Troubleshooting](#troubleshooting)

---

## Quick Deploy (Web Service)

### Prerequisites
- GitHub account with your repository
- Render account (free tier available)
- Cloudflare R2 credentials

### Step 1: Push to GitHub

```bash
git add .
git commit -m "Ready for Render deployment"
git push origin main
```

### Step 2: Create Render Web Service

1. Go to [Render Dashboard](https://dashboard.render.com/)
2. Click **"New +"** → **"Web Service"**
3. Connect your GitHub repository
4. Select `image-upload-r2` repository

### Step 3: Configure Service

**Basic Settings:**
- **Name:** `image-upload-service`
- **Region:** Choose closest to your users
- **Branch:** `main`
- **Root Directory:** Leave empty
- **Runtime:** `Go`
- **Build Command:** `go build -o app main.go`
- **Start Command:** `./app`

**Instance Type:**
- Free tier: `Free` (512 MB RAM, sleeps after inactivity)
- Production: `Starter` ($7/month, always on)

### Step 4: Add Environment Variables

Click **"Advanced"** → **"Add Environment Variable"**

Add these variables:

| Key | Value |
|-----|-------|
| `PORT` | `10000` (Render default) |
| `R2_ACCOUNT_ID` | Your Cloudflare account ID |
| `R2_ACCESS_KEY` | Your R2 access key |
| `R2_SECRET_KEY` | Your R2 secret key |
| `R2_BUCKET_NAME` | Your bucket name |
| `R2_PUBLIC_URL` | Your R2 public URL |

### Step 5: Deploy

1. Click **"Create Web Service"**
2. Render will automatically:
   - Clone your repository
   - Install Go dependencies
   - Build your application
   - Deploy and start the service

### Step 6: Get Your URL

After deployment completes:
- Your service URL: `https://image-upload-service.onrender.com`
- Test endpoint: `https://image-upload-service.onrender.com/upload`

---

## Docker Deployment

### Step 1: Create Dockerfile

Create `Dockerfile` in your project root:

```dockerfile
FROM golang:1.25-alpine AS builder

WORKDIR /app

# Copy dependency files
COPY go.mod go.sum ./
RUN go mod download

# Copy source code
COPY . .

# Build application
RUN go build -o image-upload main.go

# Final stage
FROM alpine:latest

RUN apk --no-cache add ca-certificates

WORKDIR /root/

COPY --from=builder /app/image-upload .

EXPOSE 10000

CMD ["./image-upload"]
```

### Step 2: Create render.yaml

Create `render.yaml` for infrastructure as code:

```yaml
services:
  - type: web
    name: image-upload-service
    runtime: docker
    repo: https://github.com/yourusername/image-upload-r2
    region: oregon
    plan: free
    branch: main
    dockerfilePath: ./Dockerfile
    envVars:
      - key: PORT
        value: 10000
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

### Step 3: Deploy from Dashboard

1. Go to Render Dashboard
2. Click **"New +"** → **"Web Service"**
3. Connect repository
4. Render will detect `Dockerfile` automatically
5. Add environment variables
6. Click **"Create Web Service"**

---

## Environment Variables Setup

### Using Render Dashboard

1. Go to your service → **"Environment"**
2. Click **"Add Environment Variable"**
3. Add each variable individually

### Using render.yaml (Recommended)

```yaml
envVars:
  - key: PORT
    value: 10000
  - key: R2_ACCOUNT_ID
    value: your_account_id
  - key: R2_ACCESS_KEY
    value: your_access_key
  - key: R2_SECRET_KEY
    value: your_secret_key
  - key: R2_BUCKET_NAME
    value: your_bucket_name
  - key: R2_PUBLIC_URL
    value: https://your-cdn-url.com
```

### Using Render CLI

```bash
# Install Render CLI
npm install -g @render/cli

# Login
render login

# Set environment variables
render env set R2_ACCOUNT_ID=your_account_id
render env set R2_ACCESS_KEY=your_access_key
render env set R2_SECRET_KEY=your_secret_key
render env set R2_BUCKET_NAME=your_bucket_name
render env set R2_PUBLIC_URL=https://your-cdn-url.com
```

---

## Custom Domain

### Step 1: Add Custom Domain

1. Go to your service → **"Settings"**
2. Scroll to **"Custom Domain"**
3. Click **"Add Custom Domain"**
4. Enter your domain: `api.yourdomain.com`

### Step 2: Configure DNS

Add CNAME record in your DNS provider:

| Type | Name | Value |
|------|------|-------|
| CNAME | api | your-service.onrender.com |

### Step 3: Enable HTTPS

Render automatically provisions SSL certificates via Let's Encrypt.
Wait 5-10 minutes for certificate issuance.

---

## Auto-Deploy from GitHub

### Enable Auto-Deploy

1. Go to service → **"Settings"**
2. Under **"Build & Deploy"**
3. Enable **"Auto-Deploy"** (enabled by default)

Now every push to `main` branch triggers automatic deployment.

### Deploy Specific Branch

```yaml
# In render.yaml
services:
  - type: web
    name: image-upload-service
    branch: production  # Deploy from production branch
```

### Manual Deploy

1. Go to service → **"Manual Deploy"**
2. Click **"Deploy latest commit"**
3. Or select specific commit from dropdown

---

## Health Checks

### Add Health Check Endpoint

Update `main.go`:

```go
func main() {
    // ... existing code ...
    
    http.HandleFunc("/upload", uploadHandler)
    http.HandleFunc("/health", healthHandler)  // Add this
    
    // ... rest of code ...
}

func healthHandler(w http.ResponseWriter, r *http.Request) {
    w.WriteHeader(http.StatusOK)
    w.Write([]byte("OK"))
}
```

### Configure in Render

1. Go to service → **"Settings"**
2. Under **"Health Check"**
3. Set **Health Check Path:** `/health`

---

## Monitoring & Logs

### View Logs

**From Dashboard:**
1. Go to your service
2. Click **"Logs"** tab
3. View real-time logs

**From CLI:**
```bash
render logs -f
```

### Metrics

Render provides:
- CPU usage
- Memory usage
- Request count
- Response times

Access via **"Metrics"** tab in dashboard.

---

## Scaling

### Vertical Scaling (Upgrade Plan)

| Plan | RAM | CPU | Price |
|------|-----|-----|-------|
| Free | 512 MB | 0.1 CPU | $0 |
| Starter | 512 MB | 0.5 CPU | $7/month |
| Standard | 2 GB | 1 CPU | $25/month |
| Pro | 4 GB | 2 CPU | $85/month |

### Horizontal Scaling

Render doesn't support horizontal scaling on free/starter plans.
Use Standard+ plans for multiple instances.

---

## Troubleshooting

### Service Won't Start

**Check logs:**
```bash
render logs
```

**Common issues:**
- Missing environment variables
- Wrong PORT (must be 10000 on Render)
- Build command incorrect

### Build Fails

**Check build command:**
```bash
go build -o app main.go
```

**Verify go.mod:**
```bash
go mod tidy
git add go.mod go.sum
git commit -m "Update dependencies"
git push
```

### Service Sleeps (Free Tier)

Free tier services sleep after 15 minutes of inactivity.

**Solutions:**
1. Upgrade to Starter plan ($7/month)
2. Use external ping service (UptimeRobot, Pingdom)
3. Accept cold starts (first request takes 30-60 seconds)

### Upload Fails

**Check environment variables:**
1. Go to service → **"Environment"**
2. Verify all R2 credentials are set
3. Click **"Save Changes"** if modified

**Test locally:**
```bash
curl -X POST https://your-service.onrender.com/upload \
  -F "image=@test.jpg"
```

### Port Issues

Render requires port `10000`. Update `.env`:
```env
PORT=10000
```

Or set in Render dashboard.

---

## Cost Optimization

### Free Tier Limits
- 750 hours/month (enough for 1 service)
- Sleeps after 15 min inactivity
- 100 GB bandwidth/month

### Tips
1. Use free tier for development/testing
2. Upgrade to Starter for production ($7/month)
3. Use Cloudflare R2 (cheaper than S3)
4. Enable caching with Cloudflare CDN

---

## CI/CD Pipeline

### GitHub Actions Integration

Create `.github/workflows/deploy.yml`:

```yaml
name: Deploy to Render

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Set up Go
        uses: actions/setup-go@v4
        with:
          go-version: '1.25'
      
      - name: Run tests
        run: go test ./...
      
      - name: Trigger Render Deploy
        run: |
          curl -X POST https://api.render.com/deploy/srv-xxxxx?key=your-deploy-key
```

Get deploy hook:
1. Service → **"Settings"** → **"Deploy Hook"**
2. Copy URL and add to GitHub Actions

---

## Backup & Rollback

### Rollback to Previous Version

1. Go to service → **"Events"**
2. Find successful deployment
3. Click **"Rollback"**

### Manual Rollback via Git

```bash
git revert HEAD
git push origin main
```

Render will auto-deploy the reverted version.

---

## Security Best Practices

1. **Use Environment Variables** - Never commit credentials
2. **Enable HTTPS** - Automatic with custom domains
3. **Rotate Keys** - Change R2 credentials periodically
4. **Monitor Logs** - Check for suspicious activity
5. **Rate Limiting** - Add middleware for production

---

## Comparison: Render vs Others

| Feature | Render | Heroku | AWS | Vercel |
|---------|--------|--------|-----|--------|
| Free Tier | ✅ 750h | ❌ Paid only | ✅ Limited | ❌ No Go support |
| Auto-deploy | ✅ | ✅ | ❌ Manual | ✅ |
| Custom Domain | ✅ Free SSL | ✅ Paid SSL | ✅ Manual | ✅ |
| Docker Support | ✅ | ✅ | ✅ | ❌ |
| Pricing | $7/month | $7/month | $15+/month | N/A |

---

## Support

**Render Documentation:**
- https://render.com/docs

**Community:**
- https://community.render.com

**Status Page:**
- https://status.render.com

**Application Issues:**
- GitHub: https://github.com/yourusername/image-upload-r2/issues

---

## Quick Reference

**Deploy Command:**
```bash
git push origin main
```

**View Logs:**
```bash
render logs -f
```

**Test Endpoint:**
```bash
curl -X POST https://your-service.onrender.com/upload \
  -F "image=@test.jpg"
```

**Service URL Format:**
```
https://[service-name].onrender.com
```

---

## Next Steps

1. ✅ Deploy to Render
2. ✅ Add custom domain
3. ✅ Set up monitoring
4. ✅ Configure auto-deploy
5. ✅ Test upload functionality
6. ✅ Upgrade to paid plan for production

Happy deploying! 🚀
