# OpenCode + 9Router di Docker

> Panduan arsitektur multi-project AI coding menggunakan **OpenCode per project** dan **9Router sebagai AI gateway terpusat** di Docker.

[English version](README.en.md)

## Gambaran Singkat

Arsitektur ini dirancang untuk memisahkan agent coding per project, sambil tetap memusatkan provider, routing model, fallback, dan API gateway melalui satu instance 9Router.

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

Alur request:

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

## Tujuan

Arsitektur ini ditujukan untuk:

- mengisolasi OpenCode per project;
- menghindari akses agent ke seluruh host;
- memusatkan konfigurasi provider dan routing;
- menggunakan Docker DNS, bukan hardcoded IP;
- membuat OpenCode bersifat ephemeral;
- membuat 9Router persistent;
- mempermudah pengelolaan melalui Docker atau Portainer;
- mendukung banyak project tanpa menduplikasi konfigurasi provider.

## Konsep Utama

### One Gateway

Semua OpenCode mengarah ke satu 9Router.

### One OpenCode per Project

Setiap project memiliki konfigurasi dan container OpenCode sendiri.

### Ephemeral Agent

OpenCode dijalankan hanya saat dibutuhkan:

```bash
docker compose run --rm   --name opencode-myproject   opencode
```

### Persistent Gateway

9Router berjalan sebagai service:

```yaml
restart: unless-stopped
```

### Docker DNS

OpenCode mengakses:

```text
http://9router:20128/v1
```

bukan IP container.

## Prasyarat

```bash
docker --version
docker compose version
```

Direkomendasikan:

```bash
sqlite3 --version
jq --version
```

## Quick Start

### 1. Buat external network

```bash
docker network inspect ai-network >/dev/null 2>&1 || docker network create ai-network
```

### 2. Siapkan 9Router

Gunakan contoh di:

```text
examples/9router/
```

Salin environment:

```bash
cp examples/9router/.env.example examples/9router/.env
```

Jalankan:

```bash
cd examples/9router
docker compose up -d
```

### 3. Konfigurasikan provider dan combo

Melalui dashboard lokal:

```text
http://127.0.0.1:20128
```

Contoh combo:

```text
coding-fast
coding-deep
```

### 4. Siapkan OpenCode pada project

Contoh struktur:

```text
MyProject/
├── src/
├── package.json
└── opencode/
    ├── compose.yml
    ├── opencode.json
    └── .env
```

Gunakan konfigurasi pada:

```text
examples/opencode/
```

### 5. Jalankan OpenCode

```bash
cd MyProject/opencode

docker compose run --rm   --name opencode-myproject   opencode
```

## Contoh Provider OpenCode

Bagian penting pada `opencode.json`:

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

## Pengujian Dasar

Cek Docker DNS:

```bash
docker compose run --rm   --entrypoint sh   opencode   -c 'getent hosts 9router'
```

Cek model dari host:

```bash
curl -s http://127.0.0.1:20128/v1/models   -H "Authorization: Bearer $ROUTER9_API_KEY" |
  jq -r '.data[].id'
```

## Dokumentasi

- [Arsitektur](docs/architecture.md)
- [Migrasi dari Host ke Docker](docs/migration.md)
- [Keamanan](docs/security.md)
- [Troubleshooting](docs/troubleshooting.md)

Versi Inggris:

- [Architecture](docs/architecture.en.md)
- [Migration](docs/migration.en.md)
- [Security](docs/security.en.md)
- [Troubleshooting](docs/troubleshooting.en.md)

## Security Checklist

Sebelum repository dipublikasikan:

- jangan commit `.env`;
- jangan commit API key atau token;
- jangan commit database SQLite;
- bind port 9Router ke `127.0.0.1` jika akses LAN tidak diperlukan;
- aktifkan API key enforcement;
- jangan mount Docker socket ke OpenCode;
- mount hanya workspace project yang dibutuhkan.

## Struktur Repository

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

## Catatan

Contoh konfigurasi di repository ini harus dianggap sebagai template. Sesuaikan model combo, provider, credential, dan batas akses dengan kebutuhan masing-masing environment.
