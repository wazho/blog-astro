---
title: 利用 Grafana Loki 高效收集與監控爬蟲日誌
description: 在 Lawsnote 服務期間，我們需要開發數百支網頁抓取程式，並有效率地管理這數百支 crawlers。當任何一支程式發生錯誤時，能夠即時通知相關人員並迅速定位問題，便是我們維運流程中的關鍵。
duration: 30min
date: 2025-04-06
---

在 [Lawsnote](https://about.lawsnote.com/) 服務期間，因應產品與專案需求，我們需要開發數百支網頁抓取程式 (Web Scraper) 與網頁爬取程式 (Web Crawler)，從政府主管機關的網站中擷取特定法學資料，並在資料梳理的同時，進行結構化處理並寫入資料庫中。

> 網頁抓取程式 (Web Scraper) 與網頁爬取程式 (Web Crawler)，以下統稱為 **crawlers**。

Lawsnote 提供的 [Lawsnote Search](https://lawsnote.com/) (法學搜尋引擎)、[企業法遵系統方案](https://blog.lawsnote.com/2022/12/lawsnote-regtech-compliance-solution/)、[Lawsnote Monta](https://monta.lawsnote.com/) (法遵解決方案) 等服務，皆高度講求資料的 **即時性** 與 **準確性**。

因此在我們的團隊中，除了例行性的程式開發外，也必須有效率地管理這數百支 crawlers。當任何一支程式發生錯誤時，能夠即時通知相關人員並迅速定位問題，便是我們維運流程中的關鍵。

為了達成這個目標，crawler 的日誌 (Logs) 便需要被妥善地收集與監控。這些日誌不僅包含執行狀態，也涵蓋整個過程中所爬取的請求 URL、HTTP 狀態碼、錯誤訊息 (Error Message) 等重要資訊。

在這篇文章中，我們將介紹如何利用 Grafana Loki 收集與監控這些日誌資料，並最終透過 Grafana 進行可視化展示，打造一個高效、穩定且實用的日誌監控方案。無論你是第一次接觸 Loki，或是想優化排程程式的日誌管理流程，希望這篇文章都能帶來一些實用的參考。

## 本文架構

這篇文章將從實務出發，介紹如何在 crawlers 專案中導入 Loki 與 Grafana，實作一套好用的日誌監控方案。主要內容包含：

1. **為什麼需要日誌監控？**
   描述 crawlers 的管理挑戰，以及 logs 在即時監控與錯誤追蹤中的重要性。

2. **使用 Grafana Loki 收集日誌資料**
   包含 Loki、Promtail 安裝與設定，如何將 crawler logs 傳送進 Loki。

3. **透過 Grafana 可視化監控日誌**
   如何建立 dashboard、撰寫 LogQL 查詢，以及設定警報通知機制。

4. **應用實例與延伸建議**
   實際案例分享與一些擴充應用的建議。

## 為什麼需要日誌監控？

當系統規模尚小、程式數量不多時，偵錯階段通常可以靠 **標準輸出 (stdout)** 和標準 **錯誤輸出 (stderr)** 直接在 terminal 或 console 上查看，抑或是將 log 輸出至檔案中。

倘若規模逐步擴大後，若採用相同方式進行管理，將會面臨以下幾個挑戰：

1. **日誌分散於不同主機或容器中，難以集中查閱**
   當程式數量一多，可能又分散執行在不同機器上，若沒有集中式的日誌收集機制，維運人員往往需要 SSH 到不同機器節點去翻找 log，不僅耗時，也容易遺漏資訊。

2. **缺乏統一查詢介面與格式標準，難以追溯問題源頭**
   每支程式可能使用不同的 log 格式與層級，導致即使 log 都存在，也難以快速篩選出錯誤來源或還原事件順序。

3. **日誌儲存空間有限，歷史紀錄易被覆蓋**
   若未搭配 **日誌輪替 (log rotation)** 機制，過舊的日誌可能會被覆蓋，導致事後調查時無法取得完整資訊。

4. **錯誤發生無法即時通知相關人員**
   傳統 log 記錄方式僅作為事後檢查之用，不易即時反饋問題。

5. **難以進行統計與視覺化分析**
   若想瞭解各程式的整體執行狀況，以 crawler 舉例有 *階段成功和失敗資訊*、*最常失敗的目標網站*、*HTTP 狀態碼分布* 等，將需自行撰寫腳本分析散落的 log，成效不彰。

### 什麼樣的資料適合收集？

在設計日誌系統時，並不是所有輸出都適合或需要被收集。我們應該根據系統特性與實際需求，挑選對後續除錯、分析與監控有實質幫助的資料。以開發 crawlers 的經驗來說，建議可以優先考慮以下幾類資訊：

1. **請求相關資訊**

    包括 *目標 URL*、*HTTP method*、*status code*、*response time* 等，能協助判斷目標網站是否正常、回應速度是否合理。

    ```text
    2025-01-01 12:00:00 - Requesting URL: https://example.com/api/data
    2025-01-01 12:00:01 - Response Status Code: 200
    2025-01-01 12:00:01 - Response Time: 200ms
    ```

2. **任務執行階段狀態**

    每個任務的開始與結束時間，以及中間執行的步驟 (如：下載 → 解析 → 寫入資料庫) 等，有助於還原執行流程與評估效能瓶頸。

    ```text
    2025-01-01 12:00:00 - Task started: Fetching data from example.com
    2025-01-01 12:00:01 - Parsing data...
    2025-01-01 12:00:02 - Writing data to database...
    2025-01-01 12:00:03 - Task completed: Data successfully fetched and stored.
    ```

3. **錯誤與異常**

    包括 *HTTP 錯誤*、*timeout*、*資料解析失敗* 等。這類訊息應清楚標示，方便集中查詢與設計告警條件。

    ```text
    2025-01-01 12:00:00 - HTTP Error: 404 Not Found
    2025-01-01 12:00:00 - Timeout Error: Request to https://example.com/api/data timed out.
    2025-01-01 12:00:00 - Parsing Error: Failed to parse JSON response from https://example.com/api/data
    ```

4. **環境、版本與其他自定義資訊**

    包括 *爬蟲版本*、*執行機器節點 IP*、*執行時間的環境變數*、*使用的函式庫版本* 等，有助於 debug 與還原特定部署狀態。

    ```text
    2025-01-01 12:00:00 - Crawler Version: 1.0.0
    2025-01-01 12:00:00 - Hostname: crawler-node-01
    2025-01-01 12:00:00 - IP Address: 10.0.0.15
    2025-01-01 12:00:00 - Node.js Version: v22.11.0
    2025-01-01 12:00:00 - Environment: production
    2025-01-01 12:00:00 - Dependencies: axios@1.6.8, cheerio@1.0.0-rc.12
    ```

## 如何設計日誌內容？

### 日誌等級

除了決定「記錄什麼」，另一個關鍵是「以什麼等級記錄」。合理區分 log 等級，不僅能協助後續查詢與視覺化統計，也能減少不必要的雜訊。

以下是常見的 log 等級分類及其適用情境：

- `debug` — 除錯時用的詳細資訊，例如變數值、執行流程細節。

  適合開發階段使用，通常在正式環境會關閉或降低權重。

- `info` — 代表系統正常執行的重要節點，例如任務啟動、請求成功、寫入資料完成等。

  是大多數 log 的主體，可用來觀察系統整體行為。

- `warn` — 非預期但尚可接受的狀況，例如重試、資料格式不符但能容錯處理。

  可視為潛在異常的訊號。

- `error` — 發生錯誤且可能影響任務執行的情況，例如 HTTP 失敗、解析錯誤、寫入失敗等。

  應重點監控與通知。

- `fatal` 或 `critical` — 系統無法繼續執行的嚴重錯誤，會中斷整個流程。

  可視為需立即介入的事件（有些 logger 可設定觸發終止程序）。

良好的 log 設計不只在於資訊完整，更在於資訊有層次。建議撰寫 log 時，根據情境合理設定等級，並搭配集中式日誌系統進行過濾與查詢。

### 如何判斷「哪些日誌值得記錄」？

從前面的章節中，我們可以歸納出一些通用的設計原則，協助判斷哪些日誌值得收集與追蹤：

- **可用於判斷系統是否「有在運作」**

  記錄任務開始、結束、請求發出與回應，是掌握系統存活狀態的基本資訊。這類訊息適合設為 `info` 等級，作為系統流程的日常記錄依據。

- **可用於觀察效能瓶頸或趨勢**

  例如 response time、任務執行時間等，可以透過 log 分析出哪個階段最耗時，協助做後續優化。這類資料也屬於 `info` 等級，對於後續視覺化有所幫助。

- **可用於追蹤錯誤的發生條件與上下文**

  記錄錯誤類型、來源 URL、發生時間與當下的資料，能讓除錯更快速、具體。這類訊息應設定為 `error` 或 `warn` 等級，並視嚴重程度觸發警報。

- **可用於重建問題發生時的執行環境**

  版本、IP、套件資訊等屬於部署狀態的資訊，是追蹤難以重現的 bug 時的重要依據。這些背景資訊可設定為 `info` 等級，於啟動時集中記錄一次。

這些原則不僅適用於 crawler 系統，也同樣適用於其他類型的應用系統，例如排程任務（cron jobs）、API server 等，都是日誌設計中值得參考的通則。

### 可結構化的日誌設計原則

在 JavaScript 實務中，在雖然 `console.log()` 是最簡單快速的輸出方式，但當我們希望日誌能被機器判讀、進行關鍵欄位過濾與分析時，「可結構化」就變得相當重要。

所謂 **可結構化日誌 (Structured Logging)**，指的是以明確欄位命名的格式輸出 log，常見為 JSON 格式。這樣的做法能讓日誌系統 (如 Loki) 更容易針對欄位進行搜尋與 **聚合 (Aggregation)**，提升查詢效率與可視化價值。

例如，若我們在爬取資料過程中發生了錯誤，傳統的 log 記錄方式可能會是：

```text
2025-01-01 12:00:00 - HTTP Error: 404 Not Found
2025-01-01 12:00:00 - Timeout Error: Request to https://example.com/api/data timed out.
2025-01-01 12:00:00 - Parsing Error: Failed to parse JSON response from https://example.com/api/data
```

以 JSON-based 的格式呈現，會變成：

```json
{
  "timestamp": "2025-01-01T12:00:00Z",
  "level": "error",
  "message": "HTTP Error",
  "url": "https://example.com/api/data",
  "status_code": 404,
  "error": "Not Found"
}
```

## 使用 Grafana Loki 收集日誌資料

### 使用 Docker Compose 建立環境

在以 Loki 為日誌系統的架構中，主要會扮演三種關鍵角色，分別負責日誌的儲存、傳送與呈現：

| 元件 | 角色 | 功能描述 |
|---|---|---|
| Loki     | 日誌儲存庫 | 日誌資料的儲存與查詢系統，負責接收、儲存與提供查詢介面。適合用來處理結構化或半結構化的 log 資料。 |
| Promtail | 日誌收集器 | 日誌資料的收集與傳送工具，常見用法是從檔案或容器 stdout/stderr 中擷取 log，轉送至 Loki。也支援透過標籤自動分類。 |
| Grafana  | 可視化介面 | 日誌資料的可視化與監控工具，負責提供 **使用者介面 (UI)**，讓使用者能夠查詢、分析與視覺化日誌資料。 |

以下提供我們將從零開始建立一份 **docker-compose.yaml**，目的是用來建立 *Loki* + *Promtail* + *Grafana* 的本地端測試環境，並搭配一個自定義的 Node.js 應用程式 container。

#### docker-compose.yaml 的基礎

首先，我們需要建立一個 **docker-compose.yaml** 檔案，並在其中定義 *Loki*、*Promtail*、*Grafana* 與 Node.js 應用程式的服務。

```yaml
version: '3.8'

networks:
  loki:

services:
  loki:
  # 參見下方的 loki 段落

  promtail:
  # 參見下方的 promtail 段落

  grafana:
  # 參見下方的 grafana 段落

  node-app:
  # 參見下方的 node 應用程式段落
```

> 完整 source code: [docker-compose.yaml](https://github.com/wazho/grafana-loki-practice/blob/main/docker-compose.yaml)

#### loki service

我們將額外的 config 資料夾掛載到容器中，並指定 Loki 的設定檔路徑。

細節的設定如下：

```yaml
services:
  loki:
    image: grafana/loki:3.4.1
    ports:
      - '3100:3100'
    volumes:
      - ./config:/mnt/config # 將 config 資料夾掛載到容器中
    command: -config.file=/mnt/config/loki-config.yaml
    networks:
      - loki
```

#### promtail service

我們同樣將 config 資料夾掛載到容器中，並指定 Promtail 的設定檔路徑。

另外，也會將 `/var/run/docker.sock` 掛載到容器中，這是為了讓 Promtail 可以與 host 的 Docker Daemon 溝通，取得正在執行中的容器清單與相關資訊。這樣就能從 container 的 stdout 和 stderr 中擷取 log，搭配後續小節的 **promtail-config.yaml** 設定檔，即可完成 logs 的收集與傳送。

細節的設定如下：

```yaml
services:
  promtail:
    image: grafana/promtail:3.4.1
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock # 掛載 Docker socket，讓 Promtail 可以讀取容器資訊
      - ./config:/mnt/config # 將 config 資料夾掛載到容器中
    command: -config.file=/mnt/config/promtail-config.yaml
    networks:
      - loki
```

#### grafana service

為了方便測試，我們將 Grafana 的帳號與密碼都設定為 `admin`。在第一次登入時，便不需要額外做設定。

細節的設定如下：

```yaml
services:
  grafana:
    image: grafana/grafana:10.2.3
    ports:
      - '3000:3000'
    environment:
      - GF_SECURITY_ADMIN_USER=admin # 將帳號設定為 admin
      - GF_SECURITY_ADMIN_PASSWORD=admin # 將密碼設定為 admin
    depends_on:
      - loki
    networks:
      - loki
```

#### 應用程式 service

最後，我們需要建立一個 Node.js 應用程式，並將其容器化為 **node-app** service。

細節的設定如下：

```yaml
services:
  node-app:
    build: ./app
    container_name: node-app
    labels:
      job: node-app
    restart: always
    networks:
      - loki
```

#### loki 設定檔: loki-config.yaml

在 loki service 中提到過，我們需要建立一個名為 **loki-config.yaml** 的設定檔，以供其使用。以下為設定內容：

```yaml
auth_enabled: false # 關閉權限驗證

server:
  http_listen_port: 3100

common:
  path_prefix: /tmp/loki # ①
  storage: # ②
    filesystem:
      chunks_directory: /tmp/loki/chunks
      rules_directory: /tmp/loki/rules
  replication_factor: 1 # ③
  ring: # ④
    kvstore:
      store: inmemory

schema_config:
  configs:
    - from: 2025-04-01 # ⑤
      store: tsdb # ⑥
      object_store: filesystem # ⑦
      schema: v13 # ⑧
      index: # ⑨
        prefix: index_
        period: 24h
```

`common` 各欄位說明：

1. `path_prefix`: 指定 Loki 儲存資料的根目錄，後續 chunks 與 index 等都會相對於此路徑建立。
2. `storage.filesystem`: 設定資料實際儲存位置，包括 chunks 與 rules。
3. `replication_factor`: 設定資料副本數，本地測試使用 `1` 即可。
4. `ring.kvstore.store`: 設定 ring 用來追蹤 instance 狀態的方式，單節點時使用 inmemory 即可，正式環境會使用 `etcd`、`consul` 等外部儲存。

`schema_config` 各欄位說明：

5. `from`: 指定此組 schema 的生效時間。即使只使用一組設定也必須加上，是 Loki 的必要欄位。
6. `store`: 使用 **使用時間序列資料庫 (tsdb)** 儲存資料，這是 Loki 的預設儲存方式，並且是撰文當下 Grafana 官方推薦的儲存方案。
7. `object_store`: 代表將 chunks 儲存在本機檔案系統（例如 `/tmp/loki/chunks` 目錄），適合用於本地測試。正式環境可改為 *S3*、*GCS* 等物件儲存方式。
8. `schema`: 代表使用 Grafana 官方的 schema 版本，以提供更佳的查詢效率與結構。若 schema 有變更版本，可參考最新的官方 [period_config](https://grafana.com/docs/loki/latest/configure/#period_config)。
9. `index`: 設定索引檔的檔名前綴與生成週期，這裡設為每天產生一份 (`24h`)，前綴為 `index_`。

當中需要留意的是 `from` 屬性 —— 因為它是 **required**。

Loki 會根據這個時間點來決定使用哪個 schema 來儲存資料，而這樣的設計讓我們可以在未來的版本中，針對不同的時間範圍使用不同的 schema，來定義出不同的 config 以避免一次性重建所有資料結構。

#### promtail 設定檔: promtail-config.yaml

接下來，我們需要建立一個名為 **promtail-config.yaml** 的設定檔，供前述的 promtail service 使用。

這份設定主要針對 Docker container 的 log 擷取，以下為設定內容：

```yaml
server:
  http_listen_port: 9080
  grpc_listen_port: 0

positions: # 紀錄上次讀取到的日誌位置，避免重複擷取
  filename: /tmp/positions.yaml

clients: # 定義要將日誌推送到哪個 Loki instance
  - url: http://loki:3100/loki/api/v1/push

scrape_configs:
  - job_name: docker
    docker_sd_configs: # ①
      - host: unix:///var/run/docker.sock
    relabel_configs: # ②
      - source_labels: [__meta_docker_container_name]
        target_label: job
        regex: '/(.*)'
        replacement: $1
```

`scrape_configs` 各欄位說明：

1. `docker_sd_configs`: 使用 Docker 的 **服務發現 (Service Discovery)** 機制，來自動獲取正在運行的 containers 資訊。
2. `relabel_configs`: **重新標籤 (Relabeling)** 的設定，這裡我們將抓取到的 container 名稱標籤化為 `job`，方便後續查詢與過濾。

#### 範例程式下載

> 完整範例 source code: [wazho/grafana-loki-practice](https://github.com/wazho/grafana-loki-practice)

範例程式中，包含了 `config` 資料夾與 Node.js 應用程式的 `Dockerfile` 等，可以使用以下指令來建立範例的測試環境：

```bash
# 下載範例專案
git clone git@github.com:wazho/grafana-loki-practice.git
# 進入專案資料夾
cd grafana-loki-practice
# 建立 docker-compose 環境
docker-compose up -d
```

啟動完畢後，後續章節將接續目前的進度，將開始介紹如何在 Grafana 中進行日誌的可視化與監控。

## 使用 Grafana 監控爬蟲運行狀態

![](/images/loki-crawler-logs/01-grafana-add-data-source-loki.png)
![](/images/loki-crawler-logs/02-grafana-add-data-source-loki-connection.png)
![](/images/loki-crawler-logs/03-grafana-create-dashboard.png)
![](/images/loki-crawler-logs/04-grafana-add-first-panel.png)

## 應用實例與延伸建議
