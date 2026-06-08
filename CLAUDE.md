# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

OpenList is a multi-storage file listing and aggregation server written in Go. It is a community-driven fork of AList, licensed under AGPL-3.0. The backend exposes a unified file-system interface over HTTP (REST API, WebDAV, S3-compatible, FTP, SFTP) and proxies or redirects traffic to numerous cloud storage providers.

The frontend is a separate SolidJS application living in `OpenListTeam/OpenList-Frontend`. It is not in this repository.

## Common Commands

### Run the backend locally
```bash
go run main.go server
```
Default HTTP port is `5244`. Config and database are stored under `./data/` by default.

### Build
```bash
# Quick local binary (no frontend embed required if public/dist exists)
go build -o openlist .

# Development build: fetch rolling frontend dist and build multi-platform binaries
bash build.sh dev

# Release build: fetch latest frontend release and build all targets
bash build.sh release
```
`build.sh` handles cross-compilation for Linux (glibc/musl), Windows (including Win7), macOS, Android, FreeBSD, and LoongArch (ABI 1.0/2.0). It also downloads the frontend distribution automatically.

### Tests
```bash
# All tests
go test ./...

# Single package
go test ./drivers/189/...
go test ./internal/casfile/...

# Single test
go test ./drivers/quark_uc_share -run TestLinkCache
```

### Development helpers
```bash
# Live reload (requires github.com/air-verse/air)
air

# Format before committing
go fmt ./...
```

## High-Level Architecture

### Entry & Bootstrap

- `main.go` → `cmd.Execute()` (Cobra CLI)
- `cmd/server.go` → `bootstrap.Init()` then `bootstrap.Start()`
- `bootstrap.Init()` initializes config, logging, database, seed data, stream limits, search index, and upgrade patches.
- `bootstrap.Start()` loads all configured storages from DB into memory, starts the task manager, then launches the HTTP/HTTPS/Unix/QUIC/S3/FTP/SFTP servers concurrently.

### Driver Model

The core abstraction is `driver.Driver` (`internal/driver/driver.go`). A driver is a storage backend instance composed of:

- `Meta`: config, storage record, Init/Drop lifecycle
- `Reader`: `List` and `Link` (required)
- Optional writer capabilities via separate interfaces: `Mkdir`, `Move`, `Rename`, `Copy`, `Remove`, `Put`, `PutURL`
- Optional archive interfaces: `ArchiveReader`, `ArchiveDecompress`
- Optional extras: `WithDetails`, `DirectUploader`, `LinkCacheModeResolver`

Drivers self-register in `init()` via `op.RegisterDriver()` (see `internal/op/driver.go`). `drivers/all.go` blank-imports every driver package so registration happens at startup.

**Adding a new driver:** copy `drivers/template/`, implement the required interfaces, and add a blank import in `drivers/all.go`.

### Storage Lifecycle (`internal/op/storage.go`)

Storages are persisted in the DB (GORM) and loaded into an in-memory map (`storagesMap`) keyed by `MountPath`. The main operations:

- `CreateStorage` → inserts DB row, calls `initStorage`
- `LoadStorage` → loads an existing DB row into memory
- `UpdateStorage` → drops then re-initializes the driver
- `EnableStorage` / `DisableStorage` → adds/removes from the live map
- `DeleteStorageById` → drops driver and deletes DB row

`initStorage` unmarshals the driver-specific `Addition` JSON into the driver struct, calls `Init(ctx)`, and stores the driver in `storagesMap`. Panics during init are recovered and stored as the storage status string.

### Path Resolution (`internal/fs/` and `internal/op/`)

`internal/fs/` exposes functions like `List`, `Get`, `Link`, `MakeDir`, `Move`, `Copy`, etc. These receive a **mount path** (the path the user sees) and delegate to `internal/op/` to resolve the actual storage and path within that storage.

`op.GetStorageAndActualPath(path)` performs longest-prefix matching against `storagesMap`. Multiple storages can share the same mount path; in that case `GetBalancedStorage` round-robins between them.

Virtual folders are generated when a mount path has parent paths that do not correspond to any storage (e.g. mounting `/a/b` creates a virtual folder `b` under `/a`).

### Object Model (`internal/model/`)

