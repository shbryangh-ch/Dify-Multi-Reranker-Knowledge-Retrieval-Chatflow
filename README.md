# Dify Multi-Reranker Knowledge Retrieval Chatflow

## 專案簡介

這個專案是一個可匯入 self-hosted Dify 的 Chatflow 範例，用於在同一個 Dify App 內透過 API 輸入切換不同的 reranker。

原本如果要比較三種 reranker，可能需要建立三個獨立的 Dify 專案，並在 Jupyter Notebook 或外部程式中分別呼叫三組 API key。這份 workflow 將三條 reranker 路徑整合到同一個 Dify 專案中，只要在 API 的 `inputs` 裡指定 `reranker_name`，就可以透過 If/Else 節點自動選擇對應的知識檢索流程。

## GitHub Repository Title

**Dify Multi-Reranker Knowledge Retrieval Chatflow**

## 功能特色

- 單一 Dify Chatflow 入口
- 透過 `reranker_name` input 切換不同 reranker
- 使用 If/Else 節點串接三條 knowledge retrieval 分支
- 不經過 LLM 生成答案，主要回傳知識庫檢索結果
- 適合搭配 Jupyter Notebook 做 reranker 批次比較
- 可用於 self-hosted Dify 環境

## Workflow 架構

```text
API Request
  |
  |-- query
  |-- inputs.reranker_name
  |
Start Node
  |
If/Else: reranker_name
  |
  |-- BGE --------------------> Knowledge Retrieval
  |-- contains "msmarco" -----> Knowledge Retrieval
  |-- contains "Qwen3-reranker-4B" -> Knowledge Retrieval
  |-- default ----------------> Knowledge Retrieval
  |
Code Node: 檢查是否有檢索結果
  |
If/Else: 是否有資料
  |
  |-- 有資料 -> 整理成 list -> 合併成文字 -> Answer
  |-- 無資料 -> 直接回覆
```

## 支援的 reranker 輸入值

目前 workflow 透過 `reranker_name` 判斷要走哪一條檢索分支：

| reranker_name 條件 | 說明 |
| --- | --- |
| `BGE` | 走 BGE 對應的知識檢索分支 |
| 包含 `msmarco` | 走 msmarco 對應的知識檢索分支 |
| 包含 `Qwen3-reranker-4B` | 走 Qwen3-Reranker-4B 對應的知識檢索分支 |
| 其他值 | 走預設分支 |

> 實際 reranker 模型、知識庫 ID、top_k、score threshold 等設定，請以匯入 Dify 後的節點設定為準。

## 依賴套件

此 Dify App YAML 內包含以下 marketplace plugin 依賴：

- `langgenius/localai`

匯入前請確認你的 self-hosted Dify 環境已安裝或可使用這些 plugin，並且相關模型服務已設定完成。

## 如何匯入 Dify

1. 開啟 self-hosted Dify 後台。
2. 進入 Studio / 工作室。
3. 選擇匯入 DSL / YAML。
4. 上傳此專案中的 Dify YAML 檔案。
5. 匯入後檢查以下項目：
   - Knowledge Base 是否正確對應
   - reranker model provider 是否可用
   - API key / model endpoint 是否已設定
   - If/Else 條件是否符合你要傳入的 `reranker_name`
6. 發布 App，取得 API key。

## API 呼叫範例

以下範例使用 Dify Chat Messages API：

```python
import requests

url = "http://localhost/v1/chat-messages"
api_key = "YOUR_DIFY_APP_API_KEY"

headers = {
    "Authorization": f"Bearer {api_key}",
    "Content-Type": "application/json",
}

payload = {
    "inputs": {
        "reranker_name": "BGE"
    },
    "query": "請問 RAG 和 fine-tuning 有什麼差別？",
    "response_mode": "blocking",
    "user": "reranker-eval-user"
}

response = requests.post(url, headers=headers, json=payload)
response.raise_for_status()

result = response.json()
print(result)
```

切換 reranker 時，只需要改 `reranker_name`：

```python
payload["inputs"]["reranker_name"] = "Qwen3-reranker-4B"
```

## 搭配 Jupyter Notebook 批次比較

可以使用同一個 Dify App API key，對同一組問題分別傳入不同的 `reranker_name`：

```python
import time
import pandas as pd
from tqdm.notebook import tqdm

RERANKERS = ["BGE", "msmarco", "Qwen3-reranker-4B"]

questions = [
    "什麼是向量資料庫？",
    "RAG 和 fine-tuning 有什麼差別？",
    "reranker 在檢索流程中扮演什麼角色？",
]

all_rows = []

for question in tqdm(questions):
    for reranker_name in RERANKERS:
        start = time.time()

        result = call_dify_chatmessage(
            only_return_str=False,
            inputs={"reranker_name": reranker_name},
            query=question,
            api_key="YOUR_DIFY_APP_API_KEY",
            response_mode="blocking",
            user="reranker-eval-notebook",
            max_retry=3,
            debug=False,
        )

        latency = time.time() - start

        all_rows.append({
            "question": question,
            "reranker": reranker_name,
            "latency_sec": latency,
            "raw_result": result,
        })

df_results = pd.DataFrame(all_rows)
df_results
```

## 建議的評測指標

如果只是初步比較，可以先觀察：

- 每個 reranker 回傳的 Top-K 文件
- 回傳文件是否符合問題意圖
- 三種 reranker 的 Top-K overlap
- API latency
- 是否出現空結果

如果後續有人工標註的正確文件，可以進一步計算：

- Recall@K
- MRR
- nDCG@K
- Hit Rate

## 注意事項

- 這個 workflow 的重點是比較 reranker 後的知識庫檢索結果，不是比較 LLM 生成答案品質。
- 若 Dify 匯入後發現知識庫 ID 無效，需要重新指定本機 Dify 環境中的 knowledge base。
- 若模型 provider 無法使用，請先確認 self-hosted Dify 的 plugin、model provider、API endpoint 是否設定完成。
- 若要讓 Notebook 更好解析結果，建議後續將 Answer 輸出改成固定 JSON 格式。

## 適用情境

- 比較多種 reranker 在同一批問題上的檢索效果
- 建立 RAG retrieval benchmark
- 測試 self-hosted reranker 與 Dify 的整合
- 將多個 Dify reranker 專案合併成單一 API 入口

