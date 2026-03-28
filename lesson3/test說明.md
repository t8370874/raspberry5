# test.py 程式說明

## 概述

此程式是一個 **Open WebUI Filter 插件範例**，用於在聊天 API 請求的前後進行攔截與處理。
透過 `inlet`（入口）和 `outlet`（出口）兩個鉤子函式，可以對請求與回應進行驗證、修改或分析。

- 作者：open-webui
- 版本：0.1
- 來源：https://github.com/open-webui

---

## 依賴套件

| 套件 | 用途 |
|------|------|
| `pydantic` | 資料驗證與設定管理（BaseModel、Field） |
| `typing` | 型別提示（Optional） |

---

## 類別結構

### `Filter`

主要的 Filter 類別，包含兩個內部設定類別與三個方法。

---

### `Filter.Valves`（系統層級設定）

繼承自 `BaseModel`，定義管理員可設定的全域參數。

| 欄位 | 預設值 | 說明 |
|------|--------|------|
| `priority` | `0` | Filter 的執行優先順序，數字越小越優先 |
| `max_turns` | `8` | 整個系統允許的最大對話輪數上限 |

---

### `Filter.UserValves`（使用者層級設定）

繼承自 `BaseModel`，定義每位使用者可自訂的參數。

| 欄位 | 預設值 | 說明 |
|------|--------|------|
| `max_turns` | `4` | 該使用者允許的最大對話輪數上限 |

> 注意：實際生效的 `max_turns` 會取 `UserValves.max_turns` 與 `Valves.max_turns` 兩者的最小值，確保使用者無法超過系統上限。

---

### `__init__(self)`

初始化方法，在 Filter 物件建立時執行。

- 初始化 `self.valves`，載入系統層級的 `Valves` 設定。
- 程式碼中有一段被註解掉的 `self.file_handler = True`：
  - 若啟用，可告知 WebUI 將檔案相關操作交由此 Filter 自行處理，而非使用預設邏輯。

---

### `inlet(self, body, __user__)`

**請求前置處理器（Pre-processor）**

在聊天請求送往 API 之前被呼叫，可用於驗證或修改請求內容。

**參數：**
| 參數 | 型別 | 說明 |
|------|------|------|
| `body` | `dict` | 原始請求主體，包含 `messages` 等欄位 |
| `__user__` | `Optional[dict]` | 當前使用者資訊，包含角色與個人 valves 設定 |

**處理邏輯：**
1. 印出模組名稱、請求主體、使用者資訊（用於除錯）。
2. 若使用者角色為 `"user"` 或 `"admin"`，則進行對話輪數檢查：
   - 取得 `messages` 列表。
   - 計算有效上限：`max_turns = min(使用者上限, 系統上限)`。
   - 若訊息數量超過上限，拋出 `Exception` 阻止請求繼續。
3. 回傳（可能已修改的）`body`。

**例外：**
- 當對話輪數超過限制時，拋出 `Exception`，訊息格式為：
  ```
  Conversation turn limit exceeded. Max turns: {max_turns}
  ```

---

### `outlet(self, body, __user__)`

**回應後置處理器（Post-processor）**

在 API 回傳結果後被呼叫，可用於修改回應或進行額外分析。

**參數：**
| 參數 | 型別 | 說明 |
|------|------|------|
| `body` | `dict` | API 回應主體 |
| `__user__` | `Optional[dict]` | 當前使用者資訊 |

**處理邏輯：**
1. 印出模組名稱、回應主體、使用者資訊（用於除錯）。
2. 直接回傳 `body`（目前未做額外修改，可依需求擴充）。

---

## 執行流程圖

```
使用者發送訊息
      │
      ▼
  inlet() ──── 超過對話輪數上限？ ──→ 拋出 Exception，阻止請求
      │ 否
      ▼
  呼叫 Chat API
      │
      ▼
  outlet() ──── 後處理回應
      │
      ▼
  回傳結果給使用者
```

---

## 擴充建議

- 在 `inlet` 中可加入內容過濾、關鍵字偵測等邏輯。
- 在 `outlet` 中可加入回應格式化、敏感資訊遮蔽等功能。
- 啟用 `self.file_handler = True` 可接管檔案上傳的處理流程。
