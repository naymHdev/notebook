# Reusable Monorepo Deployment Guide

এই guide অনুসরণ করে একটি website, admin dashboard এবং backend-কে একই Git repository থেকে Docker, GitHub Actions, nginx, HTTPS এবং VPS-এ selectively deploy করা যাবে। এটি কোনো নির্দিষ্ট project-এর secret বা credential ধারণ করে না।

## 1. Architecture

একটি সাধারণ repository structure:

```text
my-project/
├── .github/
│   └── workflows/
│       └── ci-cd.yml
├── website/
│   ├── Dockerfile
│   ├── .dockerignore
│   ├── .env.example
│   └── ...
├── admin-dashboard/
│   ├── Dockerfile
│   ├── .dockerignore
│   ├── .env.example
│   └── ...
├── backend/
│   ├── Dockerfile
│   ├── docker-entrypoint.sh
│   ├── .dockerignore
│   ├── .env.example
│   └── ...
├── deploy/
│   ├── nginx/
│   │   ├── default.conf
│   │   └── bootstrap.conf
│   └── scripts/
│       ├── backup-vps.sh
│       ├── bootstrap-vps.sh
│       └── deploy.sh
├── compose.yaml
├── .gitattributes
├── .gitignore
└── README.md
```

Production flow:

```text
Git push
   │
   ├── website changed ─────────► build website image ─────► deploy website only
   ├── admin-dashboard changed ─► build dashboard image ───► deploy dashboard only
   ├── backend changed ─────────► build backend image ─────► deploy backend only
   └── deploy/compose changed ──► reconcile infrastructure

GitHub Container Registry ─► VPS Docker Compose ─► nginx ─► public domains
```

## 2. Values to decide before starting

নতুন project-এর জন্য প্রথমে এই placeholders লিখে রাখুন:

| Placeholder | Example |
| --- | --- |
| `GITHUB_OWNER` | `your-github-username` |
| `REPOSITORY` | `my-project` |
| `APP_NAME` | `my-project` |
| `VPS_USER` | `deploy` |
| `VPS_HOST` | `203.0.113.10` |
| `VPS_PORT` | `22` |
| `APP_DIR` | `/home/deploy/apps/my-project` |
| `WEBSITE_DOMAIN` | `example.com` |
| `DASHBOARD_DOMAIN` | `dashboard.example.com` |
| `API_DOMAIN` | `api.example.com` |
| `DATABASE_NAME` | `my_project` |

FinderQ-এর files copy করলে প্রতিটি hardcoded `finderq`, domain এবং GHCR image path নতুন values দিয়ে replace করতে হবে।

## 3. Prerequisites

Local machine:

- Git
- Node.js/Bun/npm—project অনুযায়ী
- SSH client
- Docker Desktop optional, কিন্তু local image test-এর জন্য recommended

VPS:

- Ubuntu LTS
- Docker Engine
- Docker Compose plugin
- SSH access
- Ports `80` এবং `443` open
- পর্যাপ্ত disk space

Verify:

```bash
docker version
docker compose version
df -h /
free -h
```

DNS-এ website, dashboard এবং API domains-এর `A` records VPS IPv4 address-এ point করতে হবে। Certificate request-এর আগে verify করুন:

```bash
getent ahostsv4 example.com
getent ahostsv4 dashboard.example.com
getent ahostsv4 api.example.com
```

## 4. Git and secret safety

Root `.gitignore`-এ minimum rules:

```gitignore
**/node_modules/
**/.next/
**/dist/
**/coverage/
**/*.tsbuildinfo

.env
.env.*
!.env.example
**/.env
**/.env.*
!**/.env.example

*.log
.idea/
.vscode/
.DS_Store
Thumbs.db
```

Linux scripts যেন Windows থেকেও LF line endings পায়:

```gitattributes
* text=auto
*.sh text eol=lf
*.yml text eol=lf
*.yaml text eol=lf
Dockerfile text eol=lf
```

Commit-এর আগে verify করুন:

```powershell
git check-ignore -v backend/.env website/.env.local
git status --short
```

কখনো commit করবেন না:

- `.env` বা `.env.local`
- SSH private key
- GitHub PAT
- database dump
- cloud provider secret/client secret

## 5. Environment strategy

Environment তিন ভাগে ভাগ করুন।

### Browser-visible build variables

Next.js-এর `NEXT_PUBLIC_*` values image build-এর সময় bundle-এর মধ্যে যায়। এগুলো GitHub **Repository variables** হিসেবে রাখুন:

