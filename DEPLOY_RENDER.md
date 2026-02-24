# Deploy to Render

## Quick Deploy (Web Dashboard)

### 1. Prepare Repository
Push your code to GitHub/GitLab

### 2. Create Web Service
1. Go to [Render Dashboard](https://dashboard.render.com/)
2. Click **New +** → **Web Service**
3. Connect your repository
4. Configure:
   - **Name**: `image-upload-r2`
   - **Environment**: `Go`
   - **Build Command**: `go build -o image-upload`
   - **Start Command**: `./image-upload`
   - **Instance Type**: Free or Starter ($7/month)

### 3. Add Environment Variables
Click **Environment** → Add:
```
PORT=8080
API_KEY=your-secret-api-key-here
R2_ACCOUNT_ID=your-account-id
R2_ACCESS_KEY=your-access-key
R2_SECRET_KEY=your-secret-key
R2_BUCKET_NAME=your-bucket-name
R2_PUBLIC_URL=https://your-cdn-url.com
```

### 4. Deploy
Click **Create Web Service** → Render will auto-deploy

---

## Deploy with render.yaml (Infrastructure as Code)

### 1. Create render.yaml
```yaml
services:
  - type: web
    name: image-upload-r2
    env: go
    buildCommand: go build -o image-upload
    startCommand: ./image-upload
    plan: free  # or starter
    healthCheckPath: /
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

### 2. Push to Repository
```bash
git add render.yaml
git commit -m "Add Render config"
git push
```

### 3. Create Blueprint
1. Go to Render Dashboard
2. Click **New +** → **Blueprint**
3. Connect repository
4. Render will detect `render.yaml` and create service

### 4. Set Environment Variables
Go to service → **Environment** → Add all required variables

---

## Deploy with Render CLI

### 1. Install Render CLI
```bash
# macOS
brew install render

# Linux/Windows
curl -fsSL https://render.com/install.sh | bash
```

### 2. Login
```bash
render login
```

### 3. Create Service
```bash
render services create
```

### 4. Set Environment Variables
```bash
render env set API_KEY=your-key
render env set R2_ACCOUNT_ID=your-id
render env set R2_ACCESS_KEY=your-access
render env set R2_SECRET_KEY=your-secret
render env set R2_BUCKET_NAME=your-bucket
render env set R2_PUBLIC_URL=https://your-cdn.com
```

### 5. Deploy
```bash
render deploy
```

---

## Custom Domain Setup

### 1. Add Custom Domain
1. Go to service → **Settings** → **Custom Domains**
2. Click **Add Custom Domain**
3. Enter your domain: `api.yourdomain.com`

### 2. Configure DNS
Add CNAME record in your DNS provider:
```
Type: CNAME
Name: api
Value: your-app.onrender.com
```

### 3. SSL Certificate
Render automatically provisions SSL certificate (Let's Encrypt)

---

## Environment-Specific Deployments

### Production
```yaml
services:
  - type: web
    name: image-upload-prod
    env: go
    buildCommand: go build -o image-upload
    startCommand: ./image-upload
    plan: starter
    envVars:
      - key: API_KEY
        sync: false
```

### Staging
```yaml
services:
  - type: web
    name: image-upload-staging
    env: go
    buildCommand: go build -o image-upload
    startCommand: ./image-upload
    plan: free
    envVars:
      - key: API_KEY
        sync: false
```

---

## Auto-Deploy on Git Push

### 1. Enable Auto-Deploy
1. Go to service → **Settings**
2. Enable **Auto-Deploy**
3. Select branch: `main` or `production`

### 2. Deploy on Push
```bash
git add .
git commit -m "Update code"
git push origin main
```
Render automatically deploys changes

---

## Health Checks

Render automatically monitors your service using the health check endpoint:
```yaml
healthCheckPath: /
```

Your app must respond with 200 status code at `GET /` with valid API key.

---

## Monitoring & Logs

### View Logs
```bash
render logs
```

Or in dashboard: Service → **Logs**

### Metrics
Dashboard shows:
- CPU usage
- Memory usage
- Request count
- Response times

---

## Scaling

### Vertical Scaling
1. Go to service → **Settings**
2. Change **Instance Type**:
   - Free: 512MB RAM, 0.1 CPU
   - Starter: 512MB RAM, 0.5 CPU ($7/month)
   - Standard: 2GB RAM, 1 CPU ($25/month)

### Horizontal Scaling
Upgrade to paid plan for multiple instances

---

## Troubleshooting

### Build Fails
Check Go version in `go.mod`:
```go
go 1.25
```

### Service Won't Start
Check logs:
```bash
render logs --tail
```

Verify environment variables are set

### 502 Bad Gateway
- Ensure app listens on `0.0.0.0:$PORT`
- Check health check endpoint returns 200

### API Key Issues
Test locally:
```bash
curl -H "X-API-Key: your-key" https://your-app.onrender.com/
```

---

## Pricing

| Plan | Price | RAM | CPU | Features |
|------|-------|-----|-----|----------|
| Free | $0 | 512MB | 0.1 | Sleeps after 15min inactivity |
| Starter | $7/mo | 512MB | 0.5 | Always on |
| Standard | $25/mo | 2GB | 1.0 | Always on, more resources |

---

## Testing Deployment

```bash
# Get your Render URL
RENDER_URL="https://your-app.onrender.com"

# Health check
curl -H "X-API-Key: your-key" $RENDER_URL/

# Upload test
curl -X POST $RENDER_URL/upload \
  -H "X-API-Key: your-key" \
  -F "image=@test.jpg"
```

---

## CI/CD with GitHub Actions

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
      
      - name: Trigger Render Deploy
        run: |
          curl -X POST ${{ secrets.RENDER_DEPLOY_HOOK }}
```

Add deploy hook in Render: Settings → Deploy Hook

---

## Best Practices

- Use **Starter plan** for production (always on)
- Enable **Auto-Deploy** for main branch
- Use **Environment Groups** for shared variables
- Set up **Custom Domain** with SSL
- Monitor logs regularly
- Use **Secrets** for sensitive data
- Enable **Health Checks**
- Set appropriate **Instance Type** based on traffic

---

## Support

- [Render Documentation](https://render.com/docs)
- [Render Community](https://community.render.com/)
- [Status Page](https://status.render.com/)
