+++
title = "Deploying a Hugo Site with GitHub Actions, Docker Swarm, and Cloudflare"
date = "2025-12-10T00:00:00+01:00"
author = "Igor Mihajlov"
cover = ""
tags = ["hugo", "docker", "github-actions", "cloudflare", "devops"]
keywords = ["hugo", "docker swarm", "github actions", "ci/cd", "cloudflare", "nginx", "deployment"]
description = "A complete guide to setting up automated deployments for a Hugo static site using GitHub Actions, Docker Swarm, GHCR, and Cloudflare SSL."
showFullContent = false
readingTime = true
hideComments = false
+++

I recently set up my personal website with a fully automated deployment pipeline. Every time I push to the `master` branch, my site automatically builds, containerizes, and deploys to my VPS. Here's how I did it.

## The Stack

- **Hugo** — Static site generator
- **Docker** — Containerization
- **GitHub Actions** — CI/CD pipeline
- **GHCR** — Container registry
- **Docker Swarm** — Orchestration on the VPS
- **Nginx** — Reverse proxy
- **Cloudflare** — DNS and SSL

## Setting Up the Repository

First, I initialized a local Git repository and connected it to GitHub via SSH:

```bash
git init
git remote add origin git@github.com:username/mysite.git
```

If you haven't set up SSH keys with GitHub, you'll need to generate one and add it to your account:

```bash
ssh-keygen -t ed25519 -C "your_email@example.com"
cat ~/.ssh/id_ed25519.pub
# Add this to GitHub → Settings → SSH and GPG keys
```

## The Dockerfile

Hugo builds static files, so the container just needs to build the site and serve it with nginx:

```dockerfile
FROM hugomods/hugo:exts as builder

WORKDIR /src
COPY . .
RUN hugo --minify

FROM nginx:alpine
RUN rm -rf /usr/share/nginx/html/*
COPY --from=builder /src/public /usr/share/nginx/html
EXPOSE 80
```

The `rm -rf` line is important — it clears nginx's default welcome page before copying your Hugo output.

If you're using a theme as a Git submodule, it won't be included in the Docker build automatically. The solution is to handle it in the GitHub Actions workflow instead.

## Docker Compose for Swarm

```yaml
services:
  web:
    image: ghcr.io/username/mysite:latest
    ports:
      - "81:80"
    deploy:
      replicas: 1
      update_config:
        order: start-first
```

I'm using port 81 here because port 80 is already used by my nginx reverse proxy.

## GitHub Actions Workflow

The workflow does three things: builds the Hugo site, pushes the image to GHCR, and deploys to my VPS via SSH.

```yaml
name: Build and Deploy

on:
  push:
    branches: [master]

env:
  IMAGE: ghcr.io/${{ github.repository }}

jobs:
  build:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      packages: write
    steps:
      - uses: actions/checkout@v4
        with:
          submodules: true

      - name: Log in to GHCR
        run: echo "${{ secrets.GITHUB_TOKEN }}" | docker login ghcr.io -u ${{ github.actor }} --password-stdin

      - name: Build and push
        run: |
          docker build -t $IMAGE:latest -t $IMAGE:${{ github.sha }} .
          docker push $IMAGE --all-tags

  deploy:
    needs: build
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup SSH
        run: |
          mkdir -p ~/.ssh
          echo "${{ secrets.VPS_PRIVATE_KEY }}" > ~/.ssh/id_ed25519
          chmod 600 ~/.ssh/id_ed25519
          ssh-keyscan -H ${{ secrets.VPS_IP }} >> ~/.ssh/known_hosts

      - name: Deploy stack
        run: |
          scp compose.yml ${{ secrets.VPS_USER }}@${{ secrets.VPS_IP }}:/tmp/mysite-compose.yml
          ssh ${{ secrets.VPS_USER }}@${{ secrets.VPS_IP }} << 'EOF'
            docker pull ghcr.io/username/mysite:latest
            docker stack deploy -c /tmp/mysite-compose.yml mysite
            rm /tmp/mysite-compose.yml
          EOF
```

