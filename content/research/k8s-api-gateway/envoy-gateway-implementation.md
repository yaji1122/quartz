---
title: "Envoy Gateway 實作解析"
date: 2026-05-15
tags: [envoy-gateway, kubernetes, implementation, controller, gateway-api]
draft: false
---

## Envoy Gateway 是什麼？

**Envoy Gateway** 是一個 CNCF 託管的開源專案，作為 Envoy Proxy 的 **Kubernetes-native 控制平面 (Control Plane)**。它將 Kubernetes Gateway API 資源自動轉換為 Envoy Proxy 的配置，並管理 Envoy 實例的生命週期。

> "Envoy Gateway 結合了 Envoy 的效能與 Kubernetes 原生的配置方式，讓平台團隊以最小的維運成本暴露和管理安全、可觀測、可擴展的 API。"

**GA 時間**: 2024 年 3 月（v1.0）
**最新版本**: v1.4 (截至 2026 年 5 月)
**貢獻者**: 211 人來自 54 家公司

## 架構分層

```
┌─────────────────────────────────────────────────┐
│           用戶配置層 (User Configuration)          │
│  GatewayClass, Gateway, HTTPRoute, Policy CRDs   │
└────────────────────┬────────────────────────────┘
                     │ Watch
┌────────────────────▼────────────────────────────┐
│      Envoy Gateway Controller (Control Plane)    │
│  ┌──────────┐  ┌───────────┐  ┌──────────────┐  │
│  │ Provider │  │  Gateway  │  │ Infra Manager│  │
│  │ (K8s API)│──│  API      │──│ (Deploy Envoy)│  │
│  │          │  │ Translator│  │              │  │
│  └──────────┘  └───────────┘  └──────────────┘  │
│                       │                          │
│               xDS (ADS) gRPC Stream              │
└───────────────────────┬──────────────────────────┘
                        │
┌───────────────────────▼──────────────────────────┐
│         Envoy Proxy (Data Plane)                 │
│  ┌─────────┐  ┌─────────┐  ┌─────────┐          │
│  │ Envoy 1 │  │ Envoy 2 │  │ Envoy N │          │
│  │  Pod    │  │  Pod    │  │  Pod    │  ...      │
│  └─────────┘  └─────────┘  └─────────┘          │
└─────────────────────────────────────────────────┘
```

### 三層架構說明

| 層級 | 組件 | 職責 |
|------|------|------|
| **配置層** | Kubernetes CRDs | 用戶通過 Gateway API + EG CRDs 宣告期望狀態 |
| **控制平面** | Envoy Gateway Controller | 監聽 K8s 資源變化、翻譯為 Envoy 配置、管理 Envoy 生命週期 |
| **資料平面** | Envoy Proxy Pods | 處理實際流量，根據 xDS 配置進行路由、過濾、負載均衡 |

## Envoy Gateway Controller 內部架構

Controller 是 Envoy Gateway 的大腦，包含以下核心組件：

### 1. Provider（提供者）

監聽 Kubernetes API Server 的資源變化：

```
Kubernetes API Server
    │
    │ Informer / Watcher
    ▼
Provider (Kubernetes Provider)
    │
    │ 監聽資源類型:
    ├── GatewayClass, Gateway
    ├── HTTPRoute, GRPCRoute, TCPRoute, UDPRoute, TLSRoute
    ├── Service, Endpoints, Secrets
    ├── ReferenceGrant
    └── EG CRDs (EnvoyProxy, SecurityPolicy, etc.)
```

### 2. Gateway API Translator（翻譯器）

將 Kubernetes 資源轉換為 Envoy 的 xDS 配置（IR - Intermediate Representation 模式）：

