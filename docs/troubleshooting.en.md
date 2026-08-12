# Troubleshooting

[Versi Bahasa Indonesia](troubleshooting.md)

## 1. `curl: not found`

Some OpenCode images may not include `curl`.

Check:

```bash
docker compose run --rm   --entrypoint sh   opencode   -c 'command -v curl || command -v wget || command -v nc || true'
```

If `wget` is available:

```bash
wget -qO- http://9router:20128/v1/models
```

## 2. `jq: parse error`

If the response looks like:

```text
data: {...}
data: {...}
data: {...}
```

the endpoint is using SSE streaming.

Use:

```bash
curl -N ...
```

Do not treat the entire stream as one JSON object.

## 3. `getent hosts 9router` Fails

Test:

```bash
docker compose run --rm   --entrypoint sh   opencode   -c 'getent hosts 9router'
```

If it fails:

```bash
docker network inspect ai-network
```

Make sure both Compose files declare:

```yaml
networks:
  - ai-network
```

and:

```yaml
networks:
  ai-network:
    external: true
```

## 4. 9Router Is Missing from the Network

```bash
docker network inspect ai-network   --format '{{range .Containers}}{{.Name}} -> {{.IPv4Address}}{{println}}{{end}}'
```

If `9router` is absent, verify it is running:

```bash
docker compose ps
```

Recreate if necessary:

```bash
docker compose up -d --force-recreate
```

## 5. Port 20128 Is Already in Use

```bash
ss -ltnp | grep ':20128'
```

During migration use a pilot mapping:

```yaml
ports:
  - "127.0.0.1:20129:20128"
```

After the old service is stopped:

```yaml
ports:
  - "127.0.0.1:20128:20128"
```

## 6. `/v1/models` Does Not Show the Combo

Check that:

- the correct database/state is mounted;
- the combo still exists in the 9Router dashboard;
- the API key is valid;
- the container is using the expected volume.

Inspect:

```bash
docker inspect 9router
```

Logs:

```bash
docker logs --tail 100 9router
```

## 7. Request Returns `401`

Possible causes:

- wrong API key;
- a new container did not receive the updated `.env`;
- the old key was revoked.

Check without printing the key:

```bash
grep -q '^ROUTER9_API_KEY=' .env   && echo "key exists"   || echo "key missing"
```

Then start a fresh OpenCode session.

## 8. OpenCode Still Uses the Host Endpoint

Check:

```bash
grep -n "baseURL" opencode.json
```

Expected:

```text
http://9router:20128/v1
```

Not:

```text
http://host.docker.internal:20128/v1
```

## 9. OpenCode Cannot See the Project

Verify:

```yaml
volumes:
  - ../:/workspace
```

and:

```yaml
working_dir: /workspace
```

Test:

```bash
docker compose run --rm   --entrypoint sh   opencode   -c 'pwd && ls -la'
```

## 10. OpenCode Container Disappears After Exit

If it was started with:

```bash
docker compose run --rm
```

this is expected.

`--rm` removes the container when the process exits.

9Router remains persistent.

## 11. State Disappears After Restart

Check:

```bash
ls -lh ~/docker/9router/data/db/
```

Verify the Compose volume:

```yaml
volumes:
  - ./data:/app/data
```

Restart:

```bash
docker restart 9router
```

then test `/v1/models` again.

## 12. Minimal Network Debug Sequence

```bash
docker compose ps

docker network inspect ai-network

docker compose run --rm   --entrypoint sh   opencode   -c 'getent hosts 9router'

docker logs --tail 100 9router
```

Use this sequence before changing multiple configuration variables at once.
