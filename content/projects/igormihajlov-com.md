+++
title = "igormihajlov.com"
date = "2025-12-10T00:00:00+01:00"
author = "Igor Mihajlov"
cover = ""
tags = ["hugo", "docker", "github-actions", "cloudflare"]
keywords = ["personal website", "hugo", "docker swarm", "ci/cd"]
description = "My personal website built with Hugo and deployed via GitHub Actions to Docker Swarm."
showFullContent = false
readingTime = false
hideComments = false
+++

My personal website — the one you're on right now.

## Tech Stack

- **Hugo** — Static site generation
- **Docker Swarm** — Container orchestration
- **GitHub Actions** — Automated CI/CD pipeline
- **GHCR** — Container registry
- **Cloudflare** — DNS and SSL
- **Nginx** — Reverse proxy

## How It Works

Every push to `master` triggers a GitHub Actions workflow that builds the Hugo site, packages it into a Docker image, pushes it to GitHub Container Registry, and deploys it to my VPS via SSH.

I wrote a detailed blog post about the entire setup: [Deploying a Hugo Site with GitHub Actions, Docker Swarm, and Cloudflare](/posts/deploying-hugo-site-github-actions-docker-swarm)

## Links

- [GitHub Repository](https://github.com/irog-c/igormihajlov.com)
- [Live Site](https://igormihajlov.com)
