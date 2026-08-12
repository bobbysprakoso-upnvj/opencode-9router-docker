# Architecture

[Versi Bahasa Indonesia](architecture.md)

## Design Goal

The architecture separates two responsibilities:

1. **OpenCode** acts as a project-scoped coding agent.
2. **9Router** acts as the centralized AI gateway for providers, routing, model combos, fallback, and API access.

Their lifecycle is therefore intentionally different:

```text
OpenCode  -> ephemeral / per project
9Router   -> persistent / shared
```

## Diagram

```text
┌──────────────────────────────────────────┐
│ HOST                                     │
│                                          │
│ Browser / IDE                            │
│                                          │
│ Docker Engine                            │
│   │                                      │
│   └── ai-network                         │
│       │                                  │
│       ├── 9router                        │
│       │    ├── coding-fast               │
│       │    ├── coding-deep               │
│       │    └── providers                 │
│       │                                  │
│       ├── opencode-project-a             │
│       ├── opencode-project-b             │
│       └── opencode-project-c             │
└──────────────────────────────────────────┘
```

## Request Flow

```text
User
 ↓
OpenCode
 ↓
http://9router:20128/v1
 ↓
9Router
 ↓
Combo
 ↓
Provider
 ↓
Cloud model
```

## Why an External Network?

`ai-network` is an external Docker network so containers managed by different Compose projects can discover each other.

```bash
docker network create ai-network
```

Both Compose configurations declare:

```yaml
networks:
  ai-network:
    external: true
```

OpenCode can then use the hostname:

```text
9router
```

without knowing the container IP.

## Why Not `localhost`?

Inside the OpenCode container:

```text
localhost
```

means the OpenCode container itself.

The correct endpoint for another container is:

```text
http://9router:20128/v1
```

## Why Not Hardcode an IP?

An address such as:

```text
10.200.4.2
```

may change after a container recreate.

Docker DNS provides a stable service name:

```text
9router
```

## One OpenCode per Project

```text
ProjectA/
└── opencode/

ProjectB/
└── opencode/

ProjectC/
└── opencode/
```

Each instance mounts only its project workspace:

```yaml
volumes:
  - ../:/workspace
```

This keeps the coding agent scoped to the relevant project.

## Model Combos

### `coding-fast`

Suitable for:

- codebase exploration;
- small edits;
- documentation;
- lightweight debugging;
- routine coding work.

### `coding-deep`

Suitable for:

- architecture analysis;
- technical debt;
- large refactors;
- complex debugging;
- security review.

The combo name becomes a stable contract between OpenCode and 9Router. Providers behind the combo can change without editing every project.

## Scaling

Adding projects does not require adding another 9Router instance.

```text
Project A ─┐
Project B ─┼──> 9Router ───> Providers
Project C ─┘
```

Only a new project-scoped OpenCode configuration is required.

## Responsibility Boundary

### OpenCode

- reads the workspace;
- uses coding tools;
- sends model requests;
- executes project-scoped workflows.

### 9Router

- providers;
- routing;
- combos;
- quota/fallback;
- API access;
- centralized gateway configuration.

## Operational Principles

- keep 9Router persistent;
- run OpenCode only when needed;
- never store secrets in the repository;
- do not mount the entire home directory;
- do not provide Docker socket access by default.