```
K8s Resources (GatewayAPI + EG CRDs)
    │
    ▼
Infra IR (基礎設施中間表示)
    ├── 需要幾個 Envoy Deployment?
    ├── 用什麼 Service type (LoadBalancer/ClusterIP)?
    ├── HPA 配置?
    └── Envoy Proxy 的 bootstrap 配置
    │
    ▼
xDS IR (xDS 中間表示)
    ├── Listeners → LDS
    ├── HTTP Routes → RDS
    ├── Clusters → CDS
    ├── Endpoints → EDS
    └── Secrets → SDS
    │
    ▼
xDS Translation
    └── 生成 Protocol Buffer 格式的 Envoy 配置
    │
    ▼
gRPC Stream (ADS) → Envoy Proxy
```

### 3. Infra Manager（基礎設施管理器）

管理 Envoy Proxy 實例的生命週期：

- 為每個 Gateway 資源建立一個 Envoy Deployment
- 自動建立對應的 Service (LoadBalancer 或 ClusterIP)
- 支援 HPA (Horizontal Pod Autoscaler) 自動擴縮
- 管理 Envoy Proxy 的 Pod 配置 (資源限制、親和性等)

## 自定義 CRD (Extension Types)

Envoy Gateway 除了標準 Gateway API 外，還提供豐富的擴展 CRD：

### EnvoyProxy — 控制 Envoy 部署

```yaml
apiVersion: gateway.envoyproxy.io/v1alpha1
kind: EnvoyProxy
metadata:
  name: custom-envoy-config
spec:
  provider:
    kubernetes:
      envoyDeployment:
        replicas: 3
        pod:
          affinity:
            nodeAffinity:
              requiredDuringSchedulingIgnoredDuringExecution:
                nodeSelectorTerms:
                - matchExpressions:
                  - key: node-type
                    operator: In
                    values: ["gateway"]
        container:
          resources:
            requests:
              cpu: 500m
              memory: 512Mi
            limits:
              cpu: 2000m
              memory: 2Gi
      envoyService:
        type: LoadBalancer
        loadBalancerIP: 203.0.113.10
        annotations:
          service.beta.kubernetes.io/aws-load-balancer-type: nlb
```

### BackendTrafficPolicy — 後端流量策略

```yaml
apiVersion: gateway.envoyproxy.io/v1alpha1
kind: BackendTrafficPolicy
metadata:
  name: api-resilience-policy
spec:
  targetRef:
    group: gateway.networking.k8s.io
    kind: HTTPRoute
    name: api-routes
  rateLimit:
    global:
      rules:
      - clientSelectors:
        - headers:
          - name: x-api-key
        limit:
          requests: 1000
          unit: Hour
  loadBalancer:
    type: LeastRequest
  healthCheck:
    active:
      type: HTTP
      http:
        path: /health
        expectedStatuses: [200]
      timeout: 3s
      interval: 10s
      unhealthyThreshold: 3
  circuitBreaker:
    maxConnections: 1000
    maxPendingRequests: 200
    maxRequests: 500
    maxRetries: 3
  retry:
    retryOn:
      httpStatuses: [502, 503, 504]
      triggers: ["connect-failure", "refused-stream"]
    numRetries: 3
    perRetryTimeout: 15s
  timeout:
    http:
      requestReceivedTimeout: 60s
```

**targetRef 層級**: 可以附加到 Gateway (全域) 或 Route (特定路由)

### SecurityPolicy — 安全策略

```yaml
apiVersion: gateway.envoyproxy.io/v1alpha1
kind: SecurityPolicy
metadata:
  name: jwt-auth-policy
spec:
  targetRef:
    group: gateway.networking.k8s.io
    kind: Gateway
    name: production-gateway
  jwt:
    providers:
    - name: google-oauth
      issuer: "https://accounts.google.com"
      audiences: ["my-app-client-id"]
      remoteJWKS:
        uri: "https://www.googleapis.com/oauth2/v3/certs"
      claimToHeaders:
      - header: x-user-id
        claim: sub
      - header: x-user-email
        claim: email
  cors:
    allowOrigins:
    - "https://app.example.com"
    - "https://admin.example.com"
    allowMethods: ["GET", "POST", "PUT", "DELETE"]
    allowHeaders: ["Authorization", "Content-Type"]
    exposeHeaders: ["X-Request-Id"]
    maxAge: 86400s
  oidc:
    provider:
      issuer: "https://accounts.google.com"
    clientID: "my-app-client-id"
    clientSecretRef:
      name: oidc-secret
    redirectURL: "https://api.example.com/oauth2/callback"
  extAuth:
    grpc:
      backendRef:
        name: ext-auth-service
        port: 9001
```

