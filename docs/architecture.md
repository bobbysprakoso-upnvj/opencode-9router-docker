# Arsitektur

[English version](architecture.en.md)

## Tujuan Desain

Arsitektur ini memisahkan dua tanggung jawab:

1. **OpenCode** sebagai coding agent yang bekerja pada satu project.
2. **9Router** sebagai gateway AI terpusat untuk provider, routing, model combo, fallback, dan akses API.

Pemisahan ini membuat lifecycle keduanya berbeda:

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

## Mengapa External Network?

`ai-network` dibuat sebagai external Docker network agar container dari Compose project yang berbeda tetap dapat saling menemukan.

Contoh:

```bash
docker network create ai-network
```

Kedua Compose mendeklarasikan:

```yaml
networks:
  ai-network:
    external: true
```

Dengan demikian OpenCode dapat menggunakan hostname:

```text
9router
```

Docker akan melakukan service discovery tanpa hardcoded IP.

## Mengapa Tidak Menggunakan `localhost`?

Di dalam container OpenCode:

```text
localhost
```

merujuk ke container OpenCode itu sendiri.

Karena 9Router berada di container lain, endpoint yang benar adalah:

```text
http://9router:20128/v1
```

## Mengapa Tidak Hardcode IP?

IP seperti:

```text
10.200.4.2
```

dapat berubah setelah recreate container.

Docker DNS jauh lebih stabil:

```text
9router
```

## OpenCode per Project

Contoh:

```text
ProjectA/
└── opencode/

ProjectB/
└── opencode/

ProjectC/
└── opencode/
```

Masing-masing hanya mount workspace project terkait:

```yaml
volumes:
  - ../:/workspace
```

Ini membantu menjaga scope agent tetap terbatas.

## Model Combo

Strategi yang digunakan dapat dibagi menjadi dua kelas.

### `coding-fast`

Cocok untuk:

- eksplorasi codebase;
- perubahan kecil;
- dokumentasi;
- debugging ringan;
- pekerjaan coding rutin.

### `coding-deep`

Cocok untuk:

- analisis arsitektur;
- technical debt;
- refactoring besar;
- debugging kompleks;
- security review.

Nama combo adalah kontrak antara OpenCode dan 9Router. Provider di belakang combo dapat berubah tanpa mengubah konfigurasi setiap project.

## Scaling

Untuk menambah project baru, tidak perlu membuat instance 9Router baru.

```text
Project A ─┐
Project B ─┼──> 9Router ───> Providers
Project C ─┘
```

Yang ditambahkan hanya container/config OpenCode project baru.

## Batas Tanggung Jawab

### OpenCode bertanggung jawab atas

- membaca workspace;
- menggunakan coding tools;
- mengirim request model;
- menjalankan workflow project-scoped.

### 9Router bertanggung jawab atas

- provider;
- routing;
- combo;
- quota/fallback;
- API access;
- central gateway configuration.

## Prinsip Operasional

- 9Router tetap hidup.
- OpenCode hidup hanya selama sesi.
- Secret tidak disimpan di repository.
- Workspace tidak diperluas ke seluruh home directory.
- Docker socket tidak diberikan ke OpenCode secara default.
