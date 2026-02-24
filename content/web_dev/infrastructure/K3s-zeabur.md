---
title: Zeabur + K3s + Nuxt 部署架構完整解析
description: 深入理解 Zeabur 背後的 K3s 與容器架構，掌握從外部使用者請求到 Nuxt Nitro Server 的完整流量路徑。
tags: [zeabur, k3s, kubernetes, nuxt, deployment, devops]
category: web_dev/infrastructure
date: 2026-02-24
---

在現代雲端部署環境中，很多平台已經替我們把底層架構封裝得相當完善。當你在平台上按下「部署」按鈕時，背後其實發生了非常多層的協作運作。

這篇文章會帶你從「使用者在瀏覽器輸入網址」開始，一路走到真正執行 Nuxt Server 的那一層，幫你完整打通 Zeabur + K3s + Nuxt 的部署模型。

---

# 整體部署架構總覽

下面這張圖，是 Zeabur 實際運作架構的簡化版示意圖：

```
            Internet User
                    │
                    │ HTTPS Request
                    ▼
    ┌──────────────────────────┐
    │      Zeabur Gateway      │
    │ (Load Balancer / Ingress)│
    └─────────────┬────────────┘
                  │
                  │ Internal Routing
                  ▼
        ┌──────────────────┐
        │   K3s Cluster    │
        │ (Lightweight K8s)│
        └────────┬─────────┘
                 │
                 ▼
    ┌──────────────────────────┐
    │        Pod (Service)     │
    │  service-xxxxx           │
    └────────┬─────────────────┘
             │
             ▼
    ┌──────────────────────────┐
    │      Docker Container    │
    │──────────────────────────│
    │  Node.js Runtime         │
    │                          │
    │  install dependencies    │
    │  build project           │
    │                          │
    │  node .output/server     │
    │        /index.mjs        │
    └──────────┬───────────────┘
               │
               ▼
       Nuxt Nitro Server
       (Listening 0.0.0.0:PORT)
```


這張圖看似複雜，但只要逐層拆解，你會發現每一層其實各司其職。

---

# 分層理解架構設計

我們從最外層開始往內部拆解。

---

## 🌐 Layer 1 — Zeabur Gateway（入口層）

Zeabur 的最外層，是負責處理所有外部流量的 Gateway。

它的角色等同於：

```

Nginx / Cloud Load Balancer / Ingress Controller

```

這一層負責：

- 處理 HTTPS
- 管理網域
- SSL 憑證
- 外部流量入口
- 將請求轉發到正確的服務

當使用者在瀏覽器輸入：

```

[https://your-app.zeabur.app](https://your-app.zeabur.app)

```

外部世界其實只會看到 Gateway，並不知道後面實際跑了幾個容器、幾個 Pod。

---

## Layer 2 — K3s Cluster

很多人以為部署平台背後只是單純的一台 VPS，但實際上 Zeabur 的底層是基於 **K3s**。



K3s 是 Kubernetes 的輕量版本，專為雲端與資源有限環境設計。

這一層負責：

- 建立與管理 Pod
- 自動重啟失敗的服務
- Health Check 檢查
- 資源限制（CPU / Memory）
- 服務間網路管理

當你看到錯誤訊息像是：

```

Startup probe failed
Pod unhealthy

```

其實這不是應用程式在說話，而是 Kubernetes 在進行健康檢查並回報狀態。

---

## Layer 3 — Pod

在 Kubernetes 世界中，有一個很重要的觀念：

```

Service ≠ Container

```

正確的關係是：

```

Service
↓
Pod
↓
Container

````

Pod 是 Kubernetes 中最小的部署單位。你可以把它理解為：

> 一個應用程式的執行實體。

在平台上建立的一個 Service，本質上會對應到一個 Pod。這個 Pod 裡面通常包含一個主要容器，也可能包含 sidecar 容器。

---

## Layer 4 — Container（真正執行環境）

到了這一層，才是你熟悉的 Linux + Node.js 環境。



這裡通常會執行：

```bash
install dependencies
build project
node .output/server/index.mjs
````

容器的特性是：

* 有自己的檔案系統
* 有自己的執行環境
* 與 Host 隔離
* 透過虛擬網路對外通訊

這也是為什麼你有時候 SSH 進入主機，卻找不到專案原始碼的原因——因為程式碼存在於容器檔案系統，而不是主機本身。

---

## Layer 5 — Nuxt Nitro Server

當你使用 Nuxt 進行建置後：

```
.output/server/index.mjs
```

其實產生的是一個 Node HTTP Server。

在 **Nuxt 3** 中，這個 server 是由 Nitro runtime 所產生，負責處理：

* SSR 請求
* API routes
* 靜態資源回傳
* Middleware

本質上它會執行類似：

```js
createServer(...)
  .listen(PORT)
```

只要這個 Server 正常監聽對外 IP 與 PORT，K3s 就能透過內部網路將流量轉發給它。

---

# 真實流量路徑解析

完整請求流程如下：

```
Browser
   ↓
Internet
   ↓
Zeabur Gateway
   ↓
K3s Ingress
   ↓
Service
   ↓
Pod
   ↓
Container
   ↓
Nuxt Nitro
```

每一層都負責不同的工作：

* Gateway 負責入口與 SSL
* K3s 負責調度與健康檢查
* Pod 負責執行單位
* Container 負責執行環境
* Nuxt 負責應用邏輯

---

# 為什麼 HOST=0.0.0.0 這麼重要？

如果你的應用只監聽：

```
localhost
```

代表只允許容器內部存取。

這會導致：

* K3s 無法健康檢查
* Gateway 無法轉發流量
* 出現 connection refused
* Startup probe failed

而當你設定：

```
HOST=0.0.0.0
```

代表：

* 對 Pod network 開放
* Kubernetes 可以探測
* Gateway 可以成功轉發

這是容器化環境中非常關鍵的一個觀念。

---

#  主機檔案系統 vs 容器檔案系統

當你透過 SSH 連進雲端主機時，實際結構可能是：

```
Host Node
   └── K3s
        └── Pod
             └── Container ← App 執行處
```

容器具有獨立檔案系統，因此：

* 主機看不到容器內部專案
* 容器停止後檔案可能消失（除非使用 Volume）

理解這一層，會讓你在 Debug 部署問題時清楚知道該從哪裡查起。

---

# 核心理解總結

可以用一句話總結整個模型：

> Zeabur = K3s（Kubernetes）+ Container Runtime + 自動化 CI/CD

它幫你自動完成：

```
git push
   ↓
build container image
   ↓
deploy to K3s
   ↓
expose via Gateway
```

你寫的其實只是應用程式，但平台幫你處理了整個分層部署架構。

---

當你真正理解這五層架構後，你會發現部署錯誤、健康檢查失敗、Port 無法連線等問題，其實都可以在這個模型中找到定位點。

部署不再是黑盒子，而是一個可以被拆解、理解與掌控的系統。


