---
title: "Deploying on AWS EC2 with a Custom Domain and HTTPS"
date: 2025-01-23
draft: false
tags: ["aws", "ec2", "https", "nextjs", "loadbalancer"]
categories: ["DevOps"]
description: "A step-by-step guide to deploying a Next.js app on EC2, connecting a custom domain via Route 53, issuing an SSL certificate with ACM, and terminating HTTPS at the load balancer."
showToc: true
---

## Introduction

### Summary

1. Run a Next.js app on an EC2 instance.
2. Purchase a domain and connect it via Route 53.
3. Issue an SSL certificate via ACM.
4. Load balancer listener: TCP 443 → TCP 3000.

### Why bother?

- The Next.js project you develop at `http://localhost:3000` needs to be deployed eventually.
- HTTPS is a mandatory security requirement for modern web services.

## 1. Running a Next.js App on EC2

### 1.1. Create an EC2 Instance

Create an EC2 instance from the AWS Management Console.

### 1.2. SSH Into the Instance

SSH into the EC2 instance you just created.

### 1.3. Install Node.js via NVM

NVM (Node Version Manager) lets you manage multiple Node.js versions. It makes installing both the latest and older versions straightforward.

```bash
wget -qO- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.1/install.sh | bash
source ~/.bashrc
```

Verify the installation:

```bash
nvm --version
```

Install Node.js. I installed version 18.20.4 based on my project's requirements:

```bash
nvm install 18.20.4
```

Verify the installed version:

```bash
node --version
```

### 1.4. Run the Next.js App with PM2

PM2 is a process manager for Node.js apps.

```bash
npm install pm2 -g
```

Start the Next.js app:

```bash
pm2 start npm --name "next" -- start
```

Check running processes:

```bash
pm2 list
```

### 1.5. Verify It Works

Access the app using the EC2 instance's public IP and port number, e.g. `http://12.34.56.78:3000/`.

If the connection succeeds, you'll see the "Not secure" warning in the address bar — that's expected at this stage.

![Not secure warning](/images/aws-ec2-deploy-domain-https/image-2.png)

## 2. Purchasing a Domain and Connecting via Route 53

This blog post covers domain purchase on Gabia and Route 53 setup in detail:

- https://jindevelopetravel0919.tistory.com/189

## 3. Issuing an SSL Certificate via ACM

Search for AWS Certificate Manager in the AWS Management Console and create an SSL/TLS certificate.

This blog post covers the ACM certificate issuance process in detail:

- https://pizza301.tistory.com/99

Here's how I configured it by following that guide:

![ACM configuration](/images/aws-ec2-deploy-domain-https/Screenshot_2024-09-29_at_8.40.33_PM.png)

## 4. Load Balancer Setup: TCP 443 → TCP 3000

Create a load balancer and add listeners.

Configure the listener to redirect HTTP (TCP 80) requests to HTTPS (TCP 443):

![Load balancer listener](/images/aws-ec2-deploy-domain-https/Screenshot_2024-09-29_at_8.32.01_PM.png)

Configure the target group:

![Target group](/images/aws-ec2-deploy-domain-https/image.png)

## Result

Visiting `https://domain.com` now serves the Next.js app, and the SSL certificate is in place — no "unsafe site" warning.

![HTTPS working](/images/aws-ec2-deploy-domain-https/image-1.png)
