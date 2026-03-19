# Lago From Source - Datar-Tech 開發筆記

## Forked Repos

| Repo | URL | 用途 |
|------|-----|------|
| lago (主 repo) | https://github.com/Datar-Tech/lago | docker-compose、scripts、submodules |
| lago-api | https://github.com/Datar-Tech/lago-api | Ruby on Rails 後端 (git submodule → `./api`) |
| lago-front | https://github.com/Datar-Tech/lago-front | React 前端 (git submodule → `./front`) |
| lago-gotenberg | https://github.com/Datar-Tech/lago-gotenberg | PDF 產生服務 |

## 分支說明

| Repo | Branch | 改動 |
|------|--------|------|
| lago | `datar-tech/dev-setup` | `.gitmodules` 指向 fork、healthcheck timeout 調整、`.gitattributes` 強制 LF |
| lago-api | `datar-tech/fix-crlf` | `.gitattributes` 強制 shell scripts 用 LF |
| lago-front | `datar-tech/fix-crlf` | `.gitattributes` 強制 shell scripts 用 LF |

## 本地路徑

- `C:\Github\lago` — 主 repo（含 submodules）
- `C:\Github\lago-front` — 獨立 clone（非 submodule）
- `C:\Github\lago-gotenberg` — 獨立 clone

## 開發環境啟動

```bash
cd C:/Github/lago
docker compose -f docker-compose.dev.yml up -d
```

### 預設登入帳密
- Email: `email@example.com`
- Password: `password`

### 存取網址（需 hosts 設定 + mkcert SSL 憑證）
- Frontend: https://app.lago.dev
- API: https://api.lago.dev
- Traefik: https://traefik.lago.dev
- Mailhog: https://mail.lago.dev
- Redpanda Console: https://console.lago.dev
- PgHero: https://pghero.lago.dev
- Webhook Tester: https://webhook.lago.dev

### hosts 設定（C:\Windows\System32\drivers\etc\hosts）
```
127.0.0.1 app.lago.dev api.lago.dev traefik.lago.dev mail.lago.dev pdf.lago.dev console.lago.dev pghero.lago.dev webhook.lago.dev
```

### SSL 憑證（mkcert）
```bash
# 安裝 CA（需管理員權限，只需一次）
mkcert -install

# 產生憑證
cd C:/Github/lago/traefik/certs
mkcert -cert-file lago.dev.pem -key-file lago.dev-key.pem "*.lago.dev" lago.dev
```

## Windows 注意事項

- Shell scripts 的 CRLF 問題已透過 `.gitattributes` 解決
- API healthcheck timeout 已從 5s 調整為 30s（Windows Docker 較慢）
- `lago-gotenberg` clone 時有 `Zone.Identifier` 檔案問題，已用 sparse checkout 跳過

## 同步上游流程

```bash
# 1. 切回 main 同步上游
git checkout main
git remote add upstream https://github.com/getlago/lago.git  # 只需做一次
git fetch upstream
git merge upstream/main

# 2. 同步 submodules
git submodule update --remote

# 3. 將上游更新合併到 dev-setup 分支
git checkout datar-tech/dev-setup
git merge main
```

### lago-api / lago-front 同步
```bash
# 以 lago-api 為例
cd C:/Github/lago/api
git remote add upstream https://github.com/getlago/lago-api.git  # 只需一次
git fetch upstream
git checkout main && git merge upstream/main
git checkout datar-tech/fix-crlf && git merge main
```
