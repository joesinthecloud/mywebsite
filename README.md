<h1 align="center">Resilient Static Hosting on Amazon S3</h1>

> **Note:** My primary portfolio is hosted at [joesinthe.cloud](https://www.joesinthe.cloud).  
> This site (**joesinthecloud.net**) is a **proof-of-concept** designed to demonstrate resilient static hosting using **GitHub Pages (primary)**, fronted by **AWS CloudFront** and **AWS Web Application Firewall (WAF)** with **S3 failover**, and automated with **GitHub Actions + OIDC** (no static AWS keys).

🔗 **Live Static Site Hosted At (DEACTIVATED):** [https://www.joesinthecloud.net](https://www.joesinthecloud.net)  
🔗 **LinkedIn:** [linkedin.com/in/joenervisjr](https://www.linkedin.com/in/joenervisjr)  
🔗 **GitHub:** [github.com/joesinthecloud](https://github.com/joesinthecloud)

---

## 1️⃣ The Problem

Modern static websites are often hosted on **single points of failure** like GitHub Pages. While cost-effective, this approach lacks resilience:  

- If GitHub Pages goes down, the site is offline.  
- No built-in failover or disaster recovery.  
- DNS and SSL/TLS management often require manual steps.  

**Real-world context:** Businesses that rely on static marketing sites, documentation, or status pages risk downtime and loss of customer trust without high availability.  

---

## 2️⃣ The Solution: How I Built It

I engineered a **resilient hosting pipeline** with automated failover and security best practices, using AWS to extend GitHub Pages.

### 🔹 Architecture
- **Route 53** → Delegated DNS to AWS, alias records for apex & `www`.  
- **ACM (us-east-1)** → Free SSL/TLS cert for `joesinthecloud.net` & `www`.  
- **CloudFront** → CDN + TLS termination + **Origin Group failover**:  
  - Primary: GitHub Pages (`*.github.io/mywebsite`).  
  - Secondary: Private S3 mirror (OAC, not public).  
- **S3 Mirror** → Synced copy of the site, accessible only via CloudFront.  
- **GitHub Actions CI/CD** →  
  - Build with Jekyll.  
  - Deploy to GitHub Pages.  
  - Download artifact → sync to S3.  
  - Invalidate CloudFront for fresh global delivery.  
- **IAM + OIDC** → Federated identity between GitHub and AWS (no long-lived credentials).  

---

### 🔹 Example Configuration

**Jekyll config (`_config.yml`):**
```yaml
title: "Joe’s Cloud & DevOps Portfolio"
description: "Resilient static hosting on GitHub Pages + CloudFront + S3 failover"
remote_theme: pages-themes/cayman@v0.2.0
plugins: [jekyll-remote-theme]
url: "https://www.joesinthecloud.net"
baseurl: ""

name: Build & Deploy (Pages + S3 Mirror)

on:
  push:
    branches: [ main ]

permissions:
  contents: read
  pages: write
  id-token: write

jobs:
  build:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Build with Jekyll
        uses: actions/jekyll-build-pages@v1
        with:
          source: .
          destination: ./_site
      - name: Upload artifact for Pages
        uses: actions/upload-pages-artifact@v3
        with:
          path: _site

  deploy:
    needs: build
    runs-on: ubuntu-latest
    environment:
      name: github-pages
    steps:
      - name: Deploy to GitHub Pages
        uses: actions/deploy-pages@v4

      - name: Download built site artifact
        uses: actions/download-artifact@v4
        with:
          name: github-pages
          path: site

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

## 🔹 Validation & Testing

- Verified GitHub Pages serves content by default.  
- Simulated GitHub outage → CloudFront seamlessly failed over to S3.  
- CloudFront cache invalidated on every deploy for up-to-date global delivery.  

---

## 3️⃣ Business Impact

This project demonstrates a **production-grade approach** to static hosting that solves the availability and maintainability gaps of GitHub Pages:

- **High Availability:** Automatic failover ensures the site stays online even if GitHub is down.  
- **Global Performance:** CloudFront CDN delivers content closer to users with caching + compression.  
- **Security:** No hard-coded AWS keys; GitHub OIDC integration follows least-privilege IAM design.  
- **Automation:** Zero manual steps in deployment — developers push code, pipeline handles the rest.  
- **Cost Efficiency:** Nearly free hosting with GitHub Pages + minimal AWS costs for S3/CloudFront.  

**Business Utility:**  
- Resilient public-facing sites build **trust** with customers.  
- Automated CI/CD reduces **operational overhead** and **human error**.  
- Proves capability in **Cloud Engineering, DevOps, and Secure Automation** — skills directly applicable to enterprise infrastructure projects.  

---

## 🔗 Quick Links

- **Primary Portfolio:** [joesinthe.cloud](https://www.joesinthe.cloud)  
- **GitHub:** [github.com/joesinthecloud](https://github.com/joesinthecloud)  
- **LinkedIn:** [linkedin.com/in/joenervisjr](https://www.linkedin.com/in/joenervisjr) 