- `Obj` interface: `GetName`, `GetSize`, `ModTime`, `IsDir`, `GetID`, `GetPath`, `GetHash`
- `Object` is the concrete implementation.
- `FileStreamer` (`model/obj.go`) extends `Obj` with `io.Reader`, mime-type, progress callbacks, and caching utilities.
- `model.Link` holds download URLs, headers, expiration, and an optional `RangeReaderIF` for seekable remote streams.
- Wrappers like `ObjWrapName`, `ObjWrapMask`, `ObjStorageDetails` wrap `Obj` to alter behavior without changing the underlying driver.

### Stream & Upload (`internal/stream/`)

`FileStream` wraps an upload `io.Reader` with caching and progress support. It can:
- Peek via `RangeRead` without consuming the stream
- Cache to memory (with mmap threshold) or a temp file via `CacheFullAndWriter`
- Track upload progress through `ReaderUpdatingProgress`

Rate limiting and server-side upload limits are enforced here via `RateLimitReader` / `RateLimitFile`.

### Server Protocols (`server/`)

- `server/router.go`: Gin router initialization. Splits public API, auth-required API, admin API, WebDAV (`/dav`), S3 (`/s3`), and static frontend serving.
- `server/handles/`: HTTP API handlers for file ops, admin, auth, sharing, tasks, etc.
- `server/webdav/`: Custom WebDAV implementation with lock support.
- `server/s3/`: S3-compatible API backend.
- `server/ftp.go` / `server/sftp.go`: FTP and SFTP server wrappers over the same `fs` layer.

### Database & Configuration

- Database: GORM with SQLite3 by default; MySQL and Postgres supported.
- Config: `internal/conf/config.go` defines the `Config` struct. Values are loaded from `data/config.json` and can be overridden by environment variables.
- Settings key/value store in DB for runtime tunables.

### Task System (`pkg/task/` and `internal/task/`)

Background operations (copy, move, upload, decompress, offline download) are submitted as tasks. `pkg/task/manager.go` provides the core queue/worker logic. Task state is persisted if configured.

### Caching

- Directory listings and storage details are cached in-memory (`internal/op/cache.go`) with configurable TTL per storage.
- Some share drivers (`QuarkUCShare`, `Cloud189Share`, `BaiduShare2`, `ThunderShare`, `AliyundriveShare2Open`) maintain a local in-process link cache (`drivers/*/driver.go`) to avoid repeated expensive share-to-account save operations. TTL is typically 60 minutes.
- `driver.LinkCacheMode` controls whether link cache keys include client IP or User-Agent.

### CAS File Support (`internal/casfile/`)

`.cas` files are lightweight metadata stubs used for rapid upload restore on certain providers.

- `internal/casfile/cas.go` parses `.cas` payloads (base64 or raw JSON with `md5` and `sliceMd5`).
- `drivers/local`: can generate `.cas` after upload and optionally delete the original file.
- `drivers/189` and `drivers/189pc`: support `.cas` generation after upload, restoring source files from uploaded `.cas` files via provider rapid-upload APIs, and background auto-restore of existing `.cas` files in watched directories.

### Offline Download (`internal/offline_download/`)

Offline download tools (Aria2, qBittorrent, Transmission, and various cloud-native tools) are registered via blank imports in `internal/offline_download/all.go`. Each tool implements the offline download interface and is orchestrated by the task system.

### Search Index

Two backends are supported:
- Bleve (local index directory)
- Meilisearch (external service)

Index building and search are exposed under `/api/admin/index` and `/api/fs/search`.

## Important Conventions

- **Go version:** 1.24+ (see `go.mod`).
- **Build tags:** `jsoniter` is the default tag. Some targets use `sqlite_cgo_compat` for CGO sqlite compatibility.
- **Commits:** Use Conventional Commits for PR titles. Format code with `go fmt` before submitting.
- **Frontend:** Changes to the UI require the separate `OpenList-Frontend` repository. This backend repo only consumes frontend release tarballs during build.
- **No `.golangci.yml`:** There is no configured linter beyond `go fmt` and `go vet`.
- **Superpowers specs:** Architecture and design specs for recent features are stored in `docs/superpowers/specs/` and can be referenced for context on CAS integration, link caching, and driver-specific designs.
