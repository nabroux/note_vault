
## **🧩 一、什麼是 API Gateway？**


> **API Gateway 是介於 Client 與後端服務之間的反向代理層**，

> 它負責統一接收所有請求，並根據規則轉發至對應的微服務。


### **📘 簡單理解：**

```
Client（App / Web） → [ API Gateway ] → 多個 Backend Services
```

### **💬 功能本質：**

> 讓外部世界看到的是「一個統一的 API」，
> 而內部實際上有很多獨立的微服務。

## **⚙️ 二、API Gateway 為什麼需要存在？**

### **🧱 沒有 Gateway 時：**

- 每個前端都要知道各後端服務的位址。
- 安全性差（每個 service 都暴露在外）。
- 每個服務都得重複做驗證、限流、日誌、CORS、壓縮等

### **✅ 有 Gateway 後：**

- 統一入口 → 統一管理金鑰、權限、驗證。
- 後端服務隱藏 → 提高安全性。
- 可以做快取、限流、A/B測試、統計等。

## **🔑 三、API Gateway 的核心功能**
|**功能分類**|**說明**|
|---|---|
|🔁 **反向代理 / 路由 (Reverse Proxy)**|將 /user/* 請求轉發到 user-service，/order/* 轉到 order-service。|
|🔐 **認證與授權 (Authentication / Authorization)**|驗證 JWT、OAuth2、API Key，控制使用者權限。|
|🚦 **流量控制 (Rate Limiting / Quota / Circuit Breaker)**|防止惡意請求、平滑突發流量。|
|⚙️ **負載平衡 (Load Balancing)**|根據策略分配請求到多個實例。|
|🧱 **請求聚合 (Request Aggregation)**|一次呼叫多個後端服務，合併結果。|
|🧩 **轉換 (Transformation)**|修改 header、request body、API 版本轉換。|
|📊 **監控與日誌 (Monitoring / Metrics / Tracing)**|記錄每個請求狀態、延遲、錯誤。|
|📈 **快取 (Caching)**|常用查詢結果快取在 Gateway。|
|🧱 **CORS 與安全控制**|管理跨域、CSRF、Header 白名單等。|
|🔀 **灰度 / A/B 測試 / Canary 部署**|部分流量導向新版本以測試。|
|🛡️ **防火牆 / 攻擊防禦 (WAF)**|攔截 SQL Injection、XSS、DDoS。|

## **🌐 四、架構層級定位**

整體架構中，Gateway 位於 **Load Balancer 後面**、**微服務前面**：

```
[ Client ]
   ↓
[ CDN / WAF ]
   ↓
[ Load Balancer ]
   ↓
[ API Gateway Cluster ]
   ↓
[ Auth Service | User Service | Payment Service | ... ]
```

- LB：分流、健康檢查（L4/L7）
- Gateway：應用層邏輯（L7，處理 token、路由） 
- Backend：執行實際業務邏輯

## **🧠 五、API Gateway 在微服務架構中的角色**
| **元件**                | **職責**                                 |
| --------------------- | -------------------------------------- |
| **Service Discovery** | 動態知道有哪些服務存在（e.g. Consul, etcd, Eureka） |
| **API Gateway**       | 統一入口、統一規範、安全治理                         |
| **Service Mesh**      | 管理服務與服務之間的流量（內部）                       |
| **Backend Services**  | 執行實際業務邏輯                               |
📌 Gateway 是「**外部流量入口**」，
Service Mesh 是「**內部流量治理**」。


## **🧱 六、API Gateway 的部署模式**
|**模式**|**說明**|**特點**|
|---|---|---|
|**集中式 Gateway**|所有請求都經過單一 Gateway|簡單、統一，但可能成瓶頸|
|**分散式 Gateway（per service）**|每個服務都有自己的輕量 Gateway|擴展性強，運維複雜|
|**雲端託管型**|AWS API Gateway、GCP Apigee|零維運、功能齊全但貴|
|**Service Mesh Gateway**|使用 Istio / Linkerd 入口|適合 Kubernetes 微服務環境|

## **💡 七、常見實作框架**
|**類別**|**工具 / 服務**|**說明**|
|---|---|---|
|☁️ 雲端託管|AWS API Gateway、GCP Apigee、Azure API Management|零維運，企業級功能|
|🧩 開源型|**Kong (Lua / Nginx)**、**Traefik (Go)**、**Envoy (C++)**、**KrakenD (Go)**|主流選擇，彈性大|
|🧱 自架型|**Spring Cloud Gateway**、**Netflix Zuul**|適合 Java 生態系|
|⚙️ 輕量代理|**Nginx / OpenResty / HAProxy**|可自訂規則與 Lua plugin|

## **🔧 八、範例：Nginx 做簡易 API Gateway**

```
server {
    listen 443 ssl;
    ssl_certificate     /etc/ssl/cert.pem;
    ssl_certificate_key /etc/ssl/key.pem;

    location /user/ {
        proxy_pass http://user-service:8080/;
    }

    location /auth/ {
        proxy_pass http://auth-service:8080/;
    }

    location /payment/ {
        proxy_pass http://payment-service:8080/;
    }
}
```

這樣：

- 所有請求都統一從 Gateway 進入
- SSL/TLS 統一管理
- 可加上認證、限流、日誌

## **🧠 九、延伸功能與進階設計**
|**功能**|**實作方式**|
|---|---|
|🔐 **JWT 驗證**|Middleware 或 Plugin 驗簽、解析 claims|
|📈 **Rate Limiting**|Redis / local token bucket|
|📊 **Logging / Monitoring**|Prometheus + Grafana|
|⚡ **Circuit Breaker / Retry**|使用 Envoy 或 Kong plugin|
|🌍 **Multi-Tenant 支援**|根據 Header 或 Token 路由不同租戶服務|
|🔄 **API 版本控制**|/v1/* → 舊版服務、/v2/* → 新版服務|
|🧱 **GraphQL Gateway / BFF**|聚合多服務成單一 API（如 Apollo Gateway）|

## **🧰 十、API Gateway vs 其他元件對比**
| **功能**  | **API Gateway**      | **Load Balancer**   | **Service Mesh**  |
| ------- | -------------------- | ------------------- | ----------------- |
| 層級      | L7 (HTTP/gRPC)       | L4/L7               | L7 (內部流量)         |
| 核心任務    | 外部流量進入、治理            | 流量分配                | 內部服務流量治理          |
| 驗證 / 權限 | ✅ 有                  | ❌ 無                 | ✅ 內部憑證            |
| 限流 / 熔斷 | ✅ 有                  | ⚙️ 有限               | ✅ 自動              |
| 對象      | Client → Service     | Client → LB         | Service → Service |
| 常見產品    | Kong, Traefik, Envoy | Nginx, HAProxy, ALB | Istio, Linkerd    |

## **⚙️ 十一、部署架構參考**

```
Client
  ↓
[ CDN / WAF (Cloudflare) ]
  ↓
[ Load Balancer (AWS ELB) ]
  ↓
[ API Gateway Cluster (Kong / Traefik / Envoy) ]
  ↓
[ Auth Service | User Service | Order Service | ... ]
```

- SSL Offloading 在 LB 層
- API Gateway 處理 JWT、路由、限流
- 內部服務透過 Service Discovery 找彼此

## **🧩 十二、實務經驗建議**
|**規模**|**推薦做法**|
|---|---|
|🧱 小型專案 / MVP|Nginx 反向代理 + JWT 驗證|
|🧩 中型微服務|Kong / Traefik / Envoy Gateway|
|☁️ 雲端 Serverless|AWS / GCP API Gateway|
|🧠 大型企業|Kong + Istio / Apigee + Service Mesh|
