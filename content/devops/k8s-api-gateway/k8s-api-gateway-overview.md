---
title: "K8s API Gateway 概述"
date: 2026-05-15
tags: [kubernetes, api-gateway, envoy, gateway-api, networking]
draft: false
---

## 什麼是 API Gateway？

在微服務架構中，**API Gateway** 是一個反向代理（reverse proxy），作為所有客戶端請求的**單一入口點（single entry point）**。它負責：

- **請求路由（Routing）**: 根據路徑、標頭等將請求轉發到對應的後端服務
- **負載均衡（Load Balancing）**: 將流量分配到多個後端實例
- **認證與授權（AuthN/AuthZ）**: JWT 驗證、OAuth、mTLS 等
- **速率限制（Rate Limiting）**: 防止 DDoS 和資源濫用
- **TLS 終止（TLS Termination）**: 在邊緣解密 HTTPS 流量
- **可觀察性（Observability）**: 日誌、指標、分散式追蹤
- **協議轉換（Protocol Translation）**: HTTP ↔ gRPC、REST ↔ GraphQL

## Kubernetes 中的 API Gateway 演進

### 第一代：Ingress API (2015)

```yaml
# 傳統 Ingress 範例 — 功能有限，依賴 annotations
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: my-ingress
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    # ... annotation sprawl!
spec:
  rules:
  - host: example.com
    http:
      paths:
      - path: /api
        backend:
          service:
            name: api-service
            port: { number: 80 }
```

**Ingress 的侷限性**:
- ❌ 只支援 HTTP/HTTPS (Layer 7)，不支援 TCP/UDP (Layer 4)
- ❌ 進階功能依賴**供應商特有 annotations**，缺乏可移植性
- ❌ 沒有標準化的流量權重、標頭匹配、跨命名空間路由
- ❌ 角色模糊：基礎設施團隊和開發者共用同一份設定

### 第二代：Gateway API (2023 → GA)

**Kubernetes Gateway API** 是 SIG-NETWORK 制定的新一代標準，旨在取代 Ingress API。

**核心設計原則**:
| 原則 | 說明 |
|------|------|
| **角色導向 (Role-Oriented)** | 分離基礎設施供應商、叢集管理員、應用開發者的職責 |
| **可移植 (Portable)** | 標準化 CRD，不被單一供應商鎖定 |
| **表達力強 (Expressive)** | 原生支援標頭匹配、流量權重、查詢參數路由等 |
| **可擴展 (Extensible)** | 允許在各層級連結自定義資源 |

**核心資源模型**:

```
GatewayClass (基礎設施供應商定義)
    └── Gateway (叢集管理員建立)
           └── HTTPRoute / GRPCRoute / TCPRoute (開發者定義路由規則)
                  └── Service (後端服務)
```

| 資源 | 由誰管理 | 用途 |
|------|---------|------|
| **GatewayClass** | 平台供應商 | 定義 Gateway 控制器的類型（cloud LB, Envoy, etc.） |
| **Gateway** | 叢集管理員 | 宣告流量入口的監聽埠、協議、TLS 設定 |
| **HTTPRoute** | 應用開發者 | 定義 HTTP 路由規則（路徑、標頭、流量權重） |
| **GRPCRoute** | 應用開發者 | 定義 gRPC 路由規則（服務/方法匹配） |

## Gateway API vs API Gateway

> ⚠️ **重要區分**: "Gateway API" 是 Kubernetes 的**標準規範**（一組 CRD），而 "API Gateway" 是**實作該規範的具體軟體**（如 Envoy Gateway、Kong、Istio）。

## Gateway API 在 Ingress (南北向) 中的應用

```
客戶端 → Gateway (監聽 80/443) → HTTPRoute (路由規則) → Service → Pod
```

- 一個 Gateway 可以被**多個 HTTPRoute**（甚至跨命名空間）共用
- `ReferenceGrant` 提供明確的跨命名空間授權
- 支援 shared Gateway 架構：多個團隊共用同一個基礎設施入口

## Gateway API 在 Service Mesh (東西向) 中的應用

GAMMA (Gateway API for Mesh Management and Administration) 倡議將 Gateway API 延伸到 mesh 場景：

- 在 mesh 中不建立 Gateway/GatewayClass
- HTTPRoute 直接附加到 Service 資源
- 統一的 API 管理南北向（邊緣）和東西向（網格內）流量

## 主流實作方案一覽

| 實作 | 類型 | 特點 |
|------|------|------|
| **Envoy Gateway** | Envoy-Native | CNCF 專案，最忠實的 Gateway API 實作 |
| **Istio** | Envoy-Native | 完整的 Service Mesh + Gateway API 支援 |
| **Contour** | Envoy-Native | CNCF 畢業專案，Enovy 前輩 |
| **Cilium** | eBPF + Envoy | L4 用 eBPF kernel 層處理，效能極佳 |
| **Kong** | NGINX + Lua | API 管理平台，豐富的 plugin 生態 |
| **NGINX Gateway Fabric** | NGINX | NGINX 官方 Gateway API 實作 |

## 總結

Kubernetes Gateway API 代表了 K8s 網路管理的未來方向。它解決了 Ingress 的標準化問題，通過角色導向設計讓團隊協作更順暢。在眾多實作中，**Envoy Gateway** 憑藉其 CNCF 中立性、對 Gateway API 的完整支援，以及 Envoy Proxy 的強大性能，成為最值得深入研究的標的。