```text
NEXT_PUBLIC_API_BASE_URL
NEXT_PUBLIC_SOCKET_URL
NEXT_PUBLIC_GOOGLE_CLIENT_ID
NEXT_PUBLIC_PAYPAL_CLIENT_ID
NEXT_PUBLIC_STREAM_API_KEY
CD_ENABLED
```

Path:

```text
Repository Settings
→ Secrets and variables
→ Actions
→ Variables
→ New repository variable
```

এগুলো `production` Environment variables হিসেবে রাখলে environment ব্যবহার করে না এমন build jobs values পাবে না। `NEXT_PUBLIC_*`-এ private secret রাখবেন না।

### GitHub deployment secrets

`production` environment তৈরি করে সেখানে রাখুন:

```text
VPS_HOST
VPS_PORT
VPS_USER
VPS_SSH_KEY
VPS_KNOWN_HOSTS
```

Environment name workflow-এর সঙ্গে exact match করতে হবে:

```yaml
environment: production
```

Production environment-এর deployment branch শুধু `main` allow করা recommended।

### Backend runtime secrets

Backend secrets GitHub repository-তে নয়, VPS-এর private file-এ থাকবে:

```text
/home/deploy/apps/my-project/.env
```

Example:

```env
POSTGRES_PASSWORD=generate-a-long-random-value
SERVER_URL=https://api.example.com
CLIENT_URL=https://example.com
DASHBOARD_URL=https://dashboard.example.com

JWT_ACCESS_SECRET=replace-me
JWT_REFRESH_SECRET=replace-me
JWT_PENDING_SECRET=replace-me

WEBSITE_IMAGE=ghcr.io/github-owner/my-project-website:latest
DASHBOARD_IMAGE=ghcr.io/github-owner/my-project-dashboard:latest
BACKEND_IMAGE=ghcr.io/github-owner/my-project-backend:latest

LETSENCRYPT_EMAIL=admin@example.com
```

Protect it:

```bash
chmod 600 .env
```

## 6. Dockerfiles

প্রতিটি application-এর আলাদা Dockerfile থাকবে। Frontend public environment values build arguments হিসেবে receive করবে:

```dockerfile
ARG NEXT_PUBLIC_API_BASE_URL
ARG NEXT_PUBLIC_SOCKET_URL

ENV NEXT_PUBLIC_API_BASE_URL=$NEXT_PUBLIC_API_BASE_URL \
    NEXT_PUBLIC_SOCKET_URL=$NEXT_PUBLIC_SOCKET_URL

RUN bun run build
```

Next.js production image-এর জন্য `next.config.ts`:

```ts
const nextConfig = {
  output: "standalone",
};

export default nextConfig;
```

Backend image startup-এর আগে production migration চালাতে পারে:

```sh
#!/bin/sh
set -eu

bunx prisma migrate deploy
exec "$@"
```

Rules:

- Build stage এবং runtime stage আলাদা রাখুন
- Runtime container non-root user দিয়ে চালানো preferred
- `.dockerignore` দিয়ে `.env`, `node_modules`, `.git` ও build output বাদ দিন
- Production migration-এ `migrate deploy` ব্যবহার করুন; `migrate dev` নয়
- Application source change না হলে Docker dependency layer cache হওয়া উচিত

## 7. Production Compose design

Central `compose.yaml` সাধারণত রাখবে:

- website
- dashboard
- backend
- PostgreSQL
- Redis
- nginx
- Certbot

Important design rules:

- Website/dashboard/backend host ports publish করবেন না
- শুধু nginx-এর `80:80` এবং `443:443` publish করুন
- PostgreSQL ও Redis internal network-এ রাখুন
- Database/cache/certificates named volumes-এ রাখুন
- `restart: unless-stopped` ব্যবহার করুন
- Healthchecks যোগ করুন
- Backend-এর `depends_on`-এ database/cache `service_healthy` ব্যবহার করুন
- App images `${WEBSITE_IMAGE}`, `${DASHBOARD_IMAGE}`, `${BACKEND_IMAGE}` থেকে নিন
- Deploy-এর সময় `.env`-এ immutable `sha-...` tag লিখুন

Example image reference:

```yaml
services:
  website:
    image: ${WEBSITE_IMAGE:-ghcr.io/github-owner/my-project-website:latest}
    restart: unless-stopped

volumes:
  postgres-data:
    name: my-project-postgres-data
    external: true
```

