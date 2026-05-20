---
title: "Envoy Proxy 核心架構深度解析"
date: 2026-05-15
tags: [envoy, proxy, xds, architecture, l7, filter-chain]
draft: false
---

## Envoy Proxy 簡介

**Envoy Proxy** 是由 Lyft 開發、CNCF 畢業的高效能 L4/L7 代理伺服器。以 C++ 撰寫，專為雲原生環境設計。

**核心特性**:
- 🚀 **高效能**: C++ 實作，非阻塞 I/O，多線程架構
- 🔄 **動態配置**: 通過 xDS API 實現零停機重新配置
- 📊 **原生可觀察性**: 內建分散式追蹤（Zipkin/Jaeger）、統計指標
- 🔌 **高度可擴展**: Filter Chain 架構 + WebAssembly (Wasm) 支援
- 🛡️ **安全**: 原生 TLS/mTLS、JWT 驗證、RBAC、外部授權

## 線程模型 (Threading Model)

Envoy 使用**多線程、非阻塞**的架構：

```
Main Thread ──負責──▶ xDS 通訊、健康檢查、統計聚合、管理介面
    │
    └── Worker Threads (數量 = CPU 核心數)
          ├── Worker 1: 監聽埠 A，處理連線
          ├── Worker 2: 監聽埠 B，處理連線
          └── Worker N: ...
```

- 每個 Worker 獨立綁定一個 listener socket (SO_REUSEPORT)
- Worker 之間幾乎**無鎖定（lock-free）**
- 連線一致性雜湊保證同一連線始終由同一 Worker 處理

## 四大核心組件

### 1. Listeners（監聽器）

Listener 是 Envoy 的網路入口點，定義監聽的 IP:Port。

```yaml
static_resources:
  listeners:
  - name: main_listener
    address:
      socket_address:
        address: 0.0.0.0
        port_value: 8443
    listener_filters:
    - name: envoy.filters.listener.tls_inspector
    filter_chains:
    - filter_chain_match:
        server_names: ["api.example.com"]
      filters:
      # ... HTTP 或 TCP 過濾器
```

**Listener Filter**: 在連線建立時最早執行的過濾器（如 TLS inspector 檢查 SNI）。

### 2. Filter Chains（過濾器鏈）

過濾器鏈是 Envoy 最強大的擴展機制。每個 Listener 可以有多個 Filter Chain，根據匹配條件（SNI、來源 IP 等）選擇。

```
Listener
  └── Filter Chain (根據 SNI / 來源 IP 匹配)
        ├── Network Filter 1 (e.g., TLS Inspector)
        ├── Network Filter 2 (e.g., HTTP Connection Manager)
        │     └── HTTP Filter 1 (e.g., JWT Auth)
        │     └── HTTP Filter 2 (e.g., Router)
        │           └── Route → Cluster
        └── Network Filter 3 (e.g., TCP Proxy)
```

**兩層過濾器**:

| 層級 | 類型 | 說明 |
|------|------|------|
| **L4 (Network)** | Network Filters | 處理 TCP 連線：TCP Proxy, HTTP Connection Manager, MongoDB, Redis, Postgres, Rate Limit, RBAC |
| **L7 (HTTP)** | HTTP Filters | 處理 HTTP 請求/回應：Router, JWT Auth, CORS, Fault Injection, Rate Limit, gRPC-JSON Transcoder |

### 3. Routes（路由）

HTTP Connection Manager 內的路由表，將請求對應到後端 Cluster。

```yaml
routes:
- name: api-routes
  virtual_hosts:
  - name: api
    domains: ["api.example.com"]
    routes:
    - match:
        prefix: "/v2/"
        headers:
        - name: x-canary
          exact_match: "true"
      route:
        cluster: api-canary
        timeout: 30s
        retry_policy:
          retry_on: "5xx,connect-failure"
          num_retries: 3
    - match:
        prefix: "/"
      route:
        cluster: api-stable
        weighted_clusters:
          clusters:
          - name: api-stable
            weight: 90
          - name: api-canary
            weight: 10
```

**關鍵路由功能**:
- Host / Path / Header / Query Parameter 匹配
- 流量權重分配（Canary 部署）
- 重試策略（retry_on, num_retries）
- 逾時設定（timeout, idle_timeout）
- 路由層級的 Header 修改

### 4. Clusters（集群）

Cluster 代表一組邏輯相同的上游主機。

```yaml
clusters:
- name: api-service
  type: STRICT_DNS           # 服務發現類型
  connect_timeout: 5s
  lb_policy: LEAST_REQUEST   # 負載均衡策略
  health_checks:
  - timeout: 1s
    interval: 10s
    unhealthy_threshold: 3
    healthy_threshold: 1
    http_health_check:
      path: /health
  circuit_breakers:
    thresholds:
    - max_connections: 1000
      max_pending_requests: 100
      max_requests: 500
  outlier_detection:
    consecutive_5xx: 5
    interval: 30s
    base_ejection_time: 30s
```

**服務發現類型**:
| 類型 | 說明 |
|------|------|
| `STATIC` | 靜態 IP 列表 |
| `STRICT_DNS` | 同步 DNS 解析 |
| `LOGICAL_DNS` | 非同步 DNS 解析 |
| `EDS` | 動態端點發現 (xDS) |
| `ORIGINAL_DST` | 保留原始目標位址 |

