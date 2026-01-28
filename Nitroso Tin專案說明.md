# Nitroso Tin 專案說明

## 專案概述

**Nitroso Tin** 是一個用 Go 語言開發的**火車票自動訂票系統**，專門用於馬來西亞 KTMB (Keretapi Tanah Melayu Berhad) 火車票的自動化訂購。

### 核心功能

這個專案提供了一個完整的火車票自動訂購流程，包含以下主要功能：

1. **監控票務變化** (CDC - Change Data Capture)
2. **輪詢票務可用性** (Poller)
3. **豐富票務資訊** (Enricher)
4. **預訂車票** (Reserver)
5. **購買車票** (Buyer)
6. **終止和清理** (Terminator)

## 系統架構

### 整體架構流程

```
CDC (監控變化) 
  ↓
Poller (輪詢可用性)
  ↓
Enricher (豐富資訊)
  ↓
Reserver (預訂)
  ↓
Buyer (購買)
  ↓
Terminator (清理)
```

### 核心組件說明

#### 1. **CDC (Change Data Capture)** - `cmds/cdc.go`
- **功能**: 監控外部系統的變化，將變化數據捕獲並推送到 Redis Stream
- **位置**: `lib/cdc/`
- **用途**: 作為整個流程的起點，監聽票務系統的變化

#### 2. **Poller** - `cmds/poller.go`
- **功能**: 使用 Kubernetes Job 輪詢票務可用性
- **位置**: `lib/poller/`
- **用途**: 定期檢查是否有可用的車票

#### 3. **Enricher** - `cmds/enricher.go`
- **功能**: 豐富票務數據，添加額外資訊（如用戶數據、商店資訊）
- **位置**: `lib/enricher/`
- **用途**: 為預訂流程準備完整的數據

#### 4. **Reserver** - `cmds/reserver.go`
- **功能**: 處理車票預訂邏輯
- **位置**: `lib/reserver/`
- **用途**: 執行實際的預訂操作，包含並發控制和重試機制

#### 5. **Buyer** - `cmds/buyer.go`
- **功能**: 執行實際的購票操作
- **位置**: `lib/buyer/`
- **用途**: 完成最終的購票流程，上傳票據文件

#### 6. **Terminator** - `cmds/terminator.go`
- **功能**: 終止和清理流程
- **位置**: `lib/terminator/`
- **用途**: 清理資源，終止相關進程

### 技術架構

#### 數據流
- **Redis Streams**: 用於組件間的異步消息傳遞
- **Redis Cache**: 用於數據緩存和狀態管理
- **Kubernetes Jobs**: 用於 Poller 的任務調度

#### 外部依賴
- **KTMB API**: 馬來西亞火車票系統的 API (`lib/ktmb/`)
- **Zinc API**: 內部 API 服務 (`lib/zinc/`)
- **Descope**: 用於身份驗證 (`lib/auth/`)

#### 可觀測性
- **OpenTelemetry**: 完整的遙測系統（指標、追蹤、日誌）
- **Zerolog**: 結構化日誌記錄

## 專案結構

```
nitroso.tin-main/
├── main.go                 # 應用程式入口點
├── cmds/                    # CLI 命令實現
│   ├── cdc.go
│   ├── poller.go
│   ├── enricher.go
│   ├── reserver.go
│   ├── buyer.go
│   ├── terminator.go
│   └── state.go            # 共享狀態
├── lib/                     # 核心業務邏輯庫
│   ├── auth/               # 身份驗證
│   ├── buyer/              # 購票邏輯
│   ├── cdc/                # 變化數據捕獲
│   ├── enricher/           # 數據豐富化
│   ├── ktmb/               # KTMB API 客戶端
│   ├── poller/             # 輪詢邏輯
│   ├── reserver/           # 預訂邏輯
│   ├── terminator/         # 終止邏輯
│   ├── encryptor/          # 加密工具
│   ├── otelredis/          # Redis 集成
│   └── zinc/               # Zinc API 客戶端
├── system/                  # 系統級功能
│   ├── config/             # 配置管理
│   └── telemetry/          # 可觀測性
├── config/                  # 配置文件
│   └── app/
│       ├── settings.yaml           # 基礎配置
│       ├── settings.lapras.yaml    # 環境特定配置
│       ├── settings.pichu.yaml
│       ├── settings.pikachu.yaml
│       └── settings.raichu.yaml
├── infra/                   # 基礎設施配置
│   ├── Dockerfile          # Docker 構建文件
│   └── consumer_chart/     # Kubernetes Helm Chart
└── scripts/                 # 工具腳本
```

## 如何在個人專案中使用

### 需要修改的文件清單

#### 1. **配置文件** (必須修改)

##### `config/app/settings.yaml`
這是主要的配置文件，需要根據你的環境修改：

