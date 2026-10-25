# 活網仔小精靈｜先做一個無聊的時候會想打開的聊天工具

## 為什麼想做這個？

我覺得我現在還不夠徹底了解課程的東西，所以想先做一個輕鬆的小東西，就是聊天工具。

我是一個什麼東西都看的人，不管是娛樂性的東西，還是知識性的東西，有時候在社群上看到冷知識、冷笑話或一些爛梗，都會覺得蠻有趣的。所以想做一隻「活網仔小精靈」，我無聊的時候可以丟一句話給它，它就像姐妹一樣跟我聊一下。

另外，我比較喜歡看 AI 短劇。可能今天想看復仇戲，明天想看無限流，就希望它幫我查，一次給一部、直接附連結，我就可以自己點開看看。

## 這一版想讓它做什麼？

冷笑話：直接講一則，不用解釋笑點。
諧音梗／爛梗：我丟一個詞，它就接一個梗，有點已讀亂回的感覺，但要有接到我在講什麼。
AI 短劇推薦：照我這次想看的題材找，只推一部 AI 短劇或 AI 漫劇，附劇名、簡短理由和連結。不是劇情裡有 AI，就算 AI 做的。
冷知識也有保留，不過不用每次都塞給我，我有問再給一則就好。

說話方式希望短一點、自然一點，不要太生硬、不要客服腔，也不用每次都問「還要不要更多」。

## 我是怎麼把 LINE 和 n8n 接起來的？

這次沿用課堂的 LINE Messaging API 串接，工作流是：

LINE 官方帳號
↓
Webhook（POST，接收 LINE 訊息）
↓
HTTP Request1（先顯示等待動畫）
↓
AI Agent（讀取訊息、依指令產生回覆）
↓
HTTP Request（把 AI 的回答傳回 LINE）

AI Agent 底下另外接 OpenAI Chat Model 和 Simple Memory。Web Search 是加在 Chat Model 裡的內建工具，不是另外一個主流程節點。畫面旁邊停用的 Message a model，是課程前一階段留下的版本，現在沒有接在主流程裡。

### 1. Webhook：LINE 訊息從這裡進來

先在 LINE Official Account Manager 啟用 Messaging API、關閉自動回覆，再到 LINE Developers 設定 Webhook URL 和 Use webhook。

n8n 的 Webhook 設為 POST，收到請求就先回應。測試時要先按聆聽；正式使用則先 Publish，再把 LINE 後台換成 Production URL，才不用每一次聊天都手動按測試。

收到的資料不只是一句文字，裡面還有這次回覆要用的 replyToken，以及區分使用者的 userId。

### 2. HTTP Request1：先讓 LINE 顯示等待提示

這個節點用 POST 呼叫 LINE 的 /v2/bot/chat/loading/start。
Body 的 chatId 放 Webhook 裡的 source.userId，讓使用者在 AI 還沒回完時，知道它正在處理訊息。

### 3. AI Agent：訊息和說話方式分開放

User Message 引用：
{{ $("Webhook").item.json.body.events[0].message.text }}

也就是把這次 LINE 傳來的文字交給 AI。System Message 則放角色、語氣和功能限制，例如：

「你叫『活網仔小精靈』，是陪使用者解悶、找好玩東西的姐妹。」
「繁體中文、台灣口語，接話自然、短句。」
「AI 短劇推薦：一次只推薦一部，必須是 AI 生成的短劇／AI 漫劇，排除真人演出的短劇。」

模型使用 OpenAI Chat Model 的 GPT-5-mini。找劇時要求先用 Web Search 搜尋，再核對官方 AI 劇分類或 AI 劇榜；查不到就明說，不能自己編劇名、網址或劇情。

### 4. Simple Memory：記住是同一個人在聊天

Session ID 選 Define below，Key 放：
{{ $("Webhook").item.json.body.events[0].source.userId }}