External volumes ব্যবহার করলে bootstrap script-এ প্রথমে create করতে হবে:

```bash
docker volume create my-project-postgres-data
docker volume create my-project-redis-data
docker volume create my-project-letsencrypt
docker volume create my-project-certbot-web
```

`docker compose down -v` production-এ চালাবেন না। এটি persistent volumes delete করতে পারে।

## 8. nginx and HTTPS

nginx virtual hosts:

```text
example.com               → website:3000
dashboard.example.com     → dashboard:3000
api.example.com           → backend:5000
```

API proxy-তে Socket.io/WebSocket headers রাখুন:

```nginx
proxy_http_version 1.1;
proxy_set_header Upgrade $http_upgrade;
proxy_set_header Connection $connection_upgrade;
proxy_set_header Host $host;
proxy_set_header X-Real-IP $remote_addr;
proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
proxy_set_header X-Forwarded-Proto $scheme;
```

First certificate-এর জন্য temporary HTTP-only nginx চালিয়ে Certbot webroot challenge ব্যবহার করা যায়। Certificate volume তৈরি হওয়ার পর full HTTPS nginx start করুন। Certbot renewal container এবং periodic nginx reload রাখুন।

যদি host nginx আগে থেকেই ports 80/443 ব্যবহার করে, cutover-এর ঠিক আগে interactiveভাবে stop করুন:

```bash
sudo systemctl disable --now nginx
sudo ss -ltnp | grep -E ':80|:443' || echo "ports-80-443-free"
```

Certificate এবং নতুন stack ready হওয়ার আগে এটি করবেন না, কারণ এই command থেকে downtime শুরু হয়।

## 9. SSH key for GitHub Actions

Windows PowerShell-এ deploy-only key তৈরি করুন:

```powershell
ssh-keygen -t ed25519 -C "github-actions-my-project" -f "$env:USERPROFILE\.ssh\my_project_github_actions"
```

CI workflow যদি সরাসরি private key ব্যবহার করে, key-তে passphrase রাখবেন না। Public key VPS-এ যোগ করুন:

```powershell
Get-Content "$env:USERPROFILE\.ssh\my_project_github_actions.pub" |
ssh -p 22 deploy@203.0.113.10 "umask 077; mkdir -p ~/.ssh; cat >> ~/.ssh/authorized_keys"
```

Test:

```powershell
ssh -o BatchMode=yes -i "$env:USERPROFILE\.ssh\my_project_github_actions" -p 22 deploy@203.0.113.10 "echo deploy-key-ok"
```

Private key-এর full content `VPS_SSH_KEY` secret হবে। Host public key line `VPS_KNOWN_HOSTS` secret হবে। Private key কখনো VPS public-key file বা Git repository-তে দেবেন না।

## 10. Selective GitHub Actions workflow

Workflow তিনটি কাজ করবে:

1. Changed paths detect
2. Affected images build/push
3. Affected services deploy

Path mapping:

```text
website/**             → website=true
admin-dashboard/**     → dashboard=true
backend/**             → backend=true
compose.yaml           → infrastructure=true
deploy/**              → infrastructure=true
.github/workflows/**   → rebuild/reconcile everything
```

Images publish করুন:

```text
ghcr.io/github-owner/my-project-website:latest
ghcr.io/github-owner/my-project-website:sha-<full-commit-sha>
```

Equivalent dashboard/backend images-ও publish হবে। Production deployment-এর `.env` immutable SHA tag ব্যবহার করবে, যাতে rollback deterministic হয়।

Workflow permissions minimum রাখুন:

```yaml
permissions:
  contents: read
  packages: write
```

Deploy job-এর জন্য:

```yaml
permissions:
  contents: read
  packages: read
```

Initial push-এর আগে:

```text
CD_ENABLED=false
```

তখন images build/push হবে, কিন্তু live VPS deploy হবে না। First cutover successful হওয়ার পর `CD_ENABLED=true` করুন।

## 11. GHCR visibility

VPS-এ images pull করার দুইটি option:

### Public packages

GHCR packages public করলে token ছাড়া pull করা যাবে:

```bash
docker pull ghcr.io/github-owner/my-project-website:latest
```

### Private packages

Classic GitHub PAT-এর minimum `read:packages` scope ব্যবহার করুন:

```bash
read -rsp 'GHCR token: ' MY_PROJECT_GHCR_TOKEN; echo
printf '%s' "$MY_PROJECT_GHCR_TOKEN" | docker login ghcr.io -u github-owner --password-stdin
unset MY_PROJECT_GHCR_TOKEN
```

