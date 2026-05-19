---
title: "The Simplest CI/CD: Automating Next.js Deployment with a Bash Script"
date: 2024-09-23
draft: false
tags: ["nextjs", "cicd", "bash", "aws", "ec2", "webhook"]
categories: ["DevOps"]
description: "Building a zero-cost, no-Jenkins CI/CD pipeline for Next.js on a t2.micro EC2 instance using bash scripts and webhooks."
showToc: true
---

## Introduction

### Summary

1. Set up a webhook server on an AWS EC2 instance.
2. When a `push` event occurs on a GitHub repository, GitHub sends a POST request to the webhook URL.
3. The webhook server receives the POST request and executes a specified script.
4. The script runs: `git pull` → `npm run build` → `pm2 reload`.

### Why bother?

1. The AWS EC2 t2.micro instance is provided free of charge to Free Tier users.
2. However, it only has 1 vCPU and 1 GB of memory, making it very underpowered.
3. Running tools like Jenkins for CI/CD on such an instance is impractical.
4. To achieve "zero-downtime automated deployment" with minimal resources, we'll use a bash shell script.
5. Vercel would solve this easily, but since my company doesn't use Vercel, I needed an alternative.

## Required Packages

### webhook

**Overview**

A package for building a webhook server. When it receives a POST request, it executes a specified script.

**GitHub Repository**

https://github.com/adnanh/webhook

**Install command**

```bash
apt install webhook
```

### pm2

**Overview**

A Node.js process manager. It manages processes, provides log monitoring, and automatically restarts processes that crash.

**Install command**

```bash
npm install pm2 -g
```

## Cloning an Existing Next.js Project to EC2

### Generating a GitHub Token

1. GitHub: Settings → Developer settings → Personal access tokens → Generate new token (classic)

![Generate GitHub token](/images/simple-cicd-nextjs-bash/image-1.png)

2. Set the token expiration to `No Expiration` and select `repo` as the scope.

![Token scope settings](/images/simple-cicd-nextjs-bash/image-2.png)

### Cloning the Repository

1. SSH into the EC2 instance.

```bash
ssh ubuntu@12.34.56.78 -i your-key.pem
```

2. Clone the Next.js project repository you've been working on locally.

```bash
git clone https://<username>:<token>@github.com/<username>/<repository>
```

3. In my case, the project folder was structured as follows.

![Project folder structure](/images/simple-cicd-nextjs-bash/image-3.png)

4. The `deploy` folder and `run.sh` script in the screenshot above were added by me.

### Writing the Build and Run Scripts

**deploy/hooks.json**

Defines the task to execute when the webhook receives a POST request.

```json
[
  {
    "id": "rebuild",
    "execute-command": "rebuild.sh",
    "command-working-directory": "/home/ubuntu/my-nextjs-project/deploy",
    "response-message": "push event received",
    "trigger-rule": {
      "match": {
        "type": "value",
        "value": "refs/heads/master",
        "parameter": {
          "source": "payload",
          "name": "ref"
        }
      }
    }
  }
]
```

**deploy/rebuild.sh**

Runs in order: `git pull` → `npm run build` → `pm2 reload`.

```bash
#!/bin/bash

cd /home/ubuntu/my-nextjs-project
git pull
chmod +x ./deploy/rebuild.sh
npm run build
pm2 reload deploy/ecosystem.config.js
```

**deploy/ecosystem.config.js**

The pm2 configuration file.

```js
module.exports = {
  apps: [
    {
      name: "my-nextjs-project",
      cwd: "/home/ubuntu/my-nextjs-project",
      script: "npm",
      args: "start",
      instances: 0,
      exec_mode: "cluster",
    },
  ],
};
```

**run.sh**

Starts both `pm2` and `webhook`. The `&` suffix runs them in the background.

```bash
#!/bin/bash

cd /home/ubuntu/my-nextjs-project/deploy
pm2 start ecosystem.config.js &
webhook -hooks hooks.json -port 12345 &
```

### Running

```bash
chmod +x ./run.sh
./run.sh
```

## Opening Ports on the EC2 Instance

1. AWS console: EC2 → Security Groups → Select security group → Edit inbound rules

2. Add the desired port. I added port 12345.

![Inbound rules](/images/simple-cicd-nextjs-bash/image-5.png)

3. The `13.209.1.56/29` range in the screenshot is the AWS Seoul region Instance Connect IP range.

## Setting Up GitHub Webhooks

1. GitHub: Repository → Settings → Webhooks → Add webhook

2. Configure it as shown in the screenshot.

![GitHub webhook settings](/images/simple-cicd-nextjs-bash/image-4.png)

3. Click `Add webhook` to register the webhook.

4. After that, whenever a `push` event occurs on the `main` branch, a POST request is sent to the EC2 instance with a body like this:

```json
{
  "event": "push",
  "payload": {
    "ref": "refs/heads/main",
    "before": "1214900eca16aa54d97d062e7b72261616fd53aa",
    "after": "40a717b3644e2ddec52cf6c8bfa436767bf0704e",
    "repository": {
      "id": 17892893,
```

5. The EC2 instance receives the POST request and executes the `rebuild.sh` script.

## Wrapping Up

- This approach is very simple, but security vulnerabilities do exist.
- It is suitable for personal or small-scale projects.
