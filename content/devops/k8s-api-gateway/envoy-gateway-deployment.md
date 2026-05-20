---
title: "Envoy Gateway 實戰部署指引"
date: 2026-05-15
tags: [envoy-gateway, kubernetes, deployment, helm, quickstart]
draft: false
---

## 環境需求

- Kubernetes 叢集 (v1.26+)
- kubectl 已設定
- Helm 3.0+
- 足夠的叢集資源（至少 2 vCPU, 4GB RAM 可用）

## 快速部署 (5 分鐘上手)

### Step 1: 安裝 Gateway API CRDs

```bash
# 安裝 Standard Channel CRDs
kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/latest/download/standard-install.yaml

# 驗證安裝
kubectl get crds | grep gateway.networking.k8s.io
```

預期輸出：
```
gatewayclasses.gateway.networking.k8s.io
gateways.gateway.networking.k8s.io
httproutes.gateway.networking.k8s.io
grpcroutes.gateway.networking.k8s.io
referencegrants.gateway.networking.k8s.io
...
```

### Step 2: 安裝 Envoy Gateway (Helm)

```bash
# 加入 Helm repo
helm repo add envoyproxy https://helm.envoyproxy.io
helm repo update

# 安裝 Envoy Gateway
helm install envoy-gateway envoyproxy/gateway-helm \
  --namespace envoy-gateway-system \
  --create-namespace \
  --set config.envoyGateway.logging.level.default=info

# 驗證安裝
kubectl get pods -n envoy-gateway-system
```

預期輸出：
```
NAME                                   READY   STATUS    RESTARTS   AGE
envoy-gateway-controller-xxxx-xxxxx    1/1     Running   0          30s
```

### Step 3: 建立 GatewayClass

```yaml
# gatewayclass.yaml
apiVersion: gateway.networking.k8s.io/v1
kind: GatewayClass
metadata:
  name: eg
spec:
  controllerName: gateway.envoyproxy.io/gatewayclass-controller
```

```bash
kubectl apply -f gatewayclass.yaml
kubectl get gatewayclass
```

### Step 4: 建立 Gateway

```yaml
# gateway.yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: my-gateway
  namespace: default
spec:
  gatewayClassName: eg
  listeners:
  - name: http
    protocol: HTTP
    port: 80
```

```bash
kubectl apply -f gateway.yaml

# 檢查 Gateway 狀態和分配的 IP
kubectl get gateway my-gateway
kubectl get svc -l gateway.envoyproxy.io/owning-gateway-name=my-gateway
```

### Step 5: 部署測試應用

```yaml
# echo-server.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: echo-server
spec:
  replicas: 2
  selector:
    matchLabels:
      app: echo-server
  template:
    metadata:
      labels:
        app: echo-server
    spec:
      containers:
      - name: echo
        image: ealen/echo-server:latest
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: echo-server
spec:
  selector:
    app: echo-server
  ports:
  - port: 80
    targetPort: 80
```

```bash
kubectl apply -f echo-server.yaml
kubectl get pods -l app=echo-server
```

### Step 6: 建立 HTTPRoute

```yaml
# httproute.yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: echo-route
spec:
  parentRefs:
  - name: my-gateway
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /echo
    backendRefs:
    - name: echo-server
      port: 80
```

```bash
kubectl apply -f httproute.yaml
kubectl get httproute echo-route
```

### Step 7: 測試

```bash
# 獲取 Gateway IP
export GATEWAY_IP=$(kubectl get gateway my-gateway -o jsonpath='{.status.addresses[0].value}')

# 發送請求
curl -v http://$GATEWAY_IP/echo/test

# 預期回應: 顯示請求的詳細資訊（echo server 回應）
```

## 常見部署模式

### 模式 1: 單一 Gateway，多團隊共用

```
Gateway (infra namespace)
  ├── HTTPRoute (team-a namespace) → team-a 的 services
  ├── HTTPRoute (team-b namespace) → team-b 的 services
  └── HTTPRoute (team-c namespace) → team-c 的 services
```

```yaml
# Gateway 設定允許指定標籤的命名空間
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: shared-gateway
  namespace: infra
spec:
  gatewayClassName: eg
  listeners:
  - name: http
    protocol: HTTP
    port: 80
    allowedRoutes:
      namespaces:
        from: Selector
        selector:
          matchLabels:
            gateway-access: "true"
```

開發團隊只需標記命名空間：
```bash
kubectl label namespace team-a gateway-access=true
kubectl label namespace team-b gateway-access=true
```

### 模式 2: 金絲雀部署 (Canary Deployment)

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: canary-route
spec:
  parentRefs:
  - name: my-gateway
  rules:
  - matches:
    - path:
        type: PathPrefix
        value: /api
    backendRefs:
    - name: api-v1-stable
      port: 8080
      weight: 95
    - name: api-v2-canary
      port: 8080
      weight: 5
```

修改權重即可調整流量：
```bash
# 增加金絲雀流量到 20%
kubectl patch httproute canary-route --type='json' \
  -p='[{"op": "replace", "path": "/spec/rules/0/backendRefs/0/weight", "value": 80},
       {"op": "replace", "path": "/spec/rules/0/backendRefs/1/weight", "value": 20}]'
