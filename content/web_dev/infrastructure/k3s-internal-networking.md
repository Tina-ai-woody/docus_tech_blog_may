---

title: Zeabur 內網服務連線完整入門指南
description: 用最直覺的方式理解 Zeabur 內網機制，學會在同一個 Project 中讓服務彼此連線，避免 localhost 與 IP 的常見錯誤。
tags: [zeabur, devops, networking, docker, backend]
category: web_dev
date: 2026-02-24
----------------

在使用 Zeabur 部署專案時，許多新手第一個卡關的地方，往往不是程式碼，而是「服務怎麼互相連線」。資料庫連不上、API 呼叫失敗、容器之間彼此找不到對方，這些問題其實都跟「內網觀念」有關。

這篇文章會用最直覺的方式，帶你理解 Zeabur 的內網機制。你不需要先懂 Kubernetes，也不需要背誦太多名詞，只要掌握一個核心概念，就能解決 90% 的連線問題。

---

## 從一棟辦公大樓開始理解

想像你有一棟辦公大樓。這棟大樓就是一個 Zeabur Project。

在這棟大樓裡，有很多房間，每個房間代表一個服務。例如：

* nuxt-app：你的網站前端
* api-server：你的後端 API
* postgres：資料庫
* redis：快取系統

如果 API 要去找資料庫，它需要：

* 打電話到外面嗎？
* 查 IP 位址嗎？
* 經過大樓警衛嗎？

都不需要。

因為它們在同一棟大樓裡，只要知道「房間名稱」就能找到彼此。

這個「用名字找服務」的機制，就是 Zeabur 的內網服務連線原理。

---

## Zeabur 內網是怎麼運作的？

Zeabur 的底層是基於輕量化 Kubernetes 發行版 K3s 所建立的叢集架構。K3s 本質上仍然是 Kubernetes，只是更輕量，適合雲端平台快速部署。

在 Kubernetes 架構中，每個 Service 都會自動被註冊進內部 DNS 系統。因此，只要在同一個 Project（同一個叢集命名空間）裡，服務名稱本身就是它的內網地址。

這意味著：

> 每個 Service 名稱，就是它的內網 Host。

假設你的專案長這樣：

```
Project
 ├── nuxt
 ├── openclaw
 └── postgres
```

那麼它們彼此之間可以直接透過以下名稱連線：

```
nuxt
openclaw
postgres
```

不需要 IP，也不需要公開網址。

---

## 最常見情境：後端連資料庫

這是幾乎每個新手都會遇到的情境。

### 常見錯誤寫法

```env
DB_HOST=localhost
```

這裡的問題在於：

`localhost` 永遠只代表「目前這個容器自己」。

如果你的後端和資料庫是兩個不同的服務，那麼後端寫 `localhost`，只會找到自己，不會找到資料庫。

這就像你在自己房間裡喊自己的名字。

---

### 正確寫法

```env
DB_HOST=postgres
DB_PORT=5432
```

這代表：

> 去找名為 postgres 的服務。

Zeabur 會透過內網 DNS 幫你正確導向對應的容器。

---

## 服務之間互相呼叫 API

假設 openclaw 需要呼叫 nuxt 的 API。

你不需要：

* 使用公開網址
* 使用 VPS IP
* 開防火牆

只要這樣寫：

```
http://nuxt:3000
```

格式非常固定：

```
http://服務名稱:port
```

例如：

```
postgres:5432
redis:6379
nuxt:3000
openclaw:18789
```

這些都是內網呼叫。

---

## 為什麼不能用 IP？

在容器環境中，IP 是動態分配的。

例如今天 postgres 的 IP 是：

```
10.42.0.8
```

明天可能變成：

```
10.42.0.21
```

但「postgres」這個 Service 名稱不會變。

Kubernetes 透過 Service abstraction 層，讓你永遠用名稱存取服務，而不是直接對 Pod IP 連線。這也是容器編排系統的核心設計理念之一（參考 Kubernetes 官方 Service 說明）[1]。

因此：

> 永遠使用服務名稱，而不是 IP。

---

## 內網 vs 外網：為什麼差很多？

### 內網連線

```
http://postgres:5432
```

特點：

* 走叢集內部網路
* 不經過公開網際網路
* 速度快
* 安全
* 不產生額外對外流量

---

### 外網連線

```
https://postgres.zeabur.app
```

這會變成：

```
容器 → 出網 → 公網 → 再回來
```

這不但慢，還可能產生額外風險與流量成本。

在同一個 Project 裡，幾乎不應該使用公開網址互相呼叫。

---

## 如何確認內網是否正常？

如果你透過 SSH 進入容器或 VPS，可以用以下方式測試：

```bash
ping postgres
```

或：

```bash
curl http://nuxt:3000
```

如果成功回應，代表內網 DNS 與服務運作正常。

這是一個非常實用的排錯方式。

---

## 新手最常踩的三個坑

第一個是使用 `localhost`。
這只能連到自己，不能連到其他服務。

第二個是使用 VPS IP。
IP 在容器環境中是動態的，重建後會改變。

第三個是用公開網址打自己服務。
這會繞一大圈網路，變慢又不必要。

---

## 30 秒總結

在 Zeabur 中，只要記住一句話：

> 在同一個 Project 裡，用 Service 名稱互相連線。

格式永遠是：

```
http://服務名稱:port
```

當你理解這個觀念後，資料庫連線問題、API 呼叫錯誤、甚至部分容器啟動異常，都會變得容易分析。

如果你接下來想更深入理解為什麼有時候「服務可以連線，但 Startup Probe 卻失敗」，那就會牽涉到容器健康檢查與 Kubernetes Probe 機制。這會是下一個更關鍵的觀念。

---

## 參考資料

1. Kubernetes 官方文件 – Service 概念說明
   [https://kubernetes.io/docs/concepts/services-networking/service/](https://kubernetes.io/docs/concepts/services-networking/service/)

2. K3s 官方文件
   [https://docs.k3s.io/](https://docs.k3s.io/)
