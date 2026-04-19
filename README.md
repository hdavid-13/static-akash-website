# Deploying a Static Website on Akash Network

This guide walks you through deploying any static HTML/CSS website on **Akash Network** (decentralized cloud), with automatic builds via **GitHub Actions** and your own custom domain via **Cloudflare**.

> 💡 **Don't have a domain yet?** You can register one at [Namecheap](https://www.namecheap.com), [Gandi](https://www.gandi.net), or [OVH](https://www.ovh.com), then transfer its DNS management to [Cloudflare](https://www.cloudflare.com) for free.

---

## Stack Overview

| Tool | Role |
|------|------|
| **GitHub** | Code storage + automatic Docker image builds |
| **ghcr.io** | Docker image registry (built into GitHub, free) |
| **Akash Network** | Decentralized hosting (pays with AKT tokens) |
| **Cloudflare** | DNS — points your domain to Akash |

### How it all fits together

```
You edit site/index.html
        ↓
git push to GitHub
        ↓
GitHub Actions automatically builds a Docker image
(nginx + your HTML/CSS files)
        ↓
Image is pushed to ghcr.io with a unique tag
        ↓
deploy.yaml is updated with the new image tag
        ↓
You update the deployment on Akash Console
        ↓
yourdomain.com shows your updated site ✅
```

---

## Project Structure

```
demo/
├── site/
│   ├── index.html              ← your website (edit this)
│   └── style.css               ← your styles (edit this)
├── Dockerfile                  ← packages nginx + your site into a Docker image
├── deploy.yaml                 ← Akash deployment config
└── .github/
    └── workflows/
        └── deploy.yml          ← automatic build on every git push
```

---

## Prerequisites