Token command history, `.env`, README বা chat-এ paste করবেন না।

## 12. First deployment sequence

এই order follow করা গুরুত্বপূর্ণ।

### Step 1: Push with deployment disabled

```powershell
git add .
git commit -m "chore: add monorepo CI/CD"
git push origin main
```

Expected:

```text
Detect changes   success
Build website    success
Build dashboard  success
Build backend    success
Deploy services  skipped
```

### Step 2: Copy deployment definition

```powershell
ssh -p 22 deploy@203.0.113.10 "mkdir -p ~/apps/my-project"
scp -P 22 compose.yaml deploy@203.0.113.10:/home/deploy/apps/my-project/
scp -P 22 -r deploy deploy@203.0.113.10:/home/deploy/apps/my-project/
```

Windows `scp` executable bit preserve নাও করতে পারে। Scripts সবসময় `bash script.sh` দিয়ে invoke করুন অথবা VPS-এ:

```bash
chmod 700 ~/apps/my-project/deploy/scripts/*.sh
```

### Step 3: Create production `.env`

```bash
cd ~/apps/my-project
cp deploy/.env.example .env
chmod 600 .env
nano .env
```

### Step 4: Validate before downtime

```bash
docker compose config --quiet && echo "compose-config-ok"
docker compose pull website dashboard backend
```

### Step 5: Back up existing database

নতুন project হলে এই step empty হতে পারে। Existing deployment replace করলে database dump অবশ্যই নিন:

```bash
bash deploy/scripts/backup-vps.sh
test -s /path/to/backup/database.dump && ls -lh /path/to/backup/database.dump
```

Backup file size zero হলে cutover করবেন না।

### Step 6: Start downtime and bootstrap

```bash
sudo systemctl disable --now nginx
sudo ss -ltnp | grep -E ':80|:443' || echo "ports-80-443-free"

CONFIRM_RESET=RESET_MY_PROJECT bash deploy/scripts/bootstrap-vps.sh
```

Destructive confirmation string project অনুযায়ী আলাদা রাখুন। Bootstrap script-এর expected responsibilities:

- preflight validation
- timestamped database/config backup
- শুধু explicitly known legacy containers stop/remove
- old app directories recoverable backup location-এ move
- named volumes create
- PostgreSQL/Redis start
- database restore
- initial TLS certificate request
- complete stack start
- service status print

কখনো generic `docker rm -f $(docker ps -aq)` বা `docker system prune --volumes` ব্যবহার করবেন না। একই VPS-এর অন্য projects ক্ষতিগ্রস্ত হতে পারে।

### Step 7: Verify production

```bash
docker compose ps

curl -fsS https://api.example.com/
curl -fsSI https://example.com/
curl -fsSI https://dashboard.example.com/
```

Database restore verify করুন:

```bash
docker compose exec -T postgres \
  psql -U postgres -d my_project -Atc \
  "select pg_size_pretty(pg_database_size(current_database()));"
```

Login, authentication refresh, uploads, payments, webhooks এবং real-time Socket.io manually test করুন। Health endpoint success মানেই সব business flow success নয়।

### Step 8: Enable automatic deployment

GitHub Repository variable:

```text
CD_ENABLED=true
```

তারপর workflow manual run করে complete automatic deployment একবার verify করুন। Production images-এর tags `sha-...` হওয়া উচিত।

## 13. Daily selective deployment

### Website only

```powershell
git status
git add website
git commit -m "feat(website): describe the change"
git push origin main
```

Expected:

```text
Build website            success
Build dashboard          skipped
Build backend            skipped
Deploy changed services  success
```

### Dashboard only

```powershell
git add admin-dashboard
git commit -m "feat(dashboard): describe the change"
git push origin main
```

### Backend only

```powershell
git add backend
git commit -m "feat(backend): describe the change"
git push origin main
```

Infrastructure changes ইচ্ছাকৃত না হলে `git add .` এড়িয়ে affected path stage করুন। Commit-এর আগে সবসময় `git status` দেখুন।

## 14. Verification and operations

Status:

```bash
cd ~/apps/my-project
docker compose ps
```

Logs:

```bash
docker compose logs --tail=200 backend
docker compose logs -f website
docker compose logs -f nginx
```

Current deployed images:

```bash
grep -E '^(WEBSITE|DASHBOARD|BACKEND)_IMAGE=' .env
```

