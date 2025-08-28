Joe’s Cloud & DevOps Portfolio — Resilient Static Hosting (PoC)

Note: My primary portfolio is hosted at https://www.joesinthe.cloud.

This site (joesinthecloud.net) is a proof-of-concept that demonstrates resilient static hosting using GitHub Pages (primary) fronted by AWS CloudFront with S3 failover, automated via GitHub Actions with OIDC (no static AWS keys).

Live (PoC): https://www.joesinthecloud.net

LinkedIn: https://www.linkedin.com/in/joenervisjr

GitHub: https://github.com/joesinthecloud

🚀 Executive Summary

I built a static portfolio site engineered for high availability and secure automation:

Primary origin: GitHub Pages

CDN & failover: Amazon CloudFront with an origin group (GitHub → S3 mirror)

Private mirror: S3 bucket (OAC; not public)

TLS: AWS Certificate Manager (ACM, us-east-1)

DNS: Route 53 (delegated from registrar)

CI/CD: GitHub Actions builds Jekyll, publishes to Pages, syncs to S3, and invalidates CloudFront

Auth: GitHub → AWS OIDC (temporary creds, least privilege)

Why this matters: it’s a production-style pattern showing CDN fronting, origin failover, DNS+TLS, IaC-like repeatability, and a secure federated CI/CD pipeline.

🏗️ Architecture (What & Why)

Route 53 → Aliases apex & www to CloudFront

ACM (us-east-1) → Cert for joesinthecloud.net, www.joesinthecloud.net

CloudFront → TLS termination, caching, Brotli/Gzip, origin failover

Primary origin: *.github.io with origin path /mywebsite

Secondary origin: S3 mirror (private) via OAC

Default root object: index.html

Origin request policy: Managed-AllViewerExceptHostHeader (ensures GitHub sees its expected Host)

S3 Mirror → private bucket, public access blocked; readable only by CloudFront (OAC)

GitHub Actions → Jekyll build → Pages deploy → download artifact → sync to S3 → CloudFront invalidation

IAM + OIDC → Trusts only this repo/environment; policies limited to the mirror bucket + CloudFront invalidation

🔁 CI/CD Flow

Push to main

Build with Jekyll → _site/

Upload built site as Pages artifact

Deploy to GitHub Pages

Download artifact, extract, sync to S3 (mirror)

Invalidate CloudFront to refresh the edge

🧩 Key Configurations
_config.yml (Jekyll)
title: "Joe’s Cloud & DevOps Portfolio"
description: "Resilient static hosting on GitHub Pages + CloudFront + S3 failover"
remote_theme: pages-themes/cayman@v0.2.0
plugins: [jekyll-remote-theme]
markdown: kramdown
url: "https://www.joesinthecloud.net"
baseurl: ""

GitHub Actions (build → deploy → mirror → invalidate)
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
        id: deployment
        uses: actions/deploy-pages@v4

      - name: Download built site artifact
        uses: actions/download-artifact@v4
        with: { name: github-pages, path: site }

      - name: Extract artifact
        run: |
          tar -xf site/artifact.tar -C site
          rm site/artifact.tar

      - name: Configure AWS credentials (OIDC)
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::<ACCOUNT_ID>:role/<ROLE_NAME>
          aws-region: us-east-1

      - name: Mirror to S3 (failover origin)
        run: aws s3 sync "site/" "s3://<MIRROR_BUCKET>/" --delete

      - name: Invalidate CloudFront
        run: aws cloudfront create-invalidation --distribution-id <DISTRIBUTION_ID> --paths "/*"

IAM Trust Policy (allow this repo’s OIDC token only)
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": { "Federated": "arn:aws:iam::<ACCOUNT_ID>:oidc-provider/token.actions.githubusercontent.com" },
    "Action": "sts:AssumeRoleWithWebIdentity",
    "Condition": {
      "StringEquals": { "token.actions.githubusercontent.com:aud": "sts.amazonaws.com" },
      "StringLike": {
        "token.actions.githubusercontent.com:sub": [
          "repo:<OWNER>/<REPO>:environment:github-pages",
          "repo:<OWNER>/<REPO>:ref:refs/heads/main"
        ]
      }
    }
  }]
}

IAM Permissions (least privilege)
{
  "Version": "2012-10-17",
  "Statement": [
    { "Effect": "Allow", "Action": ["s3:ListBucket"], "Resource": "arn:aws:s3:::<MIRROR_BUCKET>" },
    { "Effect": "Allow", "Action": ["s3:PutObject","s3:DeleteObject"], "Resource": "arn:aws:s3:::<MIRROR_BUCKET>/*" },
    { "Effect": "Allow", "Action": "cloudfront:CreateInvalidation", "Resource": "arn:aws:cloudfront::<ACCOUNT_ID>:distribution/<DISTRIBUTION_ID>" }
  ]
}

✅ Validation & Failover Testing

Direct Pages URL should load the site (project path).

CloudFront domain should load the site; / resolves via index.html.

Simulate GitHub outage by marking primary origin unhealthy → confirm S3 mirror serves content.

Each deploy ends with a CloudFront invalidation so changes appear quickly.

📚 What I Learned

Federated IAM with OIDC: read JWT claims and scoped trust precisely to my repo/workflow.

CloudFront origin groups: primary/secondary, failover codes, and the importance of Host header handling.

Static site CI/CD: build artifacts, Pages deploy, artifact reuse for S3 mirroring, and edge invalidation.

DNS & TLS: Route 53 delegation + ACM (us-east-1) requirements for CloudFront.

Resilience mindset: testing failover and designing for no single point of failure.

🔗 Quick Links

Primary portfolio: https://www.joesinthe.cloud

GitHub: https://github.com/joesinthecloud

LinkedIn: https://www.linkedin.com/in/joenervisjr


If you’re a recruiter or hiring manager and want a deeper dive, I'd be happy to share the full write-up and diagrams behind this deployment.

✨ Thanks for visiting — check out https://www.joesinthe.cloud
 to learn more about me, and explore more of my projects!
