---
title: "三十幾行 Python，做一支 AI Agent"
date: 2026-09-25T00:00:00+08:00
draft: false
tags: ["AI Agent", "LLM", "Ollama", "Python"]
categories: ["筆記"]
summary: "不裝框架、不連雲端，用 Python 標準函式庫跟本機模型講上話，看清楚你跟 LLM 之間到底傳了什麼。"
description: "AI Agent 沒有神秘的地方。用 Python 標準函式庫和本機小模型寫一支最小的 agent，把送出去的 messages 和收回來的 response 逐個欄位拆開看。"
ShowToc: true
TocOpen: true
---

*這是《AI Agent 是怎麼跟 LLM 講話的》系列第 1 篇*

這個系列**不裝任何框架、不連雲端**，只用 Python 標準函式庫，用最簡單的方式試著打造自己的聊天機器人。

第一篇的目標很小：**讓它動起來**，順便把「跟 AI 對話」這件事拆開，看清楚底下到底傳了什麼。

## 用什麼模型？

要看清楚傳了什麼，最好是用 API 跟模型互動。可以用雲端模型 API，但要花錢，而且初次探索也不需要那麼厲害的模型。所以決定先安裝本機的小模型來玩玩，本機的好處是可以隨便亂試、看得到全部細節、而且不用管費用。

安裝就不多解釋，照著以下指令執行，Ollama 就會把模型包成 API 給你打：

```bash
brew install ollama
```

```bash
ollama serve
```

`ollama serve` 會一直跑著，佔住一個終端機。另外開一個視窗，把模型拉下來：

```bash
ollama pull ministral-3:8b
```

這次選的是 `ministral-3:8b`，在 16 GB 的 MacBook（M4）上跑得很順，每個回答大概三到七秒。它能處理繁體中文的對話，也支援工具呼叫。

拉好之後，輸入以下指令：

```bash
ollama run ministral-3:8b
```

待模型載入後，會出現熟悉的文字互動模式介面，可以先打些字隨便聊兩句，確認它活著。
但我們要的不是這個介面，是它底下那一層 LLM 的對話到底是怎麼運作的。

## 先不寫程式：用 curl 看一次原貌

`ollama serve` 開的是一個普通的 HTTP 服務，所以不需要任何 SDK，curl 就能叫它：

```bash
curl -s http://localhost:11434/api/chat -d '{
  "model": "ministral-3:8b",
  "messages": [{"role": "user", "content": "用兩句話說明什麼是 HTTP"}],
  "stream": false
}'
```

回來的東西長這樣（排版過，也略掉幾個這篇用不到的計時欄位）：

```json
{
  "model": "ministral-3:8b",
  "created_at": "2026-09-20T13:01:13.166765Z",
  "message": {
    "role": "assistant",
    "content": "HTTP（超文本傳輸協定，HyperText Transfer Protocol）是用於在網際網路上…（略）"
  },
  "done": true,
  "done_reason": "stop",
  "prompt_eval_count": 564,
  "eval_count": 146
}
```

一次對話的來回，全部在這裡面：

![一次對話的來回：你的 Python 程式 POST 一包含 messages 的 JSON 到 Ollama，Ollama 回一包含 message 與 token 數的 JSON，請求結束後兩邊都不保留對話](images/agent/one-round-trip.svg "一次對話的來回")

### 送出去了什麼

只有兩個欄位是必要的：

- **`model`** —— 要哪一顆模型回答
- **`messages`** —— 一個陣列，裡面是「到目前為止的對話」

`stream` 設成 `false` 是為了看得清楚：它會等模型講完，一次把整包 JSON 給你。設成 `true` 的話會一個字一個字推過來，適合做介面，但現在我們要看的是完整結構。

`messages` 裡的每一則都有 `role` 和 `content`。`role` 只有三種：

| role | 是誰說的 |
| --- | --- |
| `user` | 使用者說的話 |
| `assistant` | 模型說的話 |
| `system` | 給模型的規矩（等一下就會遇到它）|

這篇只會用到 `user`。另外兩個之後兩篇才會出場 —— 而且它們出場的理由會很具體，不是「規格上有所以要寫」。

### 模型回了什麼

| 欄位 | 是什麼 |
| --- | --- |
| `message.role` | 固定是 `assistant`，代表這句是模型說的 |
| `message.content` | 答案本身。這就是你平常在聊天視窗看到的那段字 |
| `prompt_eval_count` | 這一次**送進去**的 token 數 |
| `eval_count` | 這一次**生出來**的 token 數 |
| `done_reason` | 為什麼停下來。`stop` 是正常講完 |

