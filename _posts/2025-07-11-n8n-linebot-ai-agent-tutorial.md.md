---
title: "n8n 實戰：用 No-Code 打造Excel查詢機器人"
date: 2025-07-11 14:30:00 +0800
categories: [實作教學, n8n]
tags: [n8n, webhook, 自動化實作]
image:
  path: /assets/img/posts/n8n-webhook.jpg
  alt: n8n 實戰
pin: true
---

## 前言

最近把之前用傳統後端架設的 LineBot 讀經進度查詢機器人遷移到 n8n 平台，整個過程讓我深刻體會到 no-code 工具的威力！從原本需要 2-3 天的開發時間，縮短到半天就能完成部署。

Excel 的資料是先前已經整理過的，這次目的是透過 n8n 結合多個 AI Agent 分析字句後到 Excel 中取得符合條件的資料。今天想和大家分享完整的實作過程，包括 AI Agent 角色設定與工作流程設計 🛠️

> 本文提供完整的實作指南，適合想要學習 n8n 的朋友！
{: .prompt-info }

### 與傳統架構的對比

| 處理階段     | 舊架構 (Node.js)         | 新架構 (n8n + AI)      |
| ------------ | ------------------------ | ---------------------- |
| **語言理解** | 自然語言套件（容易錯誤） | 多 AI 模型（準確度高） |
| **對話記憶** | 無或需要自建             | Simple Memory 節點     |
| **錯誤處理** | 需要寫程式碼             | 視覺化條件節點         |
| **擴展性**   | 需要重新部署             | 拖拉節點即可           |
| **維護性**   | 需要程式知識             | 視覺化調整             |



## n8n 實際工作流程架構

### 完整流程圖解析

根據我在 n8n 中建立的實際工作流程，整個系統採用了智能分流的架構設計：

![n8n LineBot 工作流程](/assets/img/posts/n8n-webhook.jpg)
*實際的 n8n 工作流程圖*

### 核心工作流程

**1. 訊息接收階段**
- **Webhook 觸發器**：接收來自 LINE Bot 的訊息
- **訊息預處理**：透過 `Edit Fields` 節點格式化輸入資料

**2. 智能意圖識別**
- **第一個 AI Agent**：使用 Anthropic Claude 進行語意分析
- 判斷使用者意圖是「查詢資料」或「閒聊對話」
- 透過 `IF` 條件節點進行智能分流

**3. 雙軌處理機制**

**路徑 A：閒聊對話處理**
- **第二個 AI Agent**：專責聊天回應
- 使用 `Simple Memory` 節點維護對話記憶
- 提供自然的聊天互動體驗

**路徑 B：資料查詢處理**
- **第三個 AI Agent**：分析查詢需求的完整性
- **條件判斷**：確認使用者是否已提供完整查詢資訊
- **Excel 資料查詢**：透過 `Query Reading` 節點從 Excel 檔案中搜尋符合條件的資料
- **資料過濾**：使用 `Filter` 節點篩選結果

**4. 統一回應**
- 所有處理路徑最終匯聚到 `chatBot` 節點
- 透過 `Respond to Webhook` 節點統一回應給 LINE Bot

## 實作步驟詳解

### 步驟 1：n8n 環境準備