Context Window Length 設 10。replyToken 每次訊息都會換，所以記憶不能拿它當 Key；這裡用 userId 區分不同人的對話。課程版有用名字和喜好做過兩輪測試，能記住前面講的內容。

### 5. HTTP Request：把答案送回 LINE

最後用 POST 呼叫：
https://api.line.me/v2/bot/message/reply

Headers 設 Content-Type: application/json，以及 Authorization: Bearer 加 LINE Channel Access Token。這個權杖沒有放進公開截圖或作品說明。

Body 的 replyToken 引用原本的 Webhook；messages 裡 type 是 text，文字放 AI Agent 的 output。文字使用 JSON.stringify($json.output)，避免 AI 回覆有換行或引號時，把 JSON 弄壞。

## 實際傳了什麼？它回了什麼？

以下是實際測試，已移除對話中的姓名和帳號稱呼。

我：講一個冷笑話
它：為什麼數學書很憂鬱？
因為它有太多問題。

我：拿珍珠奶茶接一個諧音爛梗
它：珍珠奶茶？「真煮奶茶」，喝了才知道是不是真心煮的。

我：我想看復仇題材的 AI 短劇，或者現在熱門的。
它：推薦《拳拳AI意》，並附上紅果詳情連結。

第三個其實沒有做對。打開官方頁面後，看到它列了真人演員；「角色偽裝成 AI」不代表整部作品是 AI 生成的。所以這一輪不能算 AI 短劇推薦成功。

至少有確認，換了訊息之後，它會從冷笑話換成諧音梗、再換成找劇，不是一直回同一段。但找劇還需要多確認一次內容是不是符合我要的。

## 遇到什麼問題？後來改了哪裡？

我發現節點分別拉出來都還好，但把它串起來的時候，就會想說：為什麼東西又跑不動？累的時候真的會覺得煩躁。

這次實際遇到幾個地方：

中間多放了一個 AI 節點，後面的 HTTP Request 就不能繼續從「前一個節點」拿 LINE 原始資料，要改成明確引用 Webhook 的 replyToken。AI Agent 的回答則是 output，和前一版 Message a model 的輸出位置不同。

回覆公式也修過。欄位混進舊公式時，LINE 收不到回覆，後來清掉重填，再用新訊息測試。

語氣方面，課程版叫小花，會放很多 Emoji、有時候還會寫詩。我覺得蠻生硬的，所以改成像姐妹聊天，不要每次自我介紹，也不要一直問我要不要更多。

找劇方面，原本會推兩部真人劇，後來改成一次一部、只找 AI 的。測試又發現它把「劇情有 AI」誤當成「AI 生成」，所以補上官方 AI 劇分類的核對要求，明確排除《拳拳AI意》，也加入已核對的 AI 劇資料作為參考。

目前 LINE 收發、課程版記憶、冷笑話和諧音梗已測過。AI 劇分類修正版已發布，但還沒從 LINE 重測；改成無限流的測試也還沒完成。我不想因為它有回一個連結，就直接說這個功能做好了。

## 接下來想延伸做什麼？

因為我是什麼東西都看的人，娛樂性的、知識性的內容都會看，所以之後很希望可以做到清楚的整理，不只是看過就算了。

在板上有看到有人做 Notion 的整理。但我自己更常用 Google 家的工具，所以會想往 Google 的工具去做。像是把看到的連結、內容主題和重點整理起來，之後比較找得到。至於要放 Google Sheets、Google Docs，還是搭配 Drive，想等自己更懂一點再決定。

另外看到有人用 Apps Script，也覺得蠻感興趣，之後會想試試看。這次就先從一個輕鬆、自己會想用的聊天小工具開始。

## 流程截圖與作品連結

GitHub：https://github.com/Immataipeithing/active-net-line-sprite
流程截圖：作品庫內的 workflow.jpg。

這次公開的是流程圖和實作說明，不包含 LINE 存取權杖、API 金鑰或對話個資，也不是一份匯入就能啟用的工作流檔案。

