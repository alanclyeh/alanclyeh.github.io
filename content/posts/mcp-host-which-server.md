---
title: "MCP Host 怎麼知道要呼叫哪個 Server？"
date: 2026-09-17T00:00:00+08:00
draft: false
tags: ["MCP", "LLM", "工具呼叫"]
categories: ["筆記"]
summary: "Host 不會靠語意去猜 Server。模型先依工具描述選出要呼叫哪個 tool；Host 再用連線時建好的對照表，把那次呼叫轉給對應的 MCP Server。"
description: "MCP Host 不是「聽懂」你要找哪個 Server。選工具是模型讀 description 做的語意判斷，路由到哪一台 Server 則是 Host 查名字對照表。拆開這兩層。"
ShowToc: true
TocOpen: true
---

很多人第一次看 MCP，會以為 Host 會「聽懂」使用者在講哪一個 Server。其實中間隔了兩次完全不同的判斷——**模型選工具，Host 只負責把那個工具送到已經連上的那條線**。

## 先認識三角色

![AI Agent 內部組成：Host 裡有 agent 迴圈、工具對照表與多個 Client，模型是 Host 呼叫的外部 API，每個 Client 一對一連到一台 MCP Server](images/mcp/mcp-roles.svg "AI Agent（MCP Host）的內部組成")

- **Host**：你正在用的 AI 應用。負責呼叫模型（模型本身多半是外部 API，不住在 Host 裡）、管有哪些 Server、把工具結果塞回對話。
- **Client**：Host 裡面的連線元件。**一個 Client 對一個 Server**，協定細節都走這條線。
- **Server**：對外提供 tools / resources / prompts 的程式。可以是本機 stdio process，也可以是遠端 HTTP。

Server 從來不跟模型直接講話。模型只跟 Host 說話；Host 再透過對應的 Client 去打 Server。

圖裡 Host 內部那張「工具對照表」是等一下的重點，它就是 Host 決定該打哪一台 Server 的依據。

## MCP Server 內部長什麼樣

![MCP Server 內部結構：透過 stdio 或 Streamable HTTP 傳輸，以 JSON-RPC 對外提供 tools、resources、prompts](images/mcp/mcp-server-inside.svg "MCP Server 的內部結構")

連上之後，Client 會問 Server「你有什麼」（例如 `tools/list`）。回傳的不是「我是行事曆 Server」這種口號，而是一張工具清單。Host 把各 Server 的清單合在一起，才交給模型。

## Host 怎麼知道要呼叫哪個 Server

這層沒有語意判斷，比較像查表。

1. **設定先告訴 Host 有哪些 Server**  
   例如設定檔裡列出 command、args，或遠端 URL。沒寫進去的，Host 根本不會連。

2. **啟動時一人一條線**  
   Host 為每個 Server 建一個 Client，做能力交換，再 `tools/list`。

3. **Host 自己記一本對照表**  
   大致是：`工具名稱（必要時加上 Server 前綴）→ 哪個 Client / 哪條連線`。  
   兩個 Server 都有叫 `search` 的工具時，Host 通常會加前綴避免撞名；模型最後看到的是已經消歧義過的名字。

4. **模型回「我要 call 這個 tool」之後**  
   Host 只看名字（或名字 + Server id）查表，對那條 Client 發 `tools/call`。  
   選錯 Server 在這層幾乎不會發生——除非對照表壞了，或模型吐了一個不存在的名字。

所以：Host 不是「聽懂要找行事曆」才去找行事曆 Server；它是「模型點了 `list_events`，而這名字當初是行事曆 Server 登記的」。

## 模型怎麼知道要呼叫哪個工具

這層才是語意。

Host 把工具清單交給模型時，每個工具是「名稱 + 自然語言描述 + 參數 JSON Schema」。模型在生成回應前，會像讀任何一段文字一樣去讀這些描述。

一筆工具定義大致長這樣：

```json
{
  "name": "list_events",
  "description": "列出使用者行事曆在指定時間區間內的所有事件，包含標題、起訖時間與地點。",
  "inputSchema": {
    "type": "object",
    "properties": {
      "from": { "type": "string", "description": "起始時間，ISO 8601 格式" },
      "to":   { "type": "string", "description": "結束時間，ISO 8601 格式" }
    },
    "required": ["from", "to"]
  }
}
```

模型看得到的就只有這些字。它不知道背後是 Google Calendar 還是一個 SQLite 檔，也不需要知道。

判斷比較像：

> 使用者現在想做的事，語意上跟哪個工具的描述最吻合？

使用者說「幫我查一下這週的行程」，句子裡不必出現 `calendar`。只要有一個工具寫著「讀取使用者的行事曆事件」，模型就能對上。

這也是為什麼工具的 description 寫得好不好，直接影響選對的機率，本質是 prompt engineering。

模型經過 function-calling / tool-use 微調後，學會在「這裡該呼叫工具」時，輸出符合 schema 的結構化資料（工具名稱 + 參數），而不是一段純文字。機制仍是「下一個 token 該是什麼」，只是被限制成合法的函式呼叫格式。

關鍵字只是把描述寫清楚的手段，不是 Host 或模型在做字面 matching。模型是在生成過程裡，把「使用者意圖」和「工具說明」對上。

### description 怎麼寫比較不會選錯

- **寫「什麼時候該用」，不只寫「這是什麼」**。`取得天氣` 不如 `查詢指定城市的目前天氣與未來三天預報；使用者問到氣溫、下不下雨時使用`。
- **把邊界講出來**。同一個 Server 裡有 `search_files` 和 `read_file` 時，各自補一句「只回傳路徑清單，不含內容」「需要完整路徑，不接受萬用字元」，模型才不會拿錯。
- **參數也要有描述**。schema 裡的 `description` 一樣會被讀。時間格式、單位、列舉值寫清楚，可以少掉一輪重試。
- **別把不常用的工具全掛上去**。清單愈長，模型愈容易挑錯或誤觸；工具數量本身就是雜訊。

## 那 resources 和 prompts 呢

只有 tools 是模型自己挑的。

- **resources** 比較像「可以拿來當上下文的資料」，多半由使用者或 Host 決定要不要塞進對話（例如在 Cursor 裡手動挑一個檔案）。
- **prompts** 是給人用的模板，通常長在 UI 上讓使用者點選，點了才展開成訊息。

所以「模型讀描述做語意選擇」這件事，講的是 tools 這一層。另外兩個的入口在使用者手上。

## 兩層串起來

```text
使用者：「這週有什麼行程？」
        │
        ▼
   模型讀工具清單（語意選 tool）
        │  例如：calendar__list_events({ from, to })
        ▼
   Host 查對照表（名字路由到 Server）
        │
        ▼
   Client → MCP Server：tools/call
        │
        ▼
   結果回到 Host，再餵給模型繼續寫回答
```

選錯工具：多半是描述含糊、工具太多、或名稱撞在一起。  
打到錯 Server：多半是對照表／命名空間的問題，不是模型「認錯 Server」。

## 小結

- **誰連哪些 Server**：人寫在 Host 設定裡。
- **誰決定呼叫哪個工具**：模型看 description。
- **誰決定打到哪一台 Server**：Host 用連線時建好的名字對照表。

MCP 把「怎麼講工具」標準化了；「何時該用哪個工具」仍然是模型讀文字的能力。
