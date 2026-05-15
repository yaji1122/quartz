---
title: "K8s Gateway API 實作對比與選型指南"
date: 2026-05-15
tags: [kubernetes, gateway-api, envoy-gateway, istio, cilium, kong, nginx]
draft: false
---

## 三種實作架構模型

Gateway API 的實作可以分為三種架構模式：

### 1. Envoy-Native 模型

**代表**: Envoy Gateway, Istio, Contour

```
K8s Resources ──▶ Controller ──▶ xDS (gRPC) ──▶ Envoy Proxy
```

- Kubernetes 資源直接翻譯為 Envoy xDS 配置
- **高保真度** — API 欄位與 Envoy 配置近乎 1:1 對應
- **代價**: 大規模時的 reconcile loop 成本較高

### 2. NGINX-Adapter 模型

**代表**: NGINX Gateway Fabric, Kong

```
K8s Resources ──▶ Controller ──▶ NGINX Config Reload
```

- 依賴 NGINX 配置重載（或 Kong 的動態 Lua 補丁）
- **阻抗不匹配** — Gateway API 的動態性與 NGINX 的靜態配置模型衝突
- 不相關的叢集事件可能觸發不必要的配置重載（CPU 峰值）

### 3. Kernel-Native (eBPF) 模型

**代表**: Cilium

```
K8s Resources ──▶ Cilium Operator ──▶ eBPF Programs (L4)
                                    ──▶ Shared Envoy (L7)
```

- **L4 流量**: 在 kernel 層通過 eBPF 處理（無需 userspace proxy）
- **L7 流量**: 交給每個節點的共享 Envoy 實例
- **混合複雜度**: Gateway 邏輯分散，CNI 升級影響大

## 詳細對比

### Envoy Gateway — 標準旗手

| 維度 | 評價 |
|------|------|
| **CNCF 狀態** | CNCF 託管專案 |
| **Gateway API 覆蓋** | ⭐⭐⭐⭐⭐ 最完整的標準實作 |
| **協議支援** | HTTP, gRPC, TLS, TCP, UDP |
| **控制平面語言** | Go |
| **部署模型** | 每個 Gateway 自動建立獨立 Envoy Deployment |
| **Policy 擴展** | BackendTrafficPolicy, SecurityPolicy, ClientTrafficPolicy |
| **Escape Hatch** | EnvoyPatchPolicy (直接注入 Envoy JSON) |
| **供應商中立** | ⭐⭐⭐⭐⭐ 完全中立 |
| **社群活躍度** | 211 貢獻者，54 家公司 |

**適用場景**:
- 需要完整的 Gateway API 標準覆蓋
- 重視供應商中立性
- 需要 Envoy 的全部原生能力
- 願意接受較新的專案（2024 GA）

### Istio (Ambient Mode) — 網格王者

| 維度 | 評價 |
|------|------|
| **CNCF 狀態** | CNCF 畢業專案 |
| **Gateway API 覆蓋** | ⭐⭐⭐⭐⭐ 支援 Gateway API + Mesh |
| **協議支援** | HTTP, gRPC, TLS, TCP |
| **控制平面語言** | Go |
| **部署模型** | Ambient: ztunnel (L4) + Waypoint (L7, 每個 ServiceAccount) |
| **獨特優勢** | 統一的 Gateway + Service Mesh、FIPS 加密 |
| **控制平面效能** | ⭐⭐⭐⭐⭐ 最快的配置更新延遲 |

**適用場景**:
- 已經或計劃使用 Service Mesh
- 需要統一的南北向 + 東西向流量管理
- 對安全有高要求（mTLS、FIPS）
- 對效能和延遲敏感

### Cilium — Kernel 挑戰者

| 維度 | 評價 |
|------|------|
| **CNCF 狀態** | CNCF 畢業專案 |
| **Gateway API 覆蓋** | ⭐⭐⭐⭐ 支援主要功能 |
| **協議支援** | HTTP, gRPC, TLS, TCP |
| **獨特優勢** | eBPF 帶來的 L4 性能無與倫比 |
| **L7 限制** | 每個節點共享 Envoy → 租戶資源競爭風險 |
| **控制平面成本** | 壓力測試中 CPU 比 Istio 高 **15 倍** |

**適用場景**:
- 已經使用 Cilium 作為 CNI
- L4 效能是首要考量
- 對 L7 處理需求較簡單
- 可以接受 CNI 與 Gateway 耦合

### Kong — API 管理平台

