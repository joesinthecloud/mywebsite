<h1 align="center">Joe’s Cloud & DevOps Portfolio — Resilient Static Hosting (PoC)</h1>

> **Note:** My primary portfolio is hosted at [joesinthe.cloud](https://www.joesinthe.cloud).  
> This site (**joesinthecloud.net**) is a **proof-of-concept** demonstrating resilient static hosting using **GitHub Pages (primary)**, fronted by **AWS CloudFront** with **S3 failover**, automated via **GitHub Actions** and **OIDC** (no static AWS keys).

🔗 **Live (PoC):** [https://www.joesinthecloud.net](https://www.joesinthecloud.net)  
🔗 **LinkedIn:** [https://www.linkedin.com/in/joenervisjr](https://www.linkedin.com/in/joenervisjr)  
🔗 **GitHub:** [https://github.com/joesinthecloud](https://github.com/joesinthecloud)

---

## 🚀 Executive Summary
I built a static portfolio site engineered for **high availability** and **secure automation**:

- **Primary origin:** GitHub Pages  
- **CDN & Failover:** Amazon CloudFront with an origin group (GitHub → S3 mirror)  
- **Private mirror:** S3 bucket (OAC, not public)  
- **TLS:** AWS Certificate Manager (ACM, us-east-1)  
- **DNS:** Route 53 (delegated from registrar)  
- **CI/CD:** GitHub Actions builds Jekyll, publishes to Pages, syncs to S3, and invalidates CloudFront  
- **Auth:** GitHub → AWS OIDC (temporary credentials, least privilege)

---

## 🏗️ Architecture (What & Why)

- **Route 53** → Aliases apex & `www` to CloudFront  
- **ACM (us-east-1)** → Cert for `joesinthecloud.net` and `www.joesinthecloud.net`  
- **CloudFront** → TLS termination, caching, Brotli/Gzip, **origin failover**  
  - Primary origin: `github.io` with origin path `/mywebsite`  
  - Secondary origin: S3 mirror (private, OAC)  
  - Default root object: `index.html`  
  - Origin request policy: `Managed-AllViewerExceptHostHeader`  
- **S3 Mirror** → Private bucket, public access blocked, readable only by CloudFront  
- **GitHub Actions** → Jekyll build → Pages deploy → download artifact → sync to S3 → invalidate CloudFront  
- **IAM + OIDC** → Scoped trust policy limited to this repo & environment

---

## 🔁 CI/CD Flow

1. Push to `main`  
2. Build with Jekyll → `_site/`  
3. Upload built site as Pages artifact  
4. Deploy to GitHub Pages  
5. Download artifact, extract, **sync to S3** (mirror)  
6. **Invalidate CloudFront** to refresh edge cache

---

## 🧩 Key Configurations

### `_config.yml` (Jekyll)

```yaml
title: "Joe’s Cloud & DevOps Portfolio"
description: "Resilient static hosting on GitHub Pages + CloudFront + S3 failover"
remote_theme: pages-themes/cayman@v0.2.0
plugins: [jekyll-remote-theme]
markdown: kramdown
url: "https://www.joesinthecloud.net"
baseurl: ""

GitHub Actions Workflow
name: Build & Deploy (Pages + S3 Mirror)

on:
  push:
    branches: [ main ]
  workflow_dispatch:

permissions:
  contents: read
  pages: write
  id-token: write

concurrency:
  group: "pages"
  cancel-in-progress: true

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Build with Jekyll
        uses: actions/jekyll-build-pages@v1
        with: { source: ., destination: ./_site }
      - name: Upload artifact for Pages
        uses: actions/upload-pages-artifact@v3
        with: { path: _site }

  deploy:
    needs: build
    runs-on: ubuntu-latest
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}
    steps:
      - name: Deploy to GitHub Pages
        uses: actions/deploy-pages@v4

      - name: Download built site artifact
        uses: actions/download-artifact@v4
        with: { name: github-pages, path: site }

      - name: Extract artifact
        run: tar -xf site/artifact.tar -C site && rm site/artifact.tar

      - name: Configure AWS credentials (OIDC)
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::<ACCOUNT_ID>:role/<ROLE_NAME>
          aws-region: us-east-1

      - name: Mirror to S3 (failover origin)
        run: aws s3 sync "site/" "s3://<MIRROR_BUCKET>/" --delete

      - name: Invalidate CloudFront
        run: aws cloudfront create-invalidation --distribution-id <DIST_ID> --paths "/*"
```

### ✅ Validation & Failover Testing
Direct Pages URL should load the site (project path).

CloudFront domain should load the site with index.html.

Simulate GitHub outage → CloudFront fails over to S3.

Each deploy ends with a CloudFront invalidation so changes appear globally in minutes.

### 📚 What I Learned
IAM + OIDC federation → Scoped trust conditions and temporary tokens.

CloudFront origin groups → Primary/secondary with failover codes.

Static site CI/CD → Build artifacts, reuse for S3 mirroring, invalidate CF.

DNS & TLS → Route 53 delegation + ACM in us-east-1.

Resilience → Tested failover and designed around no single point of failure.

### 🔗 Quick Links
Primary Portfolio: joesinthe.cloud

GitHub: github.com/joesinthecloud

LinkedIn: linkedin.com/in/joenervisjr

✨ Thanks for visiting! If you’re a recruiter or hiring manager and want a deeper dive, I’d be happy to share the full project write-up and diagrams behind this deployment.