Certificate status:

```bash
docker compose logs --tail=100 certbot
```

Database backup:

```bash
bash deploy/scripts/backup-vps.sh
```

Resource usage:

```bash
docker stats --no-stream
df -h /
docker system df
```

## 15. Rollback

GHCR-এ previous immutable SHA tag থাকলে `.env`-এ affected image পরিবর্তন করুন:

```env
WEBSITE_IMAGE=ghcr.io/github-owner/my-project-website:sha-previous-commit
```

তারপর শুধু সেই service recreate করুন:

```bash
docker compose pull website
docker compose up -d --no-deps website
docker compose exec -T nginx nginx -s reload
docker compose ps
```

Backend rollback-এর আগে database migration compatibility যাচাই করুন। Code rollback database schema automatically reverse করে না। Destructive migration হলে tested restore plan প্রয়োজন।

## 16. Common failures

### `Permission denied` for a shell script

Cause: Windows/SCP executable bit হারিয়েছে।

```bash
chmod 700 deploy/scripts/*.sh
bash deploy/scripts/bootstrap-vps.sh
```

Nested scripts-ও direct execution না করে `bash /path/script.sh` দিয়ে call করা safer।

### Ports 80/443 already allocated

```bash
sudo ss -ltnp | grep -E ':80|:443'
sudo systemctl disable --now nginx
```

Unknown process হলে identify না করে kill করবেন না।

### GHCR `denied` or `unauthorized`

- Package public কিনা দেখুন
- Private হলে `read:packages` PAT দিয়ে login করুন
- Image owner/name lowercase এবং exact কিনা দেখুন
- Tag exists কিনা GitHub Packages page-এ দেখুন

### Compose says `.env` missing

```bash
cd ~/apps/my-project
cp deploy/.env.example .env
chmod 600 .env
nano .env
```

### Application is healthy but nginx returns 502

```bash
docker compose ps
docker compose logs --tail=100 nginx
docker compose exec -T nginx nginx -t
docker compose exec -T nginx nginx -s reload
```

Container recreate-এর পর nginx reload করলে upstream service address আবার resolve হয়।

### Deployment job is always skipped

Check:

- `CD_ENABLED` Repository variable কি `true`?
- এটি Environment variable হিসেবে ভুল জায়গায় আছে কি?
- Deploy job কি `environment: production` ব্যবহার করছে?
- Environment secrets-এর names workflow-এর সঙ্গে exact match করছে কি?

### Certificate request fails

Check:

- DNS সব domains-এর জন্য VPS-এ point করছে
- Port 80 publicly accessible
- Host nginx/Apache port 80 দখল করে নেই
- Certbot webroot volume nginx-এর সঙ্গে shared
- Let’s Encrypt rate limit hit হয়নি

## 17. Security and maintenance checklist

- [ ] `.env` files ignored and permission `600`
- [ ] Deploy-only SSH key ব্যবহার করা হয়েছে
- [ ] Production environment শুধু `main` allow করে
- [ ] GitHub token permissions minimum
- [ ] App/database/cache ports host-এ publish করা হয়নি
- [ ] Database volume persistent এবং backup tested
- [ ] TLS automatic renewal configured
- [ ] nginx periodically renewed certificate reload করে
- [ ] Images immutable SHA tags দিয়ে deploy হয়
- [ ] Healthchecks configured
- [ ] Backend migrations production-safe
- [ ] Rollback SHA জানা আছে
- [ ] Old backups immediately delete করা হয়নি
- [ ] VPS security updates এবং disk usage monitored

## 18. Recommended order for every new project

```text
1. Decide domains, ports, image names and repository layout
2. Add root ignores and LF rules
3. Make every app build successfully on its own
4. Add production Dockerfiles
5. Add central Compose and internal networks
6. Add nginx and Certbot configuration
7. Add guarded backup/bootstrap/deploy scripts
8. Add selective GitHub workflow
9. Add Repository variables and production Environment secrets
10. Push with CD_ENABLED=false
11. Confirm all images exist in GHCR
12. Prepare VPS .env and pull images before downtime
13. Back up old database and configuration
14. Run guarded cutover
15. Verify HTTPS, data and critical business flows
16. Set CD_ENABLED=true
17. Test one automatic deployment
18. Keep backups until the new deployment is proven stable
```

এই order-এর মূল উদ্দেশ্য হলো application build, image publication, secrets, database backup এবং DNS verify হওয়ার আগে live infrastructure না থামানো।

