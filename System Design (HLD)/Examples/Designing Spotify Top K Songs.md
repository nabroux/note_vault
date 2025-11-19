
https://www.youtube.com/watch?v=HjazbLlrWxI

https://systemdesignschool.io/problems/topk/solution

```
1. Gather Requirements
2. API Design/Database Design
3. High-Level Design
4. Deep Dive
```

# 1. Gather Requirements

### Functional Requirements
1. K = 100 to 1000
2. All time, last time, last hour

### Non-functional Requirements
1. 10B songs played per day
2. 100M total songs
3. Latency 100ms to fetch top k songs
4. Latency for a song play to be included = 1 min （一首歌放多久才會列入考量）


# API Design

GET /top-k?k=1000&time=1mon
這個案例API相對沒有什麼複雜的


# High-level Design

Event Stream like (Kafka)

用 Flink 之類的工具進行 Fixed Sliding Window 計數 或是 arbitrary sliding window（事件發生才開始window 動態的）

**Tumbling window** is a fixed size window that does not slide. Events in the same window are **not overlapping** 直接用時間段區分實務上也有可能會這樣用，像Spotify推薦播放清單就是24h更新一次

甚至直接用bucket 儲存每一分鐘的歌曲及數量，就不用去維護sliding window外減少的狀況了，直接看範圍內的bucket總和

用 Redis 或資料庫儲存 Flink 資料 提供客戶端API

### global all time top k 解法：

用miniHeap 因為最上面是最小數字 如果新資料大於他 直接pop掉替換 再進行排序
資料量大如果每次都要reheapify會很耗時間，另外儲存 hash map Stores (VideoID, HeapIndex) pairs. 當某個元素的 priority（count）改變時，可以：
    - O(1) 找到它在 heap 裡的位置
    - 然後對那個位置做 **bubble up / bubble down** → O(log K)
這比：
- 每次更新都在 heap 裡線性搜尋那個 video → O(K)
好很多

為什麼一定需要Hashmap
> 「每一次新的歌曲被播放，你都必須先知道它的**最新播放次數**。
> 你若不知道這個數字，就沒辦法跟 Min-Heap 做比較。」


### 不同window的時間段的 top K 維護方式

資料流用bucket 每分鐘分裝 動態看window涵蓋範圍 or 用pointer指定範圍出來 同樣維護 heap 和 hashmap（資料隨著window pointer移動會有增減）

bucket還是會有window時間增減的處理，如果不這樣就會變成即時運算 bucket如果很短時間 表示數量會多 會算很久，如果像window維護的話 至少不用per event處理 而是一桶一桶批量處理 精準度可能稍有誤差 但會更省成本（bucket精度）




# Deep Dives

1. Single Point of Failure
2. Through put 吞吐量
3. Window 時間區間


新增 Worker （處理Flink那段的服務 每個會有一個heap結果），需要在進行一個Snapshot 因為Kafka不一定會儲存長期的資料 通常會設定一個過期時間 而且重播資料流可能會超高成本

快照機制 flink本身就有 該機制可以讓 單點故障後 重啟後不用重播資料 可以快速從上個checkpoint繼續，在大型服務如Spotify 一小時的event資料量就很可怕了。

每個 worker 處理 根據 song_id 做 partition 用hash可能是最好的方式

合併不同worker 會需要有個合併各個partition的Top K 服務 這個資料才能寫入到資料庫or Redis 給使用者

各個worker各自維護各自的heap和hashmap，最終結果需要合併，因為各個結果以這個案例會被放在不同worker去處理，但好處是每首個只會在一個worker的heap中 所以合併單純將每個worker提交的list進行排序即可，就算將全部worker的heap資料合併成一個超大list candidates 再排序也不會太扯 

因為 num_workers 通常不會太大（例如 10～100），
而 K（例如 100 或 1000）也不大，
所以 candidates 長度在 1e3～1e5 之間是能接受的。
- 對 candidates 再跑一次 top-K：
    - 用一個 **global Min-Heap（大小 K）**
    - 掃過 candidates，維護 global top-K

這個結果就可以寫入redis之類的讓用戶端call API 存取

如果真的太多，也可以分兩層合併，某地區（亞洲、歐洲）的先合併一次，全球的最終合併

也可以降頻更新，每個worker可能10秒上報一次自己的heap就很夠了



大致資料流
```
[Client]
  ↓ play events
[Event Collector]
  ↓
[Kafka topic: play_events]
  ↓
[Streaming Aggregation]
    - update play_count counters
    - maintain real-time Top-K
  ↓
[Redis Sorted Set / KV]
    → 用於「即時榜單」
  ↓
[Batch Aggregation (每日)]
    - re-compute official charts
    - fix late events
  ↓
[Final Top-K table]
    - Redis / BigQuery / ClickHouse
  ↓
[Charts API]
```

> Spotify 的 Top-K 排行榜不是在查詢時即時計算，
> 而是透過：

1. > **事件流（Kafka）**
2. > **Streaming 累計（Flink）**
3. > **Batch 校正（Spark）**
4. > **Redis Sorted Set 快取**
>   
> 事前把「各時間窗」的 Top-K 都算好，
> API 只是讀快取，所以能非常快。


---

# AI 總整理

# **1️⃣** Global Top-K 與 Sliding Window Top-K 的差異