```

### 模式 3: HTTPS + 自動憑證 (cert-manager)

```yaml
# Gateway with HTTPS
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: tls-gateway
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
spec:
  gatewayClassName: eg
  listeners:
  - name: https
    protocol: HTTPS
    port: 443
    hostname: "api.example.com"
    tls:
      mode: Terminate
      certificateRefs:
      - name: api-tls-cert  # cert-manager 自動管理
```

### 模式 4: JWT 認證 + 速率限制

```yaml
# SecurityPolicy
apiVersion: gateway.envoyproxy.io/v1alpha1
kind: SecurityPolicy
metadata:
  name: api-security
spec:
  targetRef:
    group: gateway.networking.k8s.io
    kind: Gateway
    name: my-gateway
  jwt:
    providers:
    - name: custom-auth
      issuer: "https://auth.example.com"
      audiences: ["api-gateway"]
      remoteJWKS:
        uri: "https://auth.example.com/.well-known/jwks.json"
      claimToHeaders:
      - header: x-user-id
        claim: sub
---
# BackendTrafficPolicy
apiVersion: gateway.envoyproxy.io/v1alpha1
kind: BackendTrafficPolicy
metadata:
  name: api-rate-limit
spec:
  targetRef:
    group: gateway.networking.k8s.io
    kind: HTTPRoute
    name: echo-route
  rateLimit:
    global:
      rules:
      - clientSelectors:
        - headers:
          - name: x-user-id
        limit:
          requests: 100
          unit: Hour
```

## 監控與可觀察性

### Prometheus Metrics

Envoy Gateway 自動暴露 Prometheus 指標：

```bash
# 查看 Envoy Proxy metrics endpoint
kubectl port-forward -n envoy-gateway-system \
  deploy/$(kubectl get deploy -n envoy-gateway-system \
    -l gateway.envoyproxy.io/owning-gateway-name=my-gateway \
    -o jsonpath='{.items[0].metadata.name}') 19001:19001

curl http://localhost:19001/stats/prometheus
```

### Grafana Dashboard

社群提供的 Envoy Gateway Grafana Dashboard 可視化：
- 請求速率 (RPS)
- 錯誤率 (4xx/5xx)
- 延遲 (P50/P95/P99)
- 上游健康狀態
- 連線池使用率

## 疑難排解

### 常見問題

**Gateway 沒有獲得 IP**:
```bash
# 檢查 Gateway 狀態
kubectl describe gateway my-gateway

# 檢查 Envoy Gateway Controller 日誌
kubectl logs -n envoy-gateway-system \
  deploy/envoy-gateway-controller -f
```

**HTTPRoute 未生效**:
```bash
# 確認 Route 被接受
kubectl get httproute echo-route -o yaml | grep -A 10 "status:"
# 尋找: accepted: true

# 檢查 Route 的 parentRef 是否正確
kubectl describe httproute echo-route
```

**Envoy Proxy Pod 崩潰**:
```bash
# 查看 Envoy 日誌
kubectl logs -n envoy-gateway-system \
  -l gateway.envoyproxy.io/owning-gateway-name=my-gateway

# 查看 Envoy 配置
kubectl exec -n envoy-gateway-system \
  deploy/<envoy-deployment> -- curl -s localhost:19001/config_dump
```

### 除錯技巧

```bash
# 啟用 Envoy Gateway 的除錯日誌
helm upgrade envoy-gateway envoyproxy/gateway-helm \
  --namespace envoy-gateway-system \
  --set config.envoyGateway.logging.level.default=debug

# 匯出 Envoy 的完整配置（診斷用）
kubectl exec -n envoy-gateway-system \
  deploy/<envoy-deployment> -- \
  curl -s localhost:19001/config_dump > envoy-config.json
```

## 升級策略

```bash
# 1. 先升級 Gateway API CRDs
kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/latest/download/standard-install.yaml

# 2. 升級 Envoy Gateway Helm chart
helm repo update
helm upgrade envoy-gateway envoyproxy/gateway-helm \
  --namespace envoy-gateway-system \
  --reuse-values

# 3. 驗證
kubectl get pods -n envoy-gateway-system
kubectl get gateway -A
```

## 生產環境檢查清單

- [ ] 設定資源請求和限制 (EnvoyProxy CRD)
- [ ] 設定 HPA (EnvoyProxy CRD: `replicas` 或 `horizontalPodAutoscaler`)
- [ ] 設定 Pod 反親和性 (避免所有 Envoy 實例在同一節點)
- [ ] 啟用 TLS (生產環境必須使用 HTTPS)
- [ ] 設定適當的 RBAC（限制 Gateway/Route 的建立權限）
- [ ] 設定持久化日誌（Filebeat/Fluentd → Elasticsearch）
- [ ] 設定 Prometheus 監控 + Grafana Dashboard
- [ ] 設定告警規則（5xx 錯誤率、延遲升高）
- [ ] 測試故障轉移（主動/被動健康檢查 + Failover）
- [ ] 設定備份 Gateway 配置（GitOps: ArgoCD/Flux）

## 總結

Envoy Gateway 提供了從「5 分鐘快速啟動」到「生產級部署」的完整路徑：

1. **快速啟動**: `kubectl apply` 3-4 個 YAML 即可運行
2. **進階配置**: 通過 Policy CRDs 實現安全、限流、熔斷
3. **生產就緒**: 支援 HPA、TLS、監控、GitOps

對於想要深入學習 K8s API Gateway 的讀者，建議從這個部署指引開始實際動手操作，建立直觀的體驗。