**Zeabur 快速部署：[點擊連結](https://zeabur.com/referral?referralCode=sheentrailstudio)**

在模板輸入關鍵字：n8n → 直接使用五倍學院
![](/assets/img/posts/zeabur.jpg)

輸入 domain 名字，等等透過這條網址編輯 n8n
![](/assets/img/posts/zeabur1.jpg)

建置完覺得想用 redis，在這邊按下『＋』
![](/assets/img/posts/zeabur_redis.jpg)

選擇 Redis
![](/assets/img/posts/zeabur_redis2.jpg)

大功告成
![](/assets/img/posts/zeabur_redis2.jpg)

### 步驟 2：Webhook 設定

1. 在 n8n 中建立新的 Workflow
2. 添加 `Webhook` 觸發節點
3. 設定 HTTP 方法為 `POST`
4. 複製 Webhook URL，稍後將用於 LINE Bot 設定

### 步驟 3：AI Agent 配置

#### **AI Agent 1：意圖識別專家**

負責分析使用者訊息，判斷是「查詢讀經進度」還是「閒聊對話」。

```javascript
// 在 Anthropic Chat Model 節點中的系統提示詞
const systemMessage = `
你是意圖識別專家，專門分析用戶對話的真實意圖。

# 任務
分析用戶訊息，判斷是「查詢讀經進度」還是「閒聊對話」。

# 識別標準

## 🔍 查詢讀經進度
- 包含日期：今天、明天、04/29、4月29日、昨天等
- 包含讀經關鍵字：進度、讀經、聖經、經文
- 包含計劃名稱：一年讀經、三年讀經、聖經旅行團
- 包含查詢詞：查詢、查、找、要、需要
- 包含範圍詞：全部、所有、整個

## 💬 閒聊對話
- 問候語：你好、嗨、早安、午安、晚安、安安
- 關懷話語：你好嗎、最近如何、身體健康嗎
- 感謝話語：謝謝、感恩、讚美主、感謝神
- 分享心情：今天很開心、心情不錯、感謝主
- 功能詢問：你能做什麼、怎麼使用、有什麼功能
- 一般對話：沒什麼、隨便聊聊、無聊

# 輸出格式

## 查詢讀經進度
{
  "intent": "query_reading",
  "confidence": 0.9,
  "userMessage": "用戶原始訊息"
}

## 閒聊對話
{
  "intent": "casual_chat",
  "confidence": 0.95,
  "chatType": "greeting|care|thanks|sharing|help|general",
  "userMessage": "用戶原始訊息"
}

請分析用戶訊息，只輸出JSON格式，不要其他內容。
`;
```

#### **AI Agent 2：聊天機器人**

專門處理閒聊對話，提供溫暖的互動體驗。

```javascript
const chatSystemMessage = `
你是一個溫暖友善的讀經助手機器人，專門陪伴使用者聊天。

# 角色特點
- 溫暖、友善、有同理心
- 具有基督教背景知識
- 善於傾聽並給予適當回應
- 會適時關懷使用者的靈性生活

# 回應原則
- 保持簡潔，約50字以內
- 語氣親切自然
- 可以適時分享聖經金句
- 避免說教，重點在陪伴

請根據使用者的訊息給予適當回應。
`;
```

#### **AI Agent 3：查詢需求分析師**

分析使用者的查詢需求是否完整，並提取查詢條件。

```javascript
const queryAnalysisMessage = `
你是查詢需求分析專家，負責分析使用者的讀經進度查詢需求。

# 任務
分析使用者訊息，判斷查詢資訊是否完整，並提取查詢條件。

# 必要資訊
- 日期：今天、明天、具體日期等
- 計劃類型：一年讀經、三年讀經等（可選）

# 輸出格式

## 資訊完整
{
  "status": "complete",
  "date": "2025-07-12",
  "plan": "一年讀經",
  "query": "formatted query for Excel"
}

## 資訊不完整
{
  "status": "incomplete",
  "missing": "date|plan",
  "question": "請問您想查詢哪一天的讀經進度呢？"
}

請分析使用者訊息，只輸出JSON格式。
`;
```

### 步驟 4：條件分流邏輯

在 `IF` 節點中設定條件判斷：

```javascript
// 判斷是否為查詢意圖
{{ $json.intent === 'query_reading' }}
```

### 步驟 5：Excel 資料查詢

在 `Query Reading` 節點中設定 Excel 查詢邏輯：

```javascript
// 假設 Excel 檔案結構包含：日期、計劃名稱、經文內容
const queryConditions = {
  date: $json.date,
  plan: $json.plan || '預設計劃'
};

// 查詢邏輯將根據條件篩選 Excel 資料
```

### 步驟 6：Memory 記憶功能

使用 `Simple Memory` 節點維護對話記憶：

- **Memory Key**: 使用使用者ID作為記憶鍵值
- **Memory Size**: 設定記憶對話輪數（建議5-10輪）
- **Memory Type**: 選擇適合的記憶類型

## 關鍵技術特點

### 1. 智能分流機制
透過第一個 AI Agent 準確識別使用者意圖，將不同類型的請求導向專門的處理路徑，提高回應準確性。

### 2. 專業化 AI Agent
每個 AI Agent 都有明確的職責分工：
- **Agent 1**: 意圖識別
- **Agent 2**: 聊天回應  
- **Agent 3**: 查詢分析

### 3. 記憶與上下文
透過 `Simple Memory` 節點，機器人能夠記住對話歷史，提供更自然的互動體驗。

### 4. 容錯處理
在關鍵節點設置錯誤處理邏輯，確保即使某個環節失敗，也能提供基本的回應。

## 優化建議

### 1. 效能優化
- 設定 AI Agent 的回應時間限制
- 使用快取機制減少重複查詢
- 優化 Excel 檔案結構提高查詢效率

### 2. 使用者體驗
- 加入 typing indicator 提示
- 設定預設回應處理異常情況
- 提供查詢格式說明

### 3. 監控與分析
- 記錄使用者查詢模式
- 分析 AI Agent 的準確率
- 設定告警機制監控系統狀態


## 結語

透過 n8n 結合多個 AI Agent 的架構，我們成功打造了一個智能的 Excel 查詢機器人。這個方案不僅大幅縮短了開發時間，還提供了比傳統程式碼更好的可維護性和擴展性。

對於想要入門 no-code 開發的朋友，這個專案是一個很好的起點。透過視覺化的節點配置，即使沒有深厚的程式背景，也能快速建構出功能完整的智能應用。

未來我們還可以進一步擴展功能，比如加入語音回應、圖表生成、或是整合更多的資料源。n8n 的彈性架構讓這些擴展變得輕而易舉。

> 完整的 n8n 工作流程檔案和更多技術細節，歡迎追蹤我的 GitHub 專案！
{: .prompt-tip }


