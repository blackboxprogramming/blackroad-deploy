# BlackRoad Deploy

**Self-hosted deployment platform** - Your own Railway/Heroku alternative.

Deploy apps to your infrastructure (DigitalOcean + Raspberry Pi) with Docker, Cloudflare Tunnel, and full control.

## Why BlackRoad Deploy?

✅ **Full Control** - Own your infrastructure, no vendor lock-in
✅ **Cost Effective** - ~$12/month vs $20-100+ on Railway
✅ **Unlimited Projects** - No artificial limits
✅ **Free Custom Domains** - Via Cloudflare
✅ **Simple CLI** - `blackroad deploy` and you're live
✅ **Works Anywhere** - Cloudflare Tunnel works behind NAT/firewall

## Architecture

```
┌─────────────────────────────────────────────────┐
│  Developer                                      │
│  ├─ blackroad deploy                            │
│  └─ Pushes Docker image to ghcr.io             │
└────────────┬────────────────────────────────────┘
             │
             ▼
┌─────────────────────────────────────────────────┐
│  Cloudflare Tunnel                              │
│  ├─ deploy-api.blackroad.systems (API)         │
│  └─ *.blackroad.systems (Apps)                 │
└────────────┬────────────────────────────────────┘
             │
             ▼
┌─────────────────────────────────────────────────┐
│  Deployment Server (DigitalOcean/Pi)           │
│  ├─ API: Node.js + PostgreSQL                  │
│  ├─ Docker: Runs containers                    │
│  └─ Caddy: Reverse proxy                       │
└─────────────────────────────────────────────────┘
```

## Quick Start

### 1. Install CLI

```bash
npm install -g blackroad-cli
# or
yarn global add blackroad-cli
```

### 2. Login

```bash
blackroad login
```

### 3. Initialize Project

```bash
cd my-app
blackroad init
```

This creates `blackroad.json`:

```json
{
  "name": "my-app",
  "buildCommand": "npm run build",
  "startCommand": "npm start",
  "port": 3000,
  "env": {}
}
```

### 4. Deploy

```bash
blackroad deploy
```

Your app will be live at `https://my-app.blackroad.systems`

## CLI Commands

```bash
# Initialize project
blackroad init

# Login
blackroad login

# Deploy
blackroad deploy
blackroad deploy -n my-app -i ghcr.io/user/image:tag

# List deployments
blackroad list
blackroad ls

# View logs
blackroad logs my-app
blackroad logs my-app -f  # Follow logs
blackroad logs my-app -n 500  # Last 500 lines

# Environment variables
blackroad env set my-app DATABASE_URL=postgres://...
blackroad env get my-app
blackroad env delete my-app DATABASE_URL

# Restart deployment
blackroad restart my-app

# Delete deployment
blackroad delete my-app
blackroad rm my-app -f  # Skip confirmation
```

## Infrastructure Setup

### Prerequisites

- DigitalOcean droplet (or any VPS)
- Cloudflare account with domain
- Docker installed on server

### 1. Set Up Deployment Server

```bash
# SSH to your server
ssh root@159.65.43.12

# Clone repo
git clone https://github.com/blackroad-os/blackroad-deploy
cd blackroad-deploy

# Install dependencies
cd deployment-api
npm install

# Set up PostgreSQL
sudo apt-get install postgresql
sudo -u postgres createdb blackroad_deploy

# Configure environment
cp .env.example .env
nano .env  # Add your credentials

# Run migrations
npm run db:migrate

# Start API
npm start
```

### 2. Set Up Cloudflare Tunnel

```bash
# Run setup script
cd ../scripts
chmod +x setup-tunnel.sh
./setup-tunnel.sh
```

This will:
- Install cloudflared
- Create tunnel
- Configure DNS
- Start tunnel service

See [TUNNEL_SETUP.md](./TUNNEL_SETUP.md) for detailed instructions.

### 3. Set Up Raspberry Pi (Optional)

```bash
# SSH to Pi
ssh alice@192.168.4.49

# Copy and run setup script
scp scripts/setup-pi-tunnel.sh alice@192.168.4.49:~/
ssh alice@192.168.4.49
chmod +x setup-pi-tunnel.sh
./setup-pi-tunnel.sh
```

## Configuration

### Deployment API (.env)