`done_reason` 平常都是 `stop`，如果看到 `length`，代表它話還沒講完就被長度上限切斷了。

（被我略掉的那幾個 `*_duration` 是各階段花的時間，單位是奈秒，這篇用不到。）

那兩個 token 數字現在看起來沒什麼用，但它們是第二篇的主角 —— 而且其中一個現在就有話要說。

### 等一下，我只打了十幾個字

`prompt_eval_count` 是 **564**。

但我送出去的 `messages` 裡只有一句「用兩句話說明什麼是 HTTP」，連標點符號都算進去也才十幾個字。564 是哪來的？

因為 Ollama 在把 `messages` 交給模型之前，會先套一層**對話模板**。而這顆模型的模板裡，內建了一大段自我介紹：

```bash
ollama show ministral-3:8b --template
```

```text
[SYSTEM_PROMPT]You are Ministral-3-8B-Instruct-2512, a Large Language Model (LLM)
created by Mistral AI, a French startup headquartered in Paris.
You power an AI assistant called Le Chat.
Your knowledge base was last updated on 2023-10-01.
The current date is ...

# WEB BROWSING INSTRUCTIONS
You cannot perform any web search or access internet to open URLs, links etc.
...
```

所以**你以為只送了一句話，其實模型收到的是一整份說明書，加上你那句話**。這份說明書是模型廠商寫的，規定它是誰、能做什麼、不能做什麼。

這東西就叫 **system prompt** —— 剛剛那張 `role` 表裡的第三種。它的用途就是這樣：在對話開始之前，先把「你是誰、該怎麼做事」交代清楚。

> 這是這顆模型的情況。換一顆模型數字會不一樣，也有些模型的模板裡什麼都沒放。用 `ollama show <模型名> --template` 就能看到自己那顆是哪一種。

現在知道它存在就好。第三篇會自己寫一份，到時候會遇到一件很反直覺的事：自己寫了之後，送出去的 token 不但沒變多，**反而變少了**。

### 關鍵：這是一個一次性的請求

這件事現在看起來很平凡，但它是整個系列的地基：

> **打完這一次，連線就結束了。伺服器那端沒有留下任何東西。**

沒有「連線」、沒有 session、沒有「對話」這種東西。有的只是一次 HTTP POST，跟你打任何一支 REST API 沒有兩樣。

所有「AI 記得我們聊過什麼」的感覺，都是別人替你做出來的 —— 這件事第二篇會整篇講。

## 寫成 Python

看懂那包 JSON 之後，程式就沒什麼難的了。標準函式庫的 `urllib` 就夠：

```python
import json
import urllib.request

API = "http://localhost:11434/api/chat"
MODEL = "ministral-3:8b"


def ask(question):
    body = {"model": MODEL, "stream": False,
            "messages": [{"role": "user", "content": question}]}
    request = urllib.request.Request(API, json.dumps(body).encode(),
                                     {"Content-Type": "application/json"})
    with urllib.request.urlopen(request, timeout=300) as response:
        return json.load(response)


while True:
    question = input("\n你 > ").strip()
    if question == "/bye":
        break
    reply = ask(question)
    print("\n模型 >", reply["message"]["content"])
```


`ask()` 做的事跟剛剛那行 curl 一模一樣：組一包 JSON、POST 出去、把回來的 JSON 讀成 dict。外面的 `while` 迴圈負責問下一句。

幾個初學者容易卡住的地方：

- **`json.dumps(body).encode()`** —— 兩步。`json.dumps()` 把 Python 的 dict 變成 JSON 字串，`.encode()` 再把字串變成 bytes。HTTP 傳的是 bytes，不是字串，這一步不能省。
- **`{"Content-Type": "application/json"}`** —— 告訴伺服器「我送的是 JSON」。`urllib` 不會自己加，漏掉的話對方可能會用錯的方式解讀你的 body。
- **`timeout=300`** —— 五分鐘，看起來很誇張。但本機模型**第一次**被叫到時要先載進記憶體，十幾秒跑不掉；如果問題複雜、回答又長，一分鐘以上也很正常。設太短會在模型還在想的時候就被自己的程式掐斷。
- **`with ... as response`** —— 用 `with` 是為了離開區塊時自動把連線關掉，不必自己 `close()`。
- **`json.load(response)`** —— 注意沒有 `s`。`json.loads()` 吃字串，`json.load()` 吃像檔案一樣可以讀的東西，而 `response` 正好是這種。