The `submodules: true` option ensures your Hugo theme gets checked out if it's a submodule.

### Required Secrets

Add these in your GitHub repo under Settings → Secrets and variables → Actions:

- `VPS_IP` — Your server's IP address
- `VPS_USER` — SSH username
- `VPS_PRIVATE_KEY` — Your private SSH key (include the trailing newline)

### VPS Authentication to GHCR

Your VPS needs to pull the image from GHCR. Since images are private by default, authenticate once on your server:

```bash
echo "ghp_yourtoken" | docker login ghcr.io -u username --password-stdin
```

Create a Personal Access Token with `read:packages` scope in GitHub → Settings → Developer settings → Personal access tokens.

## Nginx Reverse Proxy

My VPS runs a dockerized nginx reverse proxy. Here's the configuration for routing traffic to the Hugo container:

```nginx
server {
    listen 80;
    server_name mysite.com www.mysite.com;
    return 301 https://mysite.com$request_uri;
}

server {
    listen 443 ssl;
    server_name www.mysite.com;

    ssl_certificate /etc/nginx/ssl/mysite.com.pem;
    ssl_certificate_key /etc/nginx/ssl/mysite.com.key;

    return 301 https://mysite.com$request_uri;
}

server {
    listen 443 ssl;
    server_name mysite.com;

    ssl_certificate /etc/nginx/ssl/mysite.com.pem;
    ssl_certificate_key /etc/nginx/ssl/mysite.com.key;

    location / {
        proxy_pass http://host.docker.internal:81;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
    }
}
```

Since the reverse proxy runs in Docker, it can't reach `localhost` on the host. The `host.docker.internal` hostname solves this — add it to your reverse proxy's compose file:

```yaml
services:
  nginx:
    image: nginx:latest
    ports:
      - "80:80"
      - "443:443"
    extra_hosts:
      - "host.docker.internal:host-gateway"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf:ro
      - ./conf.d:/etc/nginx/conf.d:ro
      - ./ssl:/etc/nginx/ssl:ro
    restart: unless-stopped
```

## Cloudflare Setup

### DNS Records

Add an A record pointing to your VPS:

- **Type:** A
- **Name:** `@`
- **IPv4 address:** Your VPS IP
- **Proxy status:** Proxied (orange cloud)

For www redirect support, add a CNAME:

- **Type:** CNAME
- **Name:** `www`
- **Target:** `mysite.com`
- **Proxy status:** Proxied

### SSL Certificates

Cloudflare Origin Certificates provide free SSL between Cloudflare and your server:

1. Go to SSL/TLS → Origin Server → Create Certificate
2. Add both `mysite.com` and `*.mysite.com` for a wildcard cert
3. Save the certificate and private key to your VPS
4. Set SSL/TLS mode to **Full (strict)**

A wildcard certificate covers your root domain and all subdomains, so you only need one cert for everything.

## Troubleshooting

### "Welcome to nginx" instead of your site

Hugo isn't generating an `index.html`, or nginx's default page is overwriting it. Make sure:

1. Your content pages have `draft: false` in the front matter
2. You have a `content/_index.md` for the homepage
3. The Dockerfile clears nginx's default files before copying

### CSS not loading

Check your `baseURL` in Hugo config — it should match your actual domain:

```toml
baseURL = "https://mysite.com/"
```

### Container can't reach host services

Use `host.docker.internal` instead of `localhost` and add the `extra_hosts` mapping to your compose file.

## The Result

Now every `git push` to master triggers a full deployment:

1. GitHub Actions builds the Hugo site
2. Creates a Docker image and pushes to GHCR
3. SSHs into the VPS
4. Pulls the new image and updates the Swarm stack

The whole process takes about a minute. No manual deployment steps, no FTP uploads, no SSH-ing in to pull changes. Just push and it's live.
