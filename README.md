# Other-MySQLOrchestrator

使用 Orchestrator 建立 MySQL HA（高可用），提供 Web UI 管理 MySQL 主從拓撲並支援自動故障轉移。

## 目錄

- [專案說明](#專案說明)
- [包含內容](#包含內容)
- [執行流程](#執行流程)
- [使用方法](#使用方法)
- [conf.json 關鍵設定說明](#confjson-關鍵設定說明)
- [建議注意事項](#建議注意事項)
- [參考資料](#參考資料)

---

## 專案說明

Orchestrator 是一套開源的 MySQL 高可用管理工具，提供：

- 自動偵測 MySQL 主從拓撲結構
- Web UI 視覺化管理主從關係
- 主節點故障時自動執行 Failover（切換）
- 支援 Raft 分散式模式部署

---

## 包含內容

```
Other-MySQLOrchestrator/
  docker-compose.yml          -- 啟動 orchestrator-alpine 容器
  conf/
    conf.json.default         -- Orchestrator 預設設定範本
    conf.json                 -- 實際使用設定（需從 default 複製後修改）
  orchestrator/               -- Orchestrator 相關資源
```

---

## 執行流程

```
[準備設定]
cp conf/conf.json.default conf/conf.json
    |
    v
[編輯 conf.json]
    |- MySQLOrchestratorHost      -- Orchestrator 後端 DB 位址
    |- MySQLOrchestratorPort      -- 後端 DB port
    |- MySQLOrchestratorDatabase  -- 後端 DB 名稱
    |- MySQLTopologyUser          -- 連線到 MySQL 拓撲的帳號
    |- MySQLTopologyPassword      -- 連線到 MySQL 拓撲的密碼
    |
    v
[啟動容器]
docker compose up -d
    |
    v
[Orchestrator Web UI]
http://localhost:3000
    |
    |- 自動探索 MySQL 拓撲
    |- 視覺化顯示 Master / Slave 關係
    |- 偵測故障並執行 Failover
    v
[MySQL HA 管理完成]
```

---

## 使用方法

### 1. 準備設定檔

```bash
cp conf/conf.json.default conf/conf.json
```

編輯 `conf/conf.json`，至少填入以下欄位：

```json
{
  "MySQLOrchestratorHost": "127.0.0.1",
  "MySQLOrchestratorPort": 3306,
  "MySQLOrchestratorDatabase": "orchestrator",
  "MySQLOrchestratorUser": "orchestrator",
  "MySQLOrchestratorPassword": "your_password",
  "MySQLTopologyUser": "orchestrator",
  "MySQLTopologyPassword": "your_topology_password"
}
```

### 2. 啟動服務

```bash
docker compose up -d
```

### 3. 開啟 Web UI

```
http://localhost:3000
```

### 4. 在 MySQL 節點建立 Orchestrator 帳號

在每個需要被 Orchestrator 管理的 MySQL 節點執行：

```sql
CREATE USER 'orchestrator'@'%' IDENTIFIED BY 'your_topology_password';
GRANT SUPER, PROCESS, REPLICATION SLAVE, RELOAD ON *.* TO 'orchestrator'@'%';
GRANT SELECT ON mysql.slave_master_info TO 'orchestrator'@'%';
FLUSH PRIVILEGES;
```

### 5. 新增 MySQL 實例到 Orchestrator

在 Web UI 的 `Discover` 頁面輸入 MySQL 主機位址與 port，或使用 API：

```bash
curl "http://localhost:3000/api/discover/your_mysql_host/3306"
```

### 6. 查看拓撲

Web UI 首頁 `Clusters` 會顯示目前偵測到的所有 MySQL 叢集與主從關係圖。

---

## conf.json 關鍵設定說明

```json
{
  "ListenAddress": ":3000",

  "MySQLOrchestratorHost": "127.0.0.1",
  "MySQLOrchestratorPort": 3306,
  "MySQLOrchestratorDatabase": "orchestrator",

  "InstancePollSeconds": 5,

  "BackendDB": "mysql",

  "RaftEnabled": false,

  "RecoverMasterClusterFilters": ["*"],
  "RecoverIntermediateMasterClusterFilters": ["*"]
}
```

| 參數 | 說明 |
|---|---|
| `ListenAddress` | Web UI 監聽 port |
| `MySQLOrchestratorHost` | Orchestrator 自身後端資料庫位址 |
| `MySQLOrchestratorDatabase` | Orchestrator 儲存拓撲資料的 DB 名稱 |
| `InstancePollSeconds` | 探測 MySQL 節點狀態的間隔（秒） |
| `BackendDB` | 後端資料庫類型（mysql 或 sqlite） |
| `RaftEnabled` | 是否啟用 Raft 分散式模式 |
| `RecoverMasterClusterFilters` | 哪些叢集啟用 Master Failover（`*` 表示全部） |
| `RecoverIntermediateMasterClusterFilters` | 哪些叢集啟用中繼 Master Failover |

---

## 建議注意事項

1. **`conf.json` 不要提交到版本控制**，內含資料庫帳密等敏感資訊。
2. **Orchestrator 後端資料庫需事先建立**，並建立對應的使用者帳號與 `orchestrator` 資料庫。
3. **每個受管理的 MySQL 節點都必須建立 Orchestrator 連線帳號**，且帳號需有足夠的權限。
4. **`InstancePollSeconds` 設定過短會增加 MySQL 負載**，建議維持預設 5 秒以上。
5. **啟用 `RaftEnabled` 需要至少 3 個 Orchestrator 節點**，單節點部署請保持 `false`。
6. **自動 Failover 前請確認 MySQL 節點的 `GTID` 或 binlog 設定正確**，否則 Failover 後可能出現資料不一致。
7. **重啟容器後 Orchestrator 會自動重新探索拓撲**，不需要手動重新設定。

---

## 參考資料

- [Orchestrator GitHub](https://github.com/openark/orchestrator)
- [Orchestrator 官方文件](https://github.com/openark/orchestrator/tree/master/docs)