```bash
DB_HOST=localhost
DB_PORT=5432
DB_NAME=blackroad_deploy
DB_USER=postgres
DB_PASSWORD=your_password

PORT=3000

CF_API_TOKEN=your_cloudflare_token
CF_ZONE_ID=your_zone_id

DOCKER_HOST=unix:///var/run/docker.sock
```

### Cloudflare Tunnel (config.yml)

```yaml
tunnel: <TUNNEL-ID>
credentials-file: /root/.cloudflared/<TUNNEL-ID>.json

ingress:
  - hostname: deploy-api.blackroad.systems
    service: http://localhost:3000

  - hostname: "*.blackroad.systems"
    service: http://localhost:8080

  - service: http_status:404
```

## API Reference

### Authentication

All API endpoints (except `/health` and `/api/auth/*`) require authentication:

```
Authorization: Bearer <API_KEY>
```

### Endpoints

#### `POST /api/auth/register`
Register new user and get API key.

#### `POST /api/auth/login`
Login and get API key.

#### `GET /api/deployments`
List all deployments.

#### `POST /api/deployments`
Create new deployment.

```json
{
  "name": "my-app",
  "image": "ghcr.io/user/my-app:latest",
  "env": {
    "DATABASE_URL": "postgres://..."
  },
  "port": 3000
}
```

#### `GET /api/deployments/:name`
Get deployment details.

#### `DELETE /api/deployments/:name`
Delete deployment.

#### `POST /api/deployments/:name/restart`
Restart deployment.

#### `GET /api/logs/:name`
Get container logs.

#### `GET /api/env/:name`
Get environment variables.

#### `POST /api/env/:name`
Set environment variable.

```json
{
  "key": "DATABASE_URL",
  "value": "postgres://..."
}
```

## Docker Images

Your apps should be containerized. Example `Dockerfile`:

```dockerfile
FROM node:20-alpine

WORKDIR /app

COPY package*.json ./
RUN npm ci --only=production

COPY . .
RUN npm run build

EXPOSE 3000

CMD ["npm", "start"]
```

Build and push to GitHub Container Registry:

```bash
docker build -t ghcr.io/your-username/my-app:latest .
docker push ghcr.io/your-username/my-app:latest
```

Or let `blackroad deploy` handle it automatically.

## Cost Breakdown

### DigitalOcean Droplet
- **Basic Droplet**: $6/month (1GB RAM, 1 vCPU)
- **Better**: $12/month (2GB RAM, 2 vCPU)
- **Production**: $24/month (4GB RAM, 2 vCPU)

### Cloudflare
- **Free**: Tunnel, DNS, SSL, DDoS protection
- **Optional**: Additional features

### Total: ~$6-24/month
vs Railway: $20-100+/month (limited projects)

## Raspberry Pi Support

Run deployments on Raspberry Pi for even lower costs:

```bash
# Run setup on Pi
./scripts/setup-pi-tunnel.sh

# Deploy to Pi
blackroad deploy -n pi-app
```

Your Pi can host:
- Development/staging environments
- Personal projects
- Internal tools
- IoT services

## Troubleshooting

### Tunnel Not Working

```bash
# Check tunnel status
cloudflared tunnel info blackroad-deploy

# Check service
systemctl status cloudflared

# View logs
journalctl -u cloudflared -f
```

### Docker Issues

```bash
# Check Docker
docker ps
docker logs <container-id>

# Restart Docker
systemctl restart docker
```

### Database Connection

```bash
# Test PostgreSQL
psql -U postgres -d blackroad_deploy

# Check running containers
docker ps | grep postgres
```

## Security

- ✅ API key authentication
- ✅ Cloudflare Tunnel (no open ports)
- ✅ HTTPS everywhere
- ✅ Environment variables encrypted at rest
- ✅ Container isolation
- ✅ DDoS protection via Cloudflare

## Roadmap

- [ ] Web dashboard (Cloudflare Pages)
- [ ] GitHub Actions integration
- [ ] Auto-scaling
- [ ] Multi-region deployments
- [ ] Database provisioning
- [ ] Metrics and monitoring
- [ ] Automatic SSL for custom domains
- [ ] Deploy from Git (auto-deploy on push)

## Contributing

Pull requests welcome! See [ARCHITECTURE.md](./ARCHITECTURE.md) for technical details.

## License

MIT

## Support

- GitHub Issues: https://github.com/blackroad-os/blackroad-deploy/issues
- Email: blackroad.systems@gmail.com

---

Built with ❤️ by BlackRoad Systems
