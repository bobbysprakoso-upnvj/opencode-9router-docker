# Keamanan

[English version](security.en.md)

## Prinsip Utama

Arsitektur container bukan otomatis aman. Tujuan konfigurasi ini adalah mengurangi blast radius dan menjaga secret, workspace, serta gateway tetap terbatas.

## 1. Jangan Commit `.env`

Gunakan:

```gitignore
.env
.env.*
!.env.example
```

Contoh publik:

```env
ROUTER9_API_KEY=replace_with_your_9router_api_key
```

Jangan pernah gunakan key nyata pada `.env.example`.

## 2. Jangan Commit State 9Router

Ignore:

```gitignore
data/
**/data/
*.sqlite
*.sqlite-wal
*.sqlite-shm
```

Database dapat berisi konfigurasi atau state yang tidak layak dipublikasikan.

## 3. Bind Port ke Loopback

Jika tidak membutuhkan akses LAN:

```yaml
ports:
  - "127.0.0.1:20128:20128"
```

Ini lebih konservatif daripada:

```yaml
ports:
  - "20128:20128"
```

## 4. Gunakan API Key Enforcement

Contoh:

```env
REQUIRE_API_KEY=true
```

Semua client harus mengirim:

```text
Authorization: Bearer <key>
```

## 5. Rotate Key yang Terekspos

Key harus diganti jika:

- tampil di chat;
- masuk screenshot;
- masuk Git history;
- tampil di issue;
- dikirim ke orang lain secara tidak sengaja;
- tercatat pada log publik.

Urutan aman:

```text
create new key
 ↓
update client
 ↓
test new key
 ↓
revoke old key
```

## 6. Jangan Mount Seluruh Home Directory

Hindari:

```yaml
volumes:
  - /home/user:/workspace
```

Gunakan project scoped mount:

```yaml
volumes:
  - ../:/workspace
```

## 7. Jangan Mount Docker Socket

Hindari:

```yaml
- /var/run/docker.sock:/var/run/docker.sock
```

Docker socket memberikan kemampuan yang sangat besar terhadap host.

Jika kebutuhan agent belum benar-benar mensyaratkannya, jangan berikan akses tersebut.

## 8. Read-only Config

Konfigurasi OpenCode dapat dimount read-only:

```yaml
- ./opencode.json:/root/.config/opencode/opencode.json:ro
```

## 9. Jangan Hardcode Secret di JSON

Gunakan:

```json
"apiKey": "{env:ROUTER9_API_KEY}"
```

bukan:

```json
"apiKey": "real-secret-value"
```

## 10. Hati-hati dengan `docker compose config`

Perintah:

```bash
docker compose config
```

dapat menampilkan environment yang sudah diinterpolasi.

Untuk mengecek keberadaan key:

```bash
grep -q '^ROUTER9_API_KEY=' .env   && echo "ROUTER9_API_KEY exists"   || echo "ROUTER9_API_KEY missing"
```

## 11. Secret Scan Sebelum Push

Contoh pemeriksaan sederhana:

```bash
grep -RniE   'sk-[A-Za-z0-9_-]{10,}|api[_-]?key[=:][^ ]+|bearer [A-Za-z0-9._-]+'   .   --exclude-dir=.git
```

Review setiap hasil. Placeholder seperti:

```text
replace_with_your_key
```

boleh tetap ada.

## 12. Verifikasi File Sensitif

```bash
find . -type f \(   -name '.env' -o   -name '*.sqlite' -o   -name '*.sqlite-wal' -o   -name '*.sqlite-shm' \)
```

Repository publik idealnya tidak mengembalikan file runtime tersebut.

## 13. Backup

Backup 9Router harus dianggap sensitif.

Gunakan folder yang tidak termasuk repository:

```text
~/backup-9router-docker/
```

Untuk database aktif:

```bash
sqlite3 ~/docker/9router/data/db/data.sqlite   ".backup '$HOME/backup-9router-docker/9router-backup.sqlite'"
```

## Security Checklist

Sebelum push:

- [ ] tidak ada `.env`;
- [ ] tidak ada API key nyata;
- [ ] tidak ada SQLite runtime database;
- [ ] tidak ada Docker socket mount;
- [ ] 9Router bind ke loopback jika cukup;
- [ ] API key enforcement aktif;
- [ ] OpenCode hanya mount project terkait;
- [ ] credential yang pernah terekspos sudah di-rotate.
