---
title: "K8s Gateway API 規範深入解析"
date: 2026-05-15
tags: [kubernetes, gateway-api, spec, crd, networking]
draft: false
---

## 設計理念與角色模型

Gateway API 的核心創新在於**角色導向（Role-Oriented）**設計，將網路配置的責任分離給三個角色：

### 三個 Persona

| 角色 | 身分 | 職責 |
|------|------|------|
| **Ian** (Infrastructure Provider) | 基礎設施供應商 | 管理底層基礎設施（cloud load balancer, 硬體設備） |
| **Chihiro** (Cluster Operator) | 叢集管理員 | 管理 K8s 叢集、設定網路政策、Gateway 實例 |
| **Ana** (Application Developer) | 應用開發者 | 定義應用的路由規則，不關心基礎設施細節 |

這種設計讓：
- 平台團隊可以管理共享的 Gateway 基礎設施
- 開發團隊可以獨立定義自己的路由規則
- 雙方不需要協調即可安全地共用同一個 Gateway

## 核心資源詳細規範

### 1. GatewayClass — 集群級資源

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: GatewayClass
metadata:
  name: envoy-gateway-class
spec:
  controllerName: gateway.envoyproxy.io/gatewayclass-controller
  parametersRef:
    group: gateway.envoyproxy.io
    kind: EnvoyProxy
    name: envoy-proxy-config
```

- `controllerName`: 指定哪個控制器管理此類 Gateway
- `parametersRef`: 引用控制器特定的配置參數
- 一個叢集中可以有多個 GatewayClass（如：內網/外網分離）

### 2. Gateway — 命名空間級資源

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: production-gateway
  namespace: infra
spec:
  gatewayClassName: envoy-gateway-class
  listeners:
  - name: http
    protocol: HTTP
    port: 80
    hostname: "*.example.com"
    allowedRoutes:
      namespaces:
        from: Selector
        selector:
          matchLabels:
            shared-gateway: "true"
  - name: https
    protocol: HTTPS
    port: 443
    hostname: "*.example.com"
    tls:
      mode: Terminate
      certificateRefs:
      - name: example-tls-secret
    allowedRoutes:
      namespaces:
        from: Selector
        selector:
          matchLabels:
            shared-gateway: "true"
```

**關鍵欄位**:
- `listeners`: 定義監聽的埠、協議、主機名
- `allowedRoutes`: 控制哪些 Route 可以附加到此 Gateway（雙向信任模型）
  - `from: Same` — 只有同命名空間的 Route
  - `from: All` — 所有命名空間的 Route
  - `from: Selector` — 特定標籤的命名空間
- `tls.mode`: `Terminate`（解密）、`Passthrough`（透傳）

### 3. HTTPRoute — 路由規則

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: api-routes
spec:
  parentRefs:
  - name: production-gateway
    namespace: infra
  hostnames:
  - "api.example.com"
  rules:
  # 精確路徑匹配
  - matches:
    - path:
        type: Exact
        value: /health
    backendRefs:
    - name: health-check-service
      port: 8080

  # 前綴匹配 + 標頭匹配
  - matches:
    - path:
        type: PathPrefix
        value: /v1
      headers:
      - name: version
        value: v2
    backendRefs:
    - name: api-v2-service
      port: 8080

  # 流量權重分配（金絲雀部署）
  - matches:
    - path:
        type: PathPrefix
        value: /api
    backendRefs:
    - name: api-stable
      port: 8080
      weight: 90
    - name: api-canary
      port: 8080
      weight: 10

  # 請求/回應標頭修改
  - matches:
    - path:
        type: PathPrefix
        value: /legacy
    filters:
    - type: RequestHeaderModifier
      requestHeaderModifier:
        add:
        - name: X-Forwarded-From
          value: gateway
        remove:
        - X-Internal-ID
    - type: ResponseHeaderModifier
      responseHeaderModifier:
        set:
        - name: X-Served-By
          value: envoy-gateway
    backendRefs:
    - name: legacy-service
      port: 8080

  # URL 重寫
  - matches:
    - path:
        type: PathPrefix
        value: /old-path
    filters:
    - type: URLRewrite
      urlRewrite:
        path:
          type: ReplacePrefixMatch
          replacePrefixMatch: /new-path
    backendRefs:
    - name: service
      port: 8080

  # 直接回應 (Direct Response)
  - matches:
    - path:
        type: PathPrefix
        value: /blocked
    filters:
    - type: RequestRedirect
      requestRedirect:
        hostname: blocked.example.com
        statusCode: 301