**負載均衡策略**:
| 策略 | 說明 |
|------|------|
| `ROUND_ROBIN` | 輪詢 |
| `LEAST_REQUEST` | 最少請求數 |
| `RING_HASH` | 一致性雜湊 |
| `RANDOM` | 隨機 |
| `MAGLEV` | Maglev 一致性雜湊 |
| `CLUSTER_PROVIDED` | 集群提供 |

**熔斷器 (Circuit Breaker)**:
- `max_connections`: 最大連線數
- `max_pending_requests`: 最大等待請求數
- `max_requests`: 最大活躍請求數
- `max_retries`: 最大重試數

**異常檢測 (Outlier Detection)**:
- 連續 5xx 錯誤 → 逐出主機
- 可設定逐出時間和恢復條件

## HTTP Connection Manager

這是 Envoy 中最重要的 Network Filter，負責將原始 TCP 位元組轉換為 HTTP 層訊息：

```
TCP Bytes → HTTP Connection Manager
              ├── Protocol Decoding (HTTP/1.1, HTTP/2, HTTP/3)
              ├── Route Resolution
              ├── Header Manipulation
              ├── Retry / Timeout
              ├── Access Logging
              └── Tracing (生成 x-request-id)
```

**支援協議**:
- HTTP/1.1 (包含 WebSocket upgrade)
- HTTP/2 (原生)
- HTTP/3 (QUIC)
- gRPC (基於 HTTP/2)

內部使用**統一的 codec API**，上層程式碼不感知底層協議差異。

## xDS 動態配置協議

xDS (x Discovery Service) 是 Envoy 的動態配置 API，通過 gRPC 雙向串流實現。

### 五大核心服務 (依賴鏈)

```
LDS ──引用──▶ RDS ──引用──▶ CDS ──委派──▶ EDS
 │               │               │
 └───────────────┴───────────────┘
                 │
              SDS (憑證)
```

| 服務 | 全名 | 配置內容 | 說明 |
|------|------|---------|------|
| **LDS** | Listener Discovery | 監聽埠、過濾器鏈 | "聽哪個埠？" |
| **RDS** | Route Discovery | 路由表、虛擬主機 | "路徑對應到哪？" |
| **CDS** | Cluster Discovery | 集群配置、LB 策略 | "後端集群怎麼連？" |
| **EDS** | Endpoint Discovery | 實際 IP 位址 | "後端實例在哪？" |
| **SDS** | Secret Discovery | TLS 憑證、私鑰 | "用什麼憑證？" |

### 嚴格依賴順序

xDS 更新必須遵循依賴順序：
1. CDS/EDS 必須先於 LDS 到達（LDS 引用的 Cluster 必須已存在）
2. RDS 必須在對應的 LDS 之後（RDS 被 LDS 引用）
3. EDS 必須在對應的 CDS 之後（EDS 由 CDS 委派）

### ADS (Aggregated Discovery Service)

為解決依賴順序問題，ADS 將所有 xDS 資源合併到**單一 gRPC 雙向串流**中：

```
Control Plane (e.g., istiod, Envoy Gateway Controller)
    │
    │  Single gRPC Bidirectional Stream (ADS)
    │  ├── LDS Response
    │  ├── CDS Response
    │  ├── EDS Response
    │  ├── RDS Response
    │  └── SDS Response
    │
    ▼
Envoy Proxy (Data Plane)
    │
    │  ACK / NACK (per resource)
    │
    ▲
```

### ACK/NACK 機制 — 自動回滾

xDS 的核心安全機制：

1. Control Plane 發送 `DiscoveryResponse`（包含 `nonce` 和 `version_info`）
2. Envoy 驗證配置有效性
3. Envoy 回傳 `DiscoveryRequest`:
   - ✅ **ACK**: 回傳最新 `version_info` → 配置生效
   - ❌ **NACK**: 回傳舊 `version_info` → **自動回滾到最後已知有效配置**

> "即使配置出錯，Envoy 也不會中斷現有連線，而是繼續使用上一個有效配置。"

### Delta xDS (增量更新)

替代 SotW (State of the World)，只發送變更的部分：

```
SotW: "這是完整的集群列表 [A, B, C, D]"  (每次全量)
Delta: "新增 C, 移除 A"                 (只發差異)
```

適合大規模部署（數千個集群/端點）。

## Envoy Admin 介面

```yaml
admin:
  address:
    socket_address:
      address: 0.0.0.0
      port_value: 9901
```

功能：
- `GET /server_info` — 伺服器資訊
- `GET /config_dump` — 完整配置匯出
- `GET /stats` — Prometheus 格式指標
- `GET /clusters` — 集群狀態
- `POST /logging` — 動態調整日誌等級
- `POST /healthcheck/fail` — 標記健康檢查失敗
- `POST /drain_listeners` — 優雅關閉監聽器

## 安全模型

| 功能 | 說明 |
|------|------|
| **TLS/mTLS** | Transport Socket 支援雙向 TLS |
| **JWT Authentication** | 內建 JWT 驗證過濾器，支援遠端 JWKS |
| **External Authorization** | 委派授權決策給外部 gRPC/HTTP 服務 |
| **RBAC** | 基於角色的存取控制（L4 和 L7 層級） |
| **Wasm** | WebAssembly 沙箱擴展安全機制 |

## 總結

Envoy Proxy 的架構展現了極高的工程品質：
- **Filter Chain** 設計讓功能組合極度靈活
- **xDS 協議** 實現真正的零停機動態配置
- **ACK/NACK 機制** 提供內建的自動回滾保護
- **多協議支援** 讓它成為統一的代理層

這些特性使 Envoy 成為 Istio、Envoy Gateway、Contour 等專案的理想資料平面（data plane）。