```yaml
# Redis 配置
cache:
  MAIN:
    endpoints:
      0: 'your-redis-host:6379'  # 修改為你的 Redis 地址
  
# KTMB API 配置
ktmb:
  apiUrl: https://online-api.ktmb.com.my  # 確認 API 地址
  appUrl: https://shuttleonline-api.ktmb.com.my
  loginKey: 'login-session'  # 根據需要修改

# 購票配置
buyer:
  contactNumber: '+6581272251'  # 修改為你的聯繫電話
  scheme: http
  host: localhost
  port: 9000

# Enricher 配置
enricher:
  email: kirinnee97@gmail.com  # 修改為你的郵箱
  password: ''  # 設置你的密碼

# 加密密鑰
encryptor:
  key: ''  # 必須設置加密密鑰

# 應用程式識別
app:
  platform: nitroso  # 可修改為你的平台名稱
  service: tin       # 可修改為你的服務名稱
```

##### `config/app/settings.{landscape}.yaml`
根據你的環境（lapras, pichu, pikachu, raichu）修改對應的配置文件，或創建新的環境配置。

#### 2. **身份驗證配置** (必須修改)

##### `config/app/settings.yaml` 中的 `auth` 部分
需要配置 Descope 認證信息（如果使用）：
```yaml
auth:
  descope:
    descopeId: ''        # 你的 Descope ID
    descopeAccessKey: '' # 你的 Descope Access Key
```

#### 3. **環境變量配置**

創建或修改 `.envrc` 或環境變量：
```bash
LANDSCAPE=lapras          # 選擇環境
BASE_CONFIG=./config/app  # 配置路徑
ENCRYPTION_KEY=your-key   # 加密密鑰
```

#### 4. **KTMB API 集成** (可能需要修改)

##### `lib/ktmb/`
如果 KTMB API 有變化，可能需要修改：
- `lib/ktmb/ktmb.go` - API 客戶端實現
- `lib/ktmb/req.go` - 請求結構
- `lib/ktmb/res.go` - 響應結構

#### 5. **Zinc API 集成** (可能需要修改)

##### `lib/zinc/`
如果使用不同的內部 API，需要修改：
- `lib/zinc/main.go` - API 客戶端

#### 6. **應用程式識別** (可選修改)

##### `main.go`
修改應用程式名稱和模組識別：
```go
psm := fmt.Sprintf("%s.%s.%s", cfgApp.Platform, cfgApp.Service, cfgApp.Module)
```

##### `go.mod`
修改模組名稱（如果需要的話）：
```go
module github.com/AtomiCloud/nitroso-tin
// 改為你的模組路徑
```

#### 7. **基礎設施配置** (如果部署到 Kubernetes)

##### `infra/consumer_chart/`
- `values.yaml` - Helm Chart 值
- `templates/` - Kubernetes 模板

##### `infra/root_chart/`
- `values.yaml` - 根 Chart 配置

### 快速開始步驟

1. **克隆並設置環境**
   ```bash
   # 設置環境變量
   export LANDSCAPE=lapras
   export BASE_CONFIG=./config/app
   export ENCRYPTION_KEY=your-encryption-key
   ```

2. **修改配置文件**
   - 編輯 `config/app/settings.yaml`
   - 設置 Redis 連接信息
   - 設置 KTMB API 配置
   - 設置身份驗證信息

3. **安裝依賴**
   ```bash
   go mod download
   ```

4. **構建應用**
   ```bash
   task build
   # 或
   go build -o bin/nitroso-tin .
   ```

5. **運行特定組件**
   ```bash
   # 運行 CDC
   ./bin/nitroso-tin cdc
   
   # 運行 Poller
   ./bin/nitroso-tin poller
   
   # 運行 Enricher
   ./bin/nitroso-tin enricher
   
   # 運行 Reserver
   ./bin/nitroso-tin reserver
   
   # 運行 Buyer
   ./bin/nitroso-tin buyer
   
   # 運行 Terminator
   ./bin/nitroso-tin terminator
   ```

### 開發模式

使用 Task 工具運行（推薦）：
```bash
# 設置環境
task setup

# 開發模式運行
task dev -- cdc

# 監視模式（自動重載）
task dev:watch -- cdc
```

## 重要注意事項

1. **加密密鑰**: 必須設置 `ENCRYPTION_KEY` 環境變量，用於加密敏感數據
2. **Redis 連接**: 確保 Redis 服務可用，並且配置正確
3. **KTMB API**: 確認 API 端點和認證方式是否正確
4. **身份驗證**: 如果使用 Descope，需要正確配置認證信息
5. **環境配置**: 根據你的部署環境選擇正確的 landscape 配置

## 總結

這個專案是一個完整的火車票自動訂購系統，採用微服務架構，使用 Redis Streams 進行組件間通信。要在個人專案中使用，主要需要：

1. **必須修改**: 配置文件（`config/app/settings.yaml`）
2. **必須設置**: 環境變量（加密密鑰、Redis 連接等）
3. **可能需要修改**: API 集成部分（如果 API 有變化）
4. **可選修改**: 應用程式名稱和模組識別

建議先從配置文件開始，逐步測試每個組件，確保所有依賴服務（Redis、KTMB API 等）都正常運作。