## 補完三件事

上面那版能跑，但還不能用。三個地方要補：

**一、乾淨地離開。** 現在按 Ctrl-C 會噴一整片 traceback，很醜。而且如果不小心按到 Enter 送出空字串，它會認真地把空問題送給模型。

**二、連不上時要說人話。** 忘記開 `ollama serve` 的時候，現在會看到一串 `URLError`。改成一句「請先執行 ollama serve」對自己好一點。

**三、把 token 用量印出來。** 這個數字在第二篇會變成主角，現在先讓它露臉。

補完之後：

```python
print(f"跟 {MODEL} 聊天。輸入 /bye 或直接按 Enter 離開。")
while True:
    try:
        question = input("\n你 > ").strip()
    except (EOFError, KeyboardInterrupt):
        break
    if question in ("", "/bye"):
        break
    try:
        reply = ask(question)
    except urllib.error.URLError:
        print("連不上 Ollama。請先在另一個終端機執行 ollama serve")
        continue
    print("\n模型 >", reply["message"]["content"])
    print(f"[送出 {reply['prompt_eval_count']} tokens，回答 {reply['eval_count']} tokens]")
```

連 import 和空行都算進去，完整的檔案[三十幾行](https://github.com/alanclyeh/ai_agent_from_scratch/blob/main/01-minimal-agent/agent.py)。

跑起來長這樣：

```text
$ python3 agent.py
跟 ministral-3:8b 聊天。輸入 /bye 或直接按 Enter 離開。

你 > 我叫小明。

模型 > 你好，小明！很高兴认识你。有什么我可以帮助你的吗？比如：
- 你想了解什么话题？
- 需要规划行程、学习或工作上的建议？
- 或者只是随便聊天？

请告诉我你的需求！
[送出 558 tokens，回答 71 tokens]
```

能動了。

> 它打招呼的時候常常用簡體，問技術問題倒是規規矩矩的繁體 —— 這是這顆模型的習性。現在沒辦法管它，因為我們還沒有任何地方可以告訴模型「該怎麼講話」。第三篇會用一行字解決。

## 這就是 agent 的骨架

送訊息 → 收回答 → 再來一次。

後面每一篇都只是往這支程式加東西 —— 加一個 list、加一則訊息、加一個函式 —— **但這個骨架不會再變**。你平常在用的那些 AI 產品，最核心的那一圈也是這個形狀，只是外面包了很多層。

所以「AI Agent」這個詞，拆開來就是：一支替你組 JSON、送出去、把結果印出來的程式。

## 它不記得剛剛講過什麼

剛剛那段對話，我已經告訴過它我叫小明了。接著問它：

```text
你 > 我叫什麼名字？

模型 > 我無法知道你的名字，因為你在對我說話之前並沒有告訴我。
       如果你想讓我記住你的名字，可以告訴我哦！
[送出 561 tokens，回答 60 tokens]
```

中間隔不到十秒鐘，它就是不知道。

而且證據就在那行數字裡：上一輪送出 558 個 token，這一輪 **561**。如果「我叫小明」那一輪有被送出去，這次應該是 558 再加上它上一輪回答的 71 個 token 往上疊，而不是只多 3 個 —— 那 3 個只是「我叫什麼名字？」比「我叫小明。」長了一點。**「我叫小明」那句從來沒有被送出去過。**

程式沒有壞，模型也沒有壞。這是 LLM 本來就會有的行為，而原因就藏在這篇看過的東西裡面 —— 回頭看一眼 `ask()`：

```python
"messages": [{"role": "user", "content": question}]
```

每一次呼叫，`messages` 裡面**只有當下這一句**。

那要怎麼讓它記得？下一篇就修這個，關鍵的改動只有兩行。

## 小結

- 跟 LLM 溝通就是一包 JSON 送出去、一包 JSON 收回來，核心只有 `model` 和 `messages` 兩個欄位
- **每一次請求都是獨立的**，模型那端不會留下任何東西
- 三十幾行的 Python 就是一支 agent 的骨架，剩下的都是往上加

完整程式碼在 [ai_agent_from_scratch](https://github.com/alanclyeh/ai_agent_from_scratch) 的 `01-minimal-agent/`。

下一篇：它為什麼不記得我剛剛說過的話？（即將推出）