### **✔ Global Top-K（全時間）**

- HashMap 中的 count **只會累加，不會遞減**
- 用於維護「全時間播放數」
- Heap 代表全時間的 Top-K
- 不需要 window eviction，不需要 -1
- 適合：
    - Spotify all-time top songs
    - YouTube all-time views
    - 統計報表、排行榜

### **✔ Sliding Window Top-K（固定時間視窗）**

- count **會遞增也會遞減**（事件進入 +1，離開 -1）
- 每個 window 需要維護自己的：
    - HashMap（window count）
    - Heap（top-k for that window）
    - begin / end 指標（或 offsets）
    
- 用於：
    
    - 最近 5 分鐘 / 1 小時 / 24 小時熱度
    - trending topics
    - fraud detection

👉 **兩者使用完全不同的 state，不可能共用同一份 HashMap。**

---

# **2️⃣** 為什麼需要 HashMap + HeapIndex？

### **✔ HashMap 的兩個用途（重要）**

1. **存 count（global 或 window count）**
2. **存 heapIndex → 讓你在 heap 裡 O(1) 找到 video**

### **✔ 沒有 heapIndex 的後果**

- 要更新某首歌的 count 時
    → 你會不知道它在 heap 裡哪個位置
    → 得掃整個 heap O(K)（很慢）
    
- 有 heapIndex 時：
    → 立即得到它在 heap 中的 index（O(1）
    → reheapify（O(log K)）

### **✔ 結論**

> **Heap（Top-K）、HashMap（count）、HeapIndex（位置）是三位一體，不可缺一。**

---

# **3️⃣** 為什麼 sliding window 一定需要雙指標？

因為 sliding window 的定義是：
```
count_in_window = 
    所有 timestamp ∈ [now - W, now] 的 event 次數
```

所以每次 window 往前滑動，需要：
- 新事件滑入 → +1
- 舊事件滑出 → -1

為了知道「哪件舊事件要被移除」，你需要 **事件序列（queue or log）**：
- end → 指向最新事件的位置
- begin → 指向 window 最舊事件的位置

這就是為什麼 systemdesignschool 用：
- Kafka 的 partition log
- 兩個 offset（begin offset / end offset）

當成 sliding window 的 queue。

---

# **4️⃣** 為什麼 Sliding Window 每個 Window 都要有自己的 Heap & HashMap？

因為每個 window 的「包含範圍」不同：
- 5 分鐘 window count ≠ 60 分鐘 window count
- 60 分鐘 window count ≠ 24 小時 window count

因此：

| **Window**  | **count_in_window** | **Heap**    |
| ----------- | ------------------- | ----------- |
| last 1 min  | 只計算 60 秒內           | Top-K(1min) |
| last 1 hr   | 只計算 3600 秒內         | Top-K(1hr)  |
| last 24 hrs | 只計算 24hr 內          | Top-K(24hr) |
➡ **Heap 必須分開**（不能混）
➡ **count 也必須分開**（但可存同一個 state object 裡）

---
# **5️⃣** Heap 無法 100% 精準維護 Sliding Window Top-K？（常見誤區）

這是你提出的一個超重要洞見：

> 「如果 heap 裡的歌曲一直掉分，但 heap 外有首歌沒掉分，它應該會擠進 heap，但會不會被忽略？」


答案是：
### **✔ 是，最簡單的 heap 實作** 
### **沒辦法保證 100% 精準 top-k**

原因：
- heap 只會調整「被事件觸及到」的元素
- heap 外的元素不會自己變強
- 除非事件「+1」它，否則它永遠不會被考慮

### **👍 真正精準的做法**

- 用 TreeMap / skiplist / balanced BST（例如 C++ multiset）
- 或每隔 X 秒做一次 full rebuild
- 或完全接受 approximate（很多大公司都接受近似 top-k）

---

# **6️⃣** 固定 Sliding Window（精準） vs Bucket Window（近似）

你提到：
  
> sliding window 不用 bucket 也能做嗎？

可以 → 用雙指標。但：
### **精準 sliding window（雙指標）**
- count 會 +1 / -1
- 重播舊事件
- 成本高但準確
### **bucket window（常用於實務）**

- 將時間切成固定 bucket，例如 1 秒、10 秒
- 每個 bucket 記錄 partial count
- window = 多個 bucket 的合併
- 比較便宜，但不是精準數值


# **7️⃣** 為什麼全時間（all-time）排行榜不需要 sliding window？

你提出：
> all-time top-k 不會保留全部 log，那怎麼算？

大多數公司作法是：

- 事件流在 streaming aggregator 內累積 count（HashMap）
- 定期 snapshot 到 Redis / Cassandra / BigQuery
- 長期 log 會 drop（只保留一段時間）
- next startup 可以透過 snapshots + log replay 恢復

「不保留所有 log」完全沒問題，因為 all-time count 永遠只會增。


# **8️⃣** 完整 Top-K 資料流程（簡化版）
  
你的理解已完全正確，可記進筆記：
```
Event Stream →
    Partition hash(videoId) →
        Local sliding windows: {1min,1hr,24hr}
        Local top-K (per window)
        Local global top-K
↓
Coordinator / Aggregator →
    Merge all local top-Ks →
    Produce final top-K result for each window
```