# Migrating 9Router from Host to Docker

[Versi Bahasa Indonesia](migration.md)

This guide describes a conservative migration path when 9Router already runs directly on the host.

## Principle

Do not remove the host instance first.

Use:

```text
Host 9Router
 ↓
Backup
 ↓
SQLite snapshot
 ↓
Docker pilot :20129
 ↓
Test
 ↓
Connect OpenCode
 ↓
Stop host
 ↓
Docker production :20128
```

## 1. Prepare the Directory

```bash
mkdir -p ~/docker/9router/data
cd ~/docker/9router
```

## 2. Back Up Host State

If state is stored in:

```text
~/.9router
```

create a backup:

```bash
mkdir -p ~/backup-9router-host

cp -a ~/.9router   ~/backup-9router-host/9router-$(date +%Y%m%d-%H%M%S)
```

## 3. Copy Initial State

```bash
cp -a ~/.9router/. ~/docker/9router/data/
```

An active SQLite database may include:

```text
data.sqlite
data.sqlite-wal
data.sqlite-shm
```

Do not treat a raw active copy as the final snapshot.

## 4. Create a Consistent SQLite Snapshot

```bash
sqlite3 ~/.9router/db/data.sqlite   ".backup '$HOME/docker/9router/data/db/data.sqlite.new'"
```

Validate:

```bash
sqlite3 ~/docker/9router/data/db/data.sqlite.new   "PRAGMA integrity_check;"
```

Expected:

```text
ok
```

Then:

```bash
mv ~/docker/9router/data/db/data.sqlite.new    ~/docker/9router/data/db/data.sqlite

rm -f ~/docker/9router/data/db/data.sqlite-wal
rm -f ~/docker/9router/data/db/data.sqlite-shm
```

## 5. Create the Network

```bash
docker network inspect ai-network >/dev/null 2>&1 || docker network create ai-network
```

## 6. Run a Pilot on a Temporary Port

Use:

```yaml
ports:
  - "127.0.0.1:20129:20128"
```

Result:

```text
Old host     -> :20128
Docker pilot -> :20129
```

## 7. Test `/v1/models`

```bash
curl -s http://127.0.0.1:20129/v1/models   -H "Authorization: Bearer $ROUTER9_API_KEY"
```

Verify the required combos are present.

## 8. Test `coding-fast`

```bash
curl -N http://127.0.0.1:20129/v1/chat/completions   -H "Authorization: Bearer $ROUTER9_API_KEY"   -H "Content-Type: application/json"   -d '{
    "model":"coding-fast",
    "messages":[
      {"role":"user","content":"Reply only with OK"}
    ]
  }'
```

## 9. Test `coding-deep`

Repeat using:

```json
"model": "coding-deep"
```

## 10. Test Persistence

```bash
docker restart 9router-pilot
sleep 5
```

Repeat `/v1/models` and at least one real completion request.

## 11. Connect OpenCode

OpenCode should use:

```text
http://9router:20128/v1
```

through `ai-network`.

Check DNS:

```bash
docker compose run --rm   --entrypoint sh   opencode   -c 'getent hosts 9router'
```

## 12. Cut Over

Confirm the old host service no longer owns port 20128:

```bash
ps aux | grep '[9]router'
ss -ltnp | grep ':20128'
```

Change the Docker deployment:

```yaml
container_name: 9router

ports:
  - "127.0.0.1:20128:20128"
```

Apply:

```bash
docker compose down
docker compose up -d
```

## 13. Validate Production

```bash
docker compose ps

curl -s http://127.0.0.1:20128/v1/models   -H "Authorization: Bearer $ROUTER9_API_KEY"
```

Also test both `coding-fast` and `coding-deep`.

## 14. Clean Up the Host

After Docker is stable:

```bash
npm uninstall -g 9router
```

Do not immediately delete the old state. Keep it as a rollback backup:

```bash
mv ~/.9router ~/.9router-host-backup
```

Remove it only after the Docker deployment has remained stable for an appropriate period.

## 15. Rotate Credentials

If a key was exposed in a terminal paste, screenshot, chat, log, or repository, treat it as compromised.

Use:

```text
create new key
 ↓
update OpenCode .env
 ↓
test
 ↓
revoke old key
```
