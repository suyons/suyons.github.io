---
title: "Connecting GitHub MCP to Claude Code (and Why the Easy Way Didn't Work)"
date: 2026-05-19
draft: false
tags: ["claude", "mcp", "github", "docker", "colima"]
categories: ["DevOps"]
description: "A walkthrough of every failed attempt to connect the GitHub MCP server to Claude Code, and the Docker + Colima setup that finally worked."
showToc: true
---

## What Is MCP?

MCP (Model Context Protocol) is an open standard that lets AI assistants like Claude connect to external tools and services. With a GitHub MCP server wired up, Claude Code can directly read repositories, create files, push commits, and manage pull requests — all without leaving the conversation.

I wanted Claude Code to have full control over my GitHub account so it could help me migrate blog posts from a private repo. Here's every approach I tried, in order.

## Attempt 1: The Hosted Endpoint

The obvious first step was the GitHub-hosted MCP endpoint. Claude Code has a built-in command for adding MCP servers:

```bash
claude mcp add --transport http github https://api.githubcopilot.com/mcp/
```

This looked promising. But when I tried to authenticate, it failed immediately:

```
SDK auth failed: Incompatible auth server: does not support dynamic client registration
```

The Copilot endpoint uses its own proprietary OAuth flow. Claude Code's MCP client expects servers that follow RFC 7591 (Dynamic Client Registration), which the Copilot endpoint doesn't implement. Dead end.

## Attempt 2: Passing the Token as a Header

I thought maybe I could bypass the OAuth dance by injecting my Personal Access Token directly:

```bash
claude mcp add --transport http github https://api.githubcopilot.com/mcp/ \
  --header "Authorization: Bearer ghp_xxxxxxxxxxxx"
```

Same result. The server still tried to initiate its own auth flow before accepting any requests. The header was ignored.

## Attempt 3: Self-Hosting with Docker

GitHub publishes an official MCP server image at `ghcr.io/github/github-mcp-server`. If I run it locally and connect Claude Code to it via stdio, the PAT just gets passed in as an environment variable — no OAuth required.

The command Claude Code needs is:

```bash
claude mcp add github \
  -e GITHUB_PERSONAL_ACCESS_TOKEN=<your_token> \
  -- docker run -i --rm -e GITHUB_PERSONAL_ACCESS_TOKEN ghcr.io/github/github-mcp-server
```

### Problem: No Docker Daemon

I ran `brew install docker` thinking that would be enough:

```bash
brew install docker
docker run hello-world
# Cannot connect to the Docker daemon at unix:///var/run/docker.sock
```

`brew install docker` only installs the CLI. You still need a daemon running somewhere. On Linux that's just `systemctl start docker`, but on macOS you need something to run the Linux VM that hosts the daemon.

### Solution: Colima

Docker Desktop works, but it's a heavy GUI application. [Colima](https://github.com/abiosoft/colima) is a lightweight CLI alternative that spins up a Lima VM and exposes a Docker socket:

```bash
brew install colima
colima start
```

After `colima start` finishes (it takes about 30 seconds the first time), `docker` commands start working:

```bash
docker run hello-world
# Hello from Docker!
```

### Wiring It Up

Now the MCP add command works:

```bash
claude mcp add github \
  -e GITHUB_PERSONAL_ACCESS_TOKEN=<your_token> \
  -- docker run -i --rm -e GITHUB_PERSONAL_ACCESS_TOKEN ghcr.io/github/github-mcp-server
```

Verify with `claude mcp list`:

```
github: docker run -i --rm -e GITHUB_PERSONAL_ACCESS_TOKEN ghcr.io/github/github-mcp-server
```

## What Kind of Token Do You Need?

Use a **Personal Access Token (Classic)**, not a Fine-grained token. The GitHub MCP server works best with the classic format.

Required scopes:
- `repo` — full repository access (read + write)
- `read:org` — needed if you want to list organization repos
- `workflow` — required if you push to branches that contain GitHub Actions workflows (`.github/workflows/`)

> **Security note:** Be careful not to paste your token into the Claude Code chat window. Once it appears in the conversation, it's in the context. Generate a fresh token if you accidentally expose one.

## How It Works at Runtime

Each time Claude Code needs to call a GitHub MCP tool (e.g. `get_file_contents`, `push_files`), it:

1. Spawns `docker run -i --rm -e GITHUB_PERSONAL_ACCESS_TOKEN ghcr.io/github/github-mcp-server` as a subprocess
2. Communicates with it over stdio using the MCP protocol
3. The container makes GitHub API calls using the PAT from the environment variable
4. Results come back to Claude Code as structured tool outputs

The container is ephemeral (`--rm`) — it starts and stops for each session. That also means Colima must be running whenever you want the MCP server available. Add `colima start` to your shell startup if you use this regularly.

## What I Used It For

Once connected, Claude Code could:
- List all public and private repositories in my account
- Read files from a private repo
- Create and update files in a public repo
- All without me manually copying anything between repos

The specific task was migrating five Korean blog posts from the private draft repo to the public Hugo site repo, including translation and image handling. Having direct GitHub API access made it possible to fetch file contents and commit to the target repo without leaving the Claude Code session.

## Summary

| Approach | Result |
|---|---|
| Hosted endpoint (`https://api.githubcopilot.com/mcp/`) | Failed — doesn't support dynamic client registration |
| Bearer token header on hosted endpoint | Failed — auth flow still triggered |
| Docker stdio with `ghcr.io/github/github-mcp-server` | **Works** |

The self-hosted Docker approach requires two extra pieces of setup (a Docker daemon and a PAT), but it's the only method that actually works with Claude Code's MCP client today.

## References

- [GitHub MCP Server](https://github.com/github/github-mcp-server)
- [Colima — Container runtimes on macOS](https://github.com/abiosoft/colima)
- [Model Context Protocol](https://modelcontextprotocol.io/)