```

**路徑匹配類型**:
| 類型 | 說明 | 範例 |
|------|------|------|
| `Exact` | 精確匹配 | `/health` 只匹配 `/health` |
| `PathPrefix` | 前綴匹配 | `/api` 匹配 `/api/v1`, `/api/users` |
| `RegularExpression` | 正則表達式 | `^/user/[0-9]+$` |

**Filter 類型**:
- `RequestHeaderModifier` — 請求標頭修改
- `ResponseHeaderModifier` — 回應標頭修改
- `RequestRedirect` — 重定向
- `URLRewrite` — URL 重寫
- `RequestMirror` — 請求鏡像（流量複製）
- `ExtensionRef` — 自定義擴展

### 4. GRPCRoute — gRPC 路由

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: GRPCRoute
metadata:
  name: grpc-routes
spec:
  parentRefs:
  - name: production-gateway
    namespace: infra
  hostnames:
  - "grpc.example.com"
  rules:
  # 精確 gRPC 方法匹配
  - matches:
    - method:
        service: com.example.UserService
        method: GetUser
    backendRefs:
    - name: user-service
      port: 50051
  # 服務層級匹配
  - matches:
    - method:
        service: com.example.OrderService
    backendRefs:
    - name: order-service
      port: 50051
  # 全部 gRPC 流量
  - backendRefs:
    - name: grpc-default-service
      port: 50051
```

**注意**: 
- 需要 HTTP/2 支援（不支援從 HTTP/1 升級）
- 支援 method-level 的服務和方法匹配
- 未匹配的 RPC 不會被轉發

### 5. ReferenceGrant — 跨命名空間授權

```yaml
apiVersion: gateway.networking.k8s.io/v1beta1
kind: ReferenceGrant
metadata:
  name: allow-team-a-gateway
  namespace: team-a
spec:
  from:
  - group: gateway.networking.k8s.io
    kind: HTTPRoute
    namespace: team-b
  to:
  - group: ""
    kind: Service
    name: backend-service
```

這允許 `team-b` 命名空間的 HTTPRoute 引用 `team-a` 命名空間的 Service。

## 路由附加模型

```
Gateway (listeners.allowedRoutes) ──篩選──▶ Route (parentRefs)
    ◀──驗證── 雙向信任 ──確認──▶
```

1. Route 通過 `parentRefs` 宣告想附加到哪個 Gateway
2. Gateway 通過 `allowedRoutes` 決定接受哪些 Route
3. 兩者必須互相匹配，Route 才會生效

## 一致性 (Conformance) 等級

Gateway API 定義了分級的一致性測試：

| 等級 | 說明 |
|------|------|
| **Core** | 所有實作必須支援的基本功能 |
| **Extended** | 進階功能（可選，但若實作則必須通過測試） |
| **Implementation-specific** | 實作特有功能 |

發布頻道 (Release Channels):
- **Standard** — GA 穩定功能
- **Experimental** — 實驗性功能，API 可能變更

## Ingress → Gateway API 遷移

官方提供遷移工具 `ingress2gateway`:

```bash
# 安裝
go install github.com/kubernetes-sigs/ingress2gateway/cmd/ingress2gateway@latest

# 轉換
ingress2gateway print --input-file ingress.yaml \
  --output-file gateway-resources.yaml
```

## 總結

Gateway API 規範通過清晰的角色分離、豐富的表達能力、和跨命名空間的雙向信任模型，解決了 Ingress API 的結構性問題。它不僅是 Ingress 的替代品，更是一套統一的 L4/L7 網路管理標準。
