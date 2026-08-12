# OpenCode + 9Router on Docker

> A multi-project AI coding architecture using **one OpenCode container per project** and **9Router as a centralized AI gateway** on Docker.

[Versi Bahasa Indonesia](README.md)

## Overview

This architecture isolates coding agents by project while centralizing providers, routing, fallback logic, and API access through a single 9Router instance.

```text
Host
│
├── Browser / IDE
└── Docker
    └── ai-network
        ├── 9router
        │   ├── coding-fast
        │   ├── coding-deep
        │   └── Cloud AI Providers
        ├── opencode-project-a
        ├── opencode-project-b
        └── opencode-project-c
```

Request flow:

```text
Project
  ↓
OpenCode Container
  ↓
ai-network
  ↓
http://9router:20128/v1
  ↓
9Router
  ↓
coding-fast / coding-deep
  ↓
Cloud AI Provider
```

## Goals

This architecture aims to:

- isolate OpenCode per project;
- prevent the coding agent from receiving unnecessary host-wide access;
- centralize provider and routing configuration;
- use Docker DNS instead of hardcoded container IPs;
- keep OpenCode ephemeral;
- keep 9Router persistent;
- simplify operations through Docker or Portainer;
- support multiple projects without duplicating provider configuration.

## Core Design Principles

### One Gateway

All OpenCode instances connect to a single 9Router service.

### One OpenCode per Project

Each project has its own OpenCode configuration and container.

### Ephemeral Agent

Run OpenCode only when needed:

```bash
docker compose run --rm   --name opencode-myproject   opencode
```

### Persistent Gateway

Run 9Router as a persistent service:

```yaml
restart: unless-stopped
```

### Docker DNS

OpenCode connects to:

```text
http://9router:20128/v1
```

rather than a container IP.

## Requirements

```bash
docker --version
docker compose version
```

Recommended:

```bash
sqlite3 --version
jq --version
```

## Quick Start

### 1. Create the external Docker network

```bash
docker network inspect ai-network >/dev/null 2>&1 || docker network create ai-network
```

### 2. Prepare 9Router

Use the example in:

```text
examples/9router/
```

Copy the environment template:

```bash
cp examples/9router/.env.example examples/9router/.env
```

Start the service:

```bash
cd examples/9router
docker compose up -d
```

### 3. Configure providers and combos

Open the local dashboard:

```text
http://127.0.0.1:20128
```

Example combos:

```text
coding-fast
coding-deep
```

### 4. Add OpenCode to a project

Example:

```text
MyProject/
├── src/
├── package.json
└── opencode/
    ├── compose.yml
    ├── opencode.json
    └── .env
```

Use:

```text
examples/opencode/
```

as the template.

### 5. Run OpenCode

```bash
cd MyProject/opencode

docker compose run --rm   --name opencode-myproject   opencode
```

## OpenCode Provider Example

The important part of `opencode.json` is:

```json
{
  "model": "9router/coding-fast",
  "small_model": "9router/coding-fast",
  "provider": {
    "9router": {
      "npm": "@ai-sdk/openai-compatible",
      "name": "9Router",
      "options": {
        "baseURL": "http://9router:20128/v1",
        "apiKey": "{env:ROUTER9_API_KEY}"
      }
    }
  }
}
```

## Basic Validation

Check Docker DNS:

```bash
docker compose run --rm   --entrypoint sh   opencode   -c 'getent hosts 9router'
```

Check models from the host:

```bash
curl -s http://127.0.0.1:20128/v1/models   -H "Authorization: Bearer $ROUTER9_API_KEY" |
  jq -r '.data[].id'
```

## Documentation

- [Architecture](docs/architecture.en.md)
- [Migration from Host to Docker](docs/migration.en.md)
- [Security](docs/security.en.md)
- [Troubleshooting](docs/troubleshooting.en.md)

Indonesian versions:

- [Arsitektur](docs/architecture.md)
- [Migrasi](docs/migration.md)
- [Keamanan](docs/security.md)
- [Troubleshooting](docs/troubleshooting.md)

## Security Checklist

Before publishing the repository:

- never commit `.env`;
- never commit API keys or tokens;
- never commit SQLite runtime databases;
- bind 9Router to `127.0.0.1` if LAN access is not required;
- enforce API key authentication;
- do not mount the Docker socket into OpenCode;
- mount only the project workspace that OpenCode needs.

## Repository Layout

```text
opencode-9router-docker/
├── README.md
├── README.en.md
├── docs/
│   ├── architecture.md
│   ├── architecture.en.md
│   ├── migration.md
│   ├── migration.en.md
│   ├── security.md
│   ├── security.en.md
│   ├── troubleshooting.md
│   └── troubleshooting.en.md
└── examples/
    ├── 9router/
    └── opencode/
```

## Note

Configuration files in this repository are templates. Adapt model combos, providers, credentials, and access boundaries to your own environment.