### ClientTrafficPolicy — 客戶端流量策略

```yaml
apiVersion: gateway.envoyproxy.io/v1alpha1
kind: ClientTrafficPolicy
metadata:
  name: client-settings
spec:
  targetRef:
    group: gateway.networking.k8s.io
    kind: Gateway
    name: production-gateway
  tcpKeepalive:
    idleTime: 7200s
    interval: 75s
    probes: 9
  enableProxyProtocol: true
  clientIPDetection:
    xForwardedFor:
      numTrustedHops: 2
  http3: {}  # 啟用 HTTP/3
```

### EnvoyPatchPolicy — 底層 Envoy 配置補丁

當 EG CRD 不支援某個 Envoy 功能時，可以通過 `EnvoyPatchPolicy` 直接注入 Envoy JSON 配置：

```yaml
apiVersion: gateway.envoyproxy.io/v1alpha1
kind: EnvoyPatchPolicy
metadata:
  name: custom-filter-patch
spec:
  targetRef:
    group: gateway.networking.k8s.io
    kind: Gateway
    name: production-gateway
  type: JSONPatch
  jsonPatches:
  - type: "type.googleapis.com/envoy.config.listener.v3.Listener"
    name: "production-gateway-http"
    operation:
      op: add
      path: "/filter_chains/0/filters/0/typed_config/http_filters/-"
      value:
        name: "envoy.filters.http.lua"
        typed_config:
          "@type": "type.googleapis.com/envoy.extensions.filters.http.lua.v3.Lua"
          inline_code: |
            function envoy_on_request(request_handle)
              request_handle:headers():add("x-custom", "hello")
            end
```

## Envoy AI Gateway — 最新演進

**Envoy AI Gateway** 是 Envoy 生態系統中專門為 AI 流量設計的擴展：

- **MCP (Model Context Protocol) 支援**: 為 AI Agent 工具呼叫提供企業級安全、路由和可觀察性
- **AI Provider 路由**: 統一管理多個 LLM Provider（OpenAI, Anthropic, etc.）
- **Token 速率限制**: 基於 token 使用量的速率控制
- **Prompt 安全過濾**: 在 Gateway 層級進行內容過濾

## 發版歷程與關鍵功能

| 版本 | 重點功能 |
|------|---------|
| **v1.0** (2024/03) | GA 發布，完整的 Gateway API 支援 |
| **v1.1** | Backend 資源（路由到 K8s 外部）、mTLS、Envoy HTTP Filter 排序 |
| **v1.2** | JWT Claims 授權、IPv4/IPv6 雙棧、直接回應、Header 改寫、Session Persistence |
| **v1.3** | TCPRoute/UDPRoute、gRPC 健康檢查、Wasm 擴展 |
| **v1.4** (最新) | 更豐富的 Policy 支援、效能優化、更多一致性測試通過 |

## 總結

Envoy Gateway 的設計體現了現代 K8s Controller 的最佳實踐：
- **IR 模式**: 通過中間表示分離關注點，提高可測試性
- **宣告式 API**: 用戶只需描述期望狀態，Controller 負責實現
- **xDS 原生**: 直接通過 ADS 推送配置到 Envoy，無需重啟
- **Policy CRDs**: 將常見需求（限流、熔斷、JWT）封裝為簡單的 CRD
- **Patches 作為 escape hatch**: 保留底層 Envoy 的全部靈活性
