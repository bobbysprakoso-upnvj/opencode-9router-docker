# Migrasi 9Router dari Host ke Docker

[English version](migration.en.md)

Dokumen ini menjelaskan migrasi aman ketika 9Router sebelumnya sudah berjalan langsung di host.

## Prinsip

Jangan langsung menghapus instance host.

Gunakan alur:

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

## 1. Siapkan Direktori

```bash
mkdir -p ~/docker/9router/data
cd ~/docker/9router
```

## 2. Backup State Host

Jika state berada pada:

```text
~/.9router
```

backup:

```bash
mkdir -p ~/backup-9router-host

cp -a ~/.9router   ~/backup-9router-host/9router-$(date +%Y%m%d-%H%M%S)
```

## 3. Copy State Awal

```bash
cp -a ~/.9router/. ~/docker/9router/data/
```

Jika database aktif, Anda dapat melihat:

```text
data.sqlite
data.sqlite-wal
data.sqlite-shm
```

Jangan gunakan copy tersebut sebagai snapshot final.

## 4. Buat Snapshot SQLite

```bash
sqlite3 ~/.9router/db/data.sqlite   ".backup '$HOME/docker/9router/data/db/data.sqlite.new'"
```

Validasi:

```bash
sqlite3 ~/docker/9router/data/db/data.sqlite.new   "PRAGMA integrity_check;"
```

Target:

```text
ok
```

Kemudian:

```bash
mv ~/docker/9router/data/db/data.sqlite.new    ~/docker/9router/data/db/data.sqlite

rm -f ~/docker/9router/data/db/data.sqlite-wal
rm -f ~/docker/9router/data/db/data.sqlite-shm
```

## 5. Buat Network

```bash
docker network inspect ai-network >/dev/null 2>&1 || docker network create ai-network
```

## 6. Jalankan Pilot pada Port Sementara

Gunakan:

```yaml
ports:
  - "127.0.0.1:20129:20128"
```

Sehingga:

```text
Host lama    -> :20128
Docker pilot -> :20129
```

## 7. Test `/v1/models`

```bash
curl -s http://127.0.0.1:20129/v1/models   -H "Authorization: Bearer $ROUTER9_API_KEY"
```

Pastikan combo yang dibutuhkan muncul.

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

Ulangi request dengan:

```json
"model": "coding-deep"
```

## 10. Test Persistence

```bash
docker restart 9router-pilot
sleep 5
```

Ulangi `/v1/models` dan satu request nyata.

## 11. Hubungkan OpenCode

OpenCode menggunakan:

```text
http://9router:20128/v1
```

melalui `ai-network`.

Verifikasi DNS:

```bash
docker compose run --rm   --entrypoint sh   opencode   -c 'getent hosts 9router'
```

## 12. Cutover

Pastikan host lama tidak listen:

```bash
ps aux | grep '[9]router'
ss -ltnp | grep ':20128'
```

Ubah Compose Docker:

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

## 13. Verifikasi Production

```bash
docker compose ps

curl -s http://127.0.0.1:20128/v1/models   -H "Authorization: Bearer $ROUTER9_API_KEY"
```

Lakukan juga test nyata `coding-fast` dan `coding-deep`.

## 14. Cleanup Host

Setelah Docker stabil:

```bash
npm uninstall -g 9router
```

Jangan langsung hapus state lama. Rename sebagai rollback backup:

```bash
mv ~/.9router ~/.9router-host-backup
```

Hapus hanya setelah deployment Docker stabil dalam periode yang dianggap cukup aman.

## 15. Rotate Credential

Jika API key pernah tampil di terminal, screenshot, chat, log, atau repository, anggap key tersebut terekspos.

Urutan:

```text
buat key baru
 ↓
update OpenCode .env
 ↓
test
 ↓
revoke key lama
```
