# Troubleshooting

[English version](troubleshooting.en.md)

## 1. `curl: not found`

Sebagian image OpenCode mungkin tidak memiliki `curl`.

Cek:

```bash
docker compose run --rm   --entrypoint sh   opencode   -c 'command -v curl || command -v wget || command -v nc || true'
```

Jika `wget` tersedia:

```bash
wget -qO- http://9router:20128/v1/models
```

## 2. `jq: parse error`

Jika respons terlihat seperti:

```text
data: {...}
data: {...}
data: {...}
```

endpoint menggunakan SSE streaming.

Gunakan:

```bash
curl -N ...
```

Jangan memperlakukan seluruh stream sebagai satu JSON object.

## 3. `getent hosts 9router` gagal

Test:

```bash
docker compose run --rm   --entrypoint sh   opencode   -c 'getent hosts 9router'
```

Jika gagal, periksa:

```bash
docker network inspect ai-network
```

Pastikan kedua Compose mendeklarasikan:

```yaml
networks:
  - ai-network
```

dan:

```yaml
networks:
  ai-network:
    external: true
```

## 4. 9Router Tidak Muncul di Network

```bash
docker network inspect ai-network   --format '{{range .Containers}}{{.Name}} -> {{.IPv4Address}}{{println}}{{end}}'
```

Jika `9router` tidak muncul, pastikan container berjalan:

```bash
docker compose ps
```

Kemudian recreate bila diperlukan:

```bash
docker compose up -d --force-recreate
```

## 5. Port 20128 Sudah Digunakan

```bash
ss -ltnp | grep ':20128'
```

Saat migrasi, gunakan pilot:

```yaml
ports:
  - "127.0.0.1:20129:20128"
```

Setelah service lama berhenti:

```yaml
ports:
  - "127.0.0.1:20128:20128"
```

## 6. `/v1/models` Tidak Menampilkan Combo

Cek:

- database/state yang digunakan benar;
- dashboard 9Router masih berisi combo;
- API key valid;
- container membaca volume yang benar.

Cek mount:

```bash
docker inspect 9router
```

Cek log:

```bash
docker logs --tail 100 9router
```

## 7. Request Menghasilkan `401`

Kemungkinan:

- API key salah;
- `.env` belum dibaca container baru;
- key lama sudah direvoke.

Cek tanpa mencetak secret:

```bash
grep -q '^ROUTER9_API_KEY=' .env   && echo "key exists"   || echo "key missing"
```

Kemudian recreate OpenCode session.

## 8. OpenCode Masih Mengakses Host

Cek:

```bash
grep -n "baseURL" opencode.json
```

Target:

```text
http://9router:20128/v1
```

Bukan:

```text
http://host.docker.internal:20128/v1
```

## 9. OpenCode Tidak Melihat Project

Periksa volume:

```yaml
volumes:
  - ../:/workspace
```

dan:

```yaml
working_dir: /workspace
```

Cek:

```bash
docker compose run --rm   --entrypoint sh   opencode   -c 'pwd && ls -la'
```

## 10. Container OpenCode Hilang Setelah Exit

Jika dijalankan dengan:

```bash
docker compose run --rm
```

ini normal.

`--rm` memang menghapus container setelah proses berakhir.

9Router tetap persistent.

## 11. State Hilang Setelah Restart

Cek:

```bash
ls -lh ~/docker/9router/data/db/
```

Pastikan volume Compose:

```yaml
volumes:
  - ./data:/app/data
```

Lakukan test:

```bash
docker restart 9router
```

lalu cek kembali `/v1/models`.

## 12. Debug Network Minimum

Urutan cepat:

```bash
docker compose ps

docker network inspect ai-network

docker compose run --rm   --entrypoint sh   opencode   -c 'getent hosts 9router'

docker logs --tail 100 9router
```

Gunakan urutan ini sebelum mengubah banyak konfigurasi sekaligus.