- A **GitHub** account → [github.com](https://github.com)
- An **Akash** wallet with AKT tokens (use **Keplr** browser extension)
- A **custom domain** — register one at [Namecheap](https://www.namecheap.com), [Gandi](https://www.gandi.net), or [OVH](https://www.ovh.com)
- Your domain's DNS managed on **Cloudflare** → [cloudflare.com](https://www.cloudflare.com) (free plan works)

---

## Step 1 — Set Up the GitHub Repository

1. Go to [github.com](https://github.com) and create a new **public** repository (e.g. `demo`)
2. Clone it locally or initialize it:

```bash
git clone https://github.com/YOUR_USERNAME/demo.git
cd demo
```

3. Create the project structure and add your files (see Project Structure above)

> ⚠️ Always use `git add .` (not `git add *`) — the `*` wildcard ignores hidden folders like `.github/`

---

## Step 2 — The Dockerfile

The Dockerfile packages nginx and your website files into a single Docker image:

```dockerfile
FROM nginx:alpine
COPY site/ /usr/share/nginx/html
EXPOSE 80
```

Nothing to change here — it just works.

---

## Step 3 — GitHub Actions Workflow

Create the file `.github/workflows/deploy.yml`:

```yaml
name: Build & Push Docker Image

on:
  push:
    branches: [main]

jobs:
  build-and-push:
    runs-on: ubuntu-latest
    permissions:
      contents: write
      packages: write

    steps:
      - name: Checkout code
        uses: actions/checkout@v4
        with:
          token: ${{ secrets.GITHUB_TOKEN }}

      - name: Login to GitHub Container Registry
        run: echo "${{ secrets.GITHUB_TOKEN }}" | docker login ghcr.io -u ${{ github.actor }} --password-stdin

      - name: Build & Push Docker image
        run: |
          docker build \
            -t ghcr.io/YOUR_USERNAME/YOUR_REPO:latest \
            -t ghcr.io/YOUR_USERNAME/YOUR_REPO:${{ github.sha }} .
          docker push ghcr.io/YOUR_USERNAME/YOUR_REPO:latest
          docker push ghcr.io/YOUR_USERNAME/YOUR_REPO:${{ github.sha }}

      - name: Update deploy.yaml with new tag
        run: |
          SHA=${{ github.sha }}
          sed -i "s|image: ghcr.io/YOUR_USERNAME/YOUR_REPO:.*|image: ghcr.io/YOUR_USERNAME/YOUR_REPO:${SHA}|" deploy.yaml

      - name: Commit updated deploy.yaml
        run: |
          git config user.email "actions@github.com"
          git config user.name "GitHub Actions"
          git remote set-url origin https://x-access-token:${{ secrets.GITHUB_TOKEN }}@github.com/YOUR_USERNAME/YOUR_REPO.git
          git add deploy.yaml
          git commit -m "update image tag"
          git push
```

Replace `YOUR_USERNAME` with your GitHub username (lowercase) and `YOUR_REPO` with your repository name.

### Enable workflow write permissions

In your GitHub repo → **Settings** → **Actions** → **General** → **Workflow permissions** → select **"Read and write permissions"** → Save.

---

## Step 4 — The Akash SDL (deploy.yaml)

```yaml
---
version: "2.0"

services:
  web:
    image: ghcr.io/YOUR_USERNAME/YOUR_REPO:latest
    expose:
      - port: 80
        as: 80
        accept:
          - yourdomain.org
          - www.yourdomain.org
        to:
          - global: true

profiles:
  compute:
    web:
      resources:
        cpu:
          units: 1
        memory:
          size: 128Mi
        storage:
          - size: 256Mi
  placement:
    akash:
      pricing:
        web:
          denom: uakt
          amount: 100
      signedBy:
        anyOf:
          - akash1365yvmc4s7awdyj3n2sav7xfx76adc6dnmlx63
        allOf: []

deployment:
  web:
    akash:
      profile: web
      count: 1
```

Replace `YOUR_USERNAME`, `YOUR_REPO`, and `yourdomain.org` with your actual values.

> After the first push, GitHub Actions will automatically update the image tag in this file. You don't need to edit it manually after that.

---

## Step 5 — Push to GitHub

```bash
git add .
git commit -m "first commit"
git branch -M main
git push -u origin main
```

Then go to the **Actions** tab on your GitHub repo. You should see the workflow running. Wait for it to turn green ✅.

---

## Step 6 — Make the Docker Image Public

By default, ghcr.io images are private. Akash needs to pull the image, so it must be public.

1. Go to **github.com/YOUR_USERNAME** → **Packages** tab
2. Click on **YOUR_REPO**
3. Click **Package settings** (bottom right)
4. **Change visibility** → **Public** → Confirm

---

## Step 7 — Deploy on Akash

1. Go to [console.akash.network](https://console.akash.network)
2. Connect your **Keplr** wallet (make sure you have AKT tokens)
3. Click **Deploy** → **From SDL**
4. Paste the contents of your `deploy.yaml`
5. Choose a provider and confirm the deployment
6. Note the **public URL** given by Akash (e.g. `xyz.provider.akash.network`)

---

## Step 8 — Configure Cloudflare DNS

In your Cloudflare dashboard for your domain → **DNS** → **Records** → Add:

| Type | Name | Value | Proxy |
|------|------|-------|-------|
| CNAME | `@` | `xyz.provider.akash.network` | ✅ Orange (proxied) |
| CNAME | `www` | `xyz.provider.akash.network` | ✅ Orange (proxied) |

The orange cloud (proxied) enables free HTTPS automatically via Cloudflare.

---

## Updating Your Website

Every time you want to update your site:

```bash
git pull                        # always pull first!
# edit site/index.html or site/style.css
git add .
git commit -m "update site"
git push
```

Then:
1. Wait for the GitHub Actions workflow to turn green ✅
2. Run `git pull` to get the updated `deploy.yaml` with the new image tag
3. Go to **Akash Console** → your deployment → **Update**
4. Paste the new `deploy.yaml` content → Confirm

Your site will reload with the latest version.

---

## Common Pitfalls

| Problem | Cause | Solution |
|---------|-------|----------|
| Workflow not triggering | `.github/` folder missing | Use `git add .` not `git add *` |
| `repository name must be lowercase` | GitHub org name has uppercase letters | Hardcode your lowercase username in the workflow |
| `permission_denied` on ghcr.io push | Org restricts package visibility | Use a personal account repo instead |
| `403` when bot tries to push | Workflow permissions too restrictive | Enable "Read and write permissions" in Settings → Actions |
| Site not updating after deploy | Akash cached the `:latest` tag | Use commit SHA tags (handled automatically by this workflow) |
| `fatal: need to specify how to reconcile` | Bot pushed while you were editing | Run `git config pull.rebase false` then `git pull` |

---

## Quick Reference

```bash
# First time setup
git clone https://github.com/YOUR_USERNAME/YOUR_REPO.git
cd YOUR_REPO
git add .
git commit -m "first commit"
git push -u origin main

# Every update
git pull
# edit your files
git add .
git commit -m "your message"
git push
# then update on Akash Console with the new deploy.yaml
```
