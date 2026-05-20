---
title: "K8s API Gateway 研究系列 — 以 Envoy Gateway 為標的"
date: 2026-05-15
tags: [kubernetes, api-gateway, envoy, gateway-api, research]
draft: false
---

## 系列概述

本系列深入探討 Kubernetes API Gateway 的生態系統，以 **Envoy Gateway** 作為主要實作研究標的。涵蓋從概念、規範、架構到實戰部署的完整知識體系。

## 文件導航

| # | 文件 | 內容 | 適合對象 |
|---|------|------|---------|
| 1 | [K8s API Gateway 概述](k8s-api-gateway-overview) | 概念介紹、演進歷程、核心術語 | 初學者 |
| 2 | [Gateway API 規範深入](k8s-gateway-api-spec) | 資源模型、YAML 範例、設計細節 | 開發者/架構師 |
| 3 | [Envoy Proxy 核心架構](envoy-proxy-architecture) | 線程模型、Filter Chain、xDS 協議 | 進階工程師 |
| 4 | [Envoy Gateway 實作解析](envoy-gateway-implementation) | Controller 架構、CRD、AI Gateway | 平台工程師 |
| 5 | [實作對比與選型指南](envoy-gateway-comparison) | 5 大方案對比、效能矩陣、決策樹 | 技術決策者 |
| 6 | [實戰部署指引](envoy-gateway-deployment) | 快速啟動、部署模式、疑難排解 | DevOps/SRE |

## 學習路徑建議

```
初學路徑:
  1. K8s API Gateway 概述 (了解為什麼需要 Gateway API)
  2. Gateway API 規範深入 (學會寫 Gateway + HTTPRoute)
  3. 實戰部署指引 (動手操作)
  
進階路徑:
  1-3 同上
  4. Envoy Proxy 核心架構 (理解底層機制)
  5. Envoy Gateway 實作解析 (理解 Controller 如何工作)
  6. 實作對比與選型指南 (做出技術選擇)
```

## 關鍵發現

1. **Gateway API 是未來**: Ingress 的 annotation-sprawl 問題已被 Gateway API 的標準化欄位解決
2. **Envoy Gateway 是最純粹的實作**: 供應商中立、完整覆蓋 Gateway API 標準
3. **xDS 是核心機制**: Envoy 的動態配置能力來自 xDS 協議，ACK/NACK 自動回滾是關鍵安全特性
4. **IR 模式是最佳實踐**: Envoy Gateway 的中間表示 (IR) 架構提升了程式碼的可測試性和可維護性
5. **Policy CRDs 降低複雜度**: 將常見需求（JWT、限流、熔斷）封裝為簡單的 CRD

## 參考資源

- [Kubernetes Gateway API 官方文件](https://gateway-api.sigs.k8s.io/)
- [Envoy Gateway 文件](https://gateway.envoyproxy.io/docs/)
- [Envoy Proxy 架構概覽](https://www.envoyproxy.io/docs/envoy/latest/intro/arch_overview/arch_overview)
- [CNCF Blog: Understanding Gateway API](https://www.cncf.io/blog/2025/05/02/understanding-kubernetes-gateway-api-a-modern-approach-to-traffic-management/)
- [CNCF Blog: A Year of Envoy Gateway GA](https://www.cncf.io/blog/2025/06/11/a-year-of-envoy-gateway-ga-building-growing-and-innovating-together/)
- [DEV: xDS Deep Dive](https://dev.to/kanywst/xds-deep-dive-dissecting-the-nervous-system-of-the-service-mesh-3m5i)

---

> 📝 **研究日期**: 2026-05-15  
> 🔧 **研究工具**: Envoy Gateway v1.4, Kubernetes Gateway API v1.2  
> 💡 **下一步**: 搭建 Lab 環境進行實際測試
