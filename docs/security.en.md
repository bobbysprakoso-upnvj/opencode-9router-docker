# Security

[Versi Bahasa Indonesia](security.md)

## Principle

Containerization is not automatically secure. The goal of this setup is to reduce blast radius and keep secrets, workspaces, and the gateway appropriately scoped.

## 1. Never Commit `.env`

Use:

```gitignore
.env
.env.*
!.env.example
```

A public example should look like:

```env
ROUTER9_API_KEY=replace_with_your_9router_api_key
```

Never put a real key in `.env.example`.

## 2. Do Not Commit 9Router Runtime State

Ignore:

```gitignore
data/
**/data/
*.sqlite
*.sqlite-wal
*.sqlite-shm
```

The database may contain configuration or sensitive runtime state.

## 3. Bind the Host Port to Loopback

If LAN access is unnecessary:

```yaml
ports:
  - "127.0.0.1:20128:20128"
```

This is more conservative than:

```yaml
ports:
  - "20128:20128"
```

## 4. Enforce API Keys

Example:

```env
REQUIRE_API_KEY=true
```

Clients must send:

```text
Authorization: Bearer <key>
```

## 5. Rotate Exposed Keys

Rotate a key if it appears in:

- chat;
- screenshots;
- Git history;
- public issues;
- accidental messages;
- public logs.

Use:

```text
create new key
 ↓
update client
 ↓
test new key
 ↓
revoke old key
```

## 6. Do Not Mount the Entire Home Directory

Avoid:

```yaml
volumes:
  - /home/user:/workspace
```

Prefer a project-scoped mount:

```yaml
volumes:
  - ../:/workspace
```

## 7. Do Not Mount the Docker Socket

Avoid:

```yaml
- /var/run/docker.sock:/var/run/docker.sock
```

The Docker socket can provide extensive control over the host.

Do not grant it unless the use case explicitly requires it and the risk is understood.

## 8. Mount Configuration Read-only

```yaml
- ./opencode.json:/root/.config/opencode/opencode.json:ro
```

## 9. Do Not Hardcode Secrets in JSON

Use:

```json
"apiKey": "{env:ROUTER9_API_KEY}"
```

instead of embedding a real key.

## 10. Be Careful with `docker compose config`

This command may print interpolated environment values:

```bash
docker compose config
```

To verify that a key exists without printing it:

```bash
grep -q '^ROUTER9_API_KEY=' .env   && echo "ROUTER9_API_KEY exists"   || echo "ROUTER9_API_KEY missing"
```

## 11. Scan for Secrets Before Push

A simple manual check:

```bash
grep -RniE   'sk-[A-Za-z0-9_-]{10,}|api[_-]?key[=:][^ ]+|bearer [A-Za-z0-9._-]+'   .   --exclude-dir=.git
```

Review each match. Placeholders such as:

```text
replace_with_your_key
```

are expected.

## 12. Check for Sensitive Files

```bash
find . -type f \(   -name '.env' -o   -name '*.sqlite' -o   -name '*.sqlite-wal' -o   -name '*.sqlite-shm' \)
```

A public repository should normally contain none of these runtime files.

## 13. Backups

Treat 9Router backups as sensitive.

Keep them outside the repository:

```text
~/backup-9router-docker/
```

For an active database:

```bash
sqlite3 ~/docker/9router/data/db/data.sqlite   ".backup '$HOME/backup-9router-docker/9router-backup.sqlite'"
```

## Security Checklist

Before pushing:

- [ ] no `.env` files;
- [ ] no real API keys;
- [ ] no runtime SQLite database;
- [ ] no Docker socket mount;
- [ ] 9Router is loopback-bound when appropriate;
- [ ] API key enforcement is enabled;
- [ ] OpenCode mounts only the intended project;
- [ ] previously exposed credentials have been rotated.