| 維度 | 評價 |
|------|------|
| **Gateway API 覆蓋** | ⭐⭐⭐ 基本覆蓋 |
| **獨特優勢** | 豐富的 Plugin 生態（Lua, Go） |
| **Plugin 類型** | 速率限制、轉換、日誌、認證等 |
| **開源 vs 企業** | ⚠️ GUI, OIDC, Analytics 僅在企業版 |
| **適用場景** | 需要完整的 API 管理平台 |

### NGINX Gateway Fabric — NGINX 官方

| 維度 | 評價 |
|------|------|
| **Gateway API 覆蓋** | ⭐⭐⭐ 基本覆蓋 |
| **記憶體使用** | ⭐⭐⭐⭐⭐ 極低 |
| **靜態檔案吞吐** | ⭐⭐⭐⭐⭐ 極高 |
| **弱點** | 不相關叢集事件的 CPU 峰值、配置更新較慢 |
| **開源 vs 商業** | ⚠️ OSS vs NGINX Plus 功能割裂 |

## 功能對比矩陣

| 功能 | Envoy Gateway | Istio (Ambient) | Cilium | Kong | NGINX GF |
|------|:---:|:---:|:---:|:---:|:---:|
| HTTPRoute | ✅ | ✅ | ✅ | ✅ | ✅ |
| GRPCRoute | ✅ | ✅ | ✅ | ✅ | ✅ |
| TLSRoute | ✅ | ✅ | ✅ | ✅ | ⚠️ |
| TCPRoute | ✅ | ✅ | ✅ | ✅ | ⚠️ |
| UDPRoute | ✅ | ✅ | ⚠️ | ❌ | ❌ |
| Traffic Splitting | ✅ | ✅ | ✅ | ✅ | ✅ |
| Header Matching | ✅ | ✅ | ✅ | ✅ | ✅ |
| URL Rewrite | ✅ | ✅ | ✅ | ✅ | ✅ |
| JWT Auth | ✅ | ✅ | ✅ | 🔌 | 🔌+ |
| OIDC | ✅ | ✅ | ❌ | 🔌+ | 🔌+ |
| Rate Limiting | ✅ | ✅ | ❌ | ✅ | 🔌+ |
| Circuit Breaker | ✅ | ✅ | ❌ | ✅ | 🔌+ |
| Health Check | ✅ | ✅ | ❌ | ✅ | 🔌+ |
| mTLS | ✅ | ✅ | ✅ | ✅ | 🔌+ |
| Wasm Extensions | ✅ | ✅ | ❌ | ❌ | ❌ |
| Cross-Namespace | ✅ | ✅ | ✅ | ✅ | ✅ |
| Service Mesh | ❌ | ✅ | ✅ | ❌ | ❌ |
| eBPF L4 | ❌ | ❌ | ✅ | ❌ | ❌ |

> 🔌 = 通過 Plugin 支援 | 🔌+ = 企業版功能 | ⚠️ = 實驗性/部分支援

## 效能對比 (2025 基準測試)

基於社群基準測試的定性比較：

| 指標 | Envoy Gateway | Istio | Cilium | NGINX |
|------|:---:|:---:|:---:|:---:|
| **L4 吞吐量** | ⭐⭐⭐⭐ | ⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐⭐ |
| **L7 延遲** | ⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐⭐⭐ |
| **控制平面更新速度** | ⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐ | ⭐⭐⭐ |
| **記憶體使用** | ⭐⭐⭐ | ⭐⭐ | ⭐⭐⭐ | ⭐⭐⭐⭐⭐ |
| **高變動下的穩定性** | ⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | ⭐⭐ | ⭐⭐⭐ |

## 選型決策樹

```
需要 Service Mesh？
├── 是 → 需要 eBPF 效能？
│        ├── 是 → Cilium
│        └── 否 → Istio (Ambient)
└── 否 → 重視什麼？
         ├── 供應商中立 + 完整 Gateway API → Envoy Gateway
         ├── API 管理平台 (Plugins) → Kong
         ├── NGINX 經驗 + 低記憶體 → NGINX Gateway Fabric
         └── 最簡單的入門體驗 → Envoy Gateway (推薦)
```

## 總結建議

對於**大多數 Kubernetes 使用者**，推薦順序：

1. **Envoy Gateway** — 最佳的 Gateway API 標準實作，供應商中立，功能完整
2. **Istio** — 如果需要 Service Mesh，這是最成熟的選擇
3. **Kong** — 如果已使用 Kong 生態或有 API 管理平台需求
4. **Cilium** — 如果已使用 Cilium CNI 且 L4 效能優先
5. **NGINX Gateway Fabric** — 如果團隊有深厚的 NGINX 經驗

> ⭐ **研究建議**: 以 Envoy Gateway 作為學習和研究的主要標的，因為它最貼近 Gateway API 標準，且沒有 Service Mesh 的額外複雜度。
