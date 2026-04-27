markdown---
name: firmware-ng-analyzer
description: >
  使用此 SKILL 來分析韌體原始碼，並產出 NG_Reasons 分頁所需的填寫資料。
  當使用者提到以下情境時應立即啟用此 SKILL：
  - 分析某個韌體版本的測試步驟（Define.h / Func.C）
  - 填寫或產出 NG_Reasons 的表格資料
  - 分析 TEST_Step_data_keyin_func 的 ERR_DATA 意義
  - 詢問某個 _STEP 步驟 NG 時代表什麼意思
  - 說出韌體資料夾路徑並要求分析 NG 原因
  即使使用者只說「幫我分析這個版本」、「看一下 Define.h」或提供了韌體資料夾路徑，
  也要主動啟用此 SKILL。
---

# 韌體 NG_Reasons 分析 SKILL

你的目標是讀取指定韌體版本的原始碼，分析每個測試步驟發生 NG 時的錯誤資料意義，
並輸出可直接貼入 Excel NG_Reasons 分頁的表格。

## 背景知識

產線測試程式以 PIC 單晶片為基礎，測試流程由 `Test_Process_Func()` 驅動。
每個測試步驟在結束時呼叫：

```c
TEST_Step_data_keyin_func(步驟代碼, err_data_1, err_data_2);
```

這個函數將三個值存入 MES 上傳資料中：
- **TEST_STEP_DATA**：停止的步驟代碼（NG 時 = 發生問題的步驟）
- **ERR_DATA_1**：第一個錯誤輔助資料
- **ERR_DATA_2**：第二個錯誤輔助資料

**關鍵判斷**：同一個步驟會有兩種呼叫：
1. `(step, 0, 0)` — 「進入此步驟」或「通過」的記錄，**不是 NG**
2. `(step, 非0值, ...)` 且後面跟著 `Bad_code_keyin` 或 `NG_MODE` — **真正的 NG 觸發點**

---

## 分析流程

### Step 1：取得必要資訊

如果使用者尚未提供，請詢問：
- 韌體資料夾完整路徑（例如 `D:\yuanyu\321\GL0Q\A_Board\M\A_BOREAD_TESTER_V40_46K80_20260330_0904`）
- 產品名稱（對應 Excel 分頁名稱，例如 `321-GL0Q A`）
- 版本日期字串（例如 `20260330_0904`，通常可從資料夾名稱用 regex `_(\d{8}_\d{4})` 擷取）

### Step 2：讀取 Define.h，取得所有步驟定義

讀取路徑：`{韌體資料夾}/Head_file/Define.h`

搜尋所有符合此格式的行：
#define  <名稱_STEP 或 名稱_STEP_N>  <數值>

排除含有 `last_data` 或 `last_time` 的行，以及 `TEST_STEP_DATA` 這個資料槽位定義。

記錄：步驟名稱 → 步驟代碼數值

### Step 3：找出實際執行的測試流程

讀取所有 `.c` / `.C` 原始碼檔案（通常在 `Source_file/` 子目錄）。

找到 `Test_Process_Func` 函數，記錄它呼叫了哪些測試函數
（例如 `GND_CHECK_FUNC()`, `MOTOR_TEST()`, `BL_TEST_Func()` 等）。

**用途**：判斷哪些在 Define.h 有定義的步驟在本版本中實際會執行，
哪些是保留/未啟用步驟（只在 Define.h 定義但 Test_Process_Func 沒有呼叫對應函數）。

### Step 4：分析每個步驟的 NG 觸發點

在所有原始碼中搜尋 `TEST_Step_data_keyin_func`（注意：原始碼可能是 Big5 編碼的 binary 檔案，
請使用 `grep -a` 或以 binary 模式讀取後用 ASCII 解碼來搜尋）。

對每一個呼叫，判斷是否為 NG 觸發：
- 如果 err_data_1 和 err_data_2 **都是 `0`** → 通過記錄，跳過
- 如果後面緊接著 `Bad_code_keyin(...)` 或 `NG_MODE()` / `NG_MODE2()` → **真正的 NG 呼叫**，記錄下來

對每個 NG 呼叫，記錄：
- 步驟名稱
- err_data_1 的變數名稱（如 `current_flag`、`lux_judge_data[0]`）
- err_data_2 的變數名稱
- 周圍程式碼的上下文（判斷變數代表什麼意義）

### Step 5：判斷 ERR 變數意義與對照

對每個 ERR 變數，閱讀周圍的程式邏輯：

**變數意義判斷原則**：
- 若是連續量測值（電流、電壓 AD 值、Lux 值）→ ERR 意義填量測說明，**對照欄留空**
- 若是索引/代碼型（x=0 代表某通道、x=1 代表另一通道）→ ERR 意義填「索引」，**對照欄填具體對照表**

請閱讀 `references/err-var-mapping.md` 取得常見變數名稱的中文說明範本，
並根據實際程式碼的上下文決定最終說明。

**需要查看周圍程式碼的情況**：
- 變數是陣列索引 `x`：查看 `for (x = ...)` 的範圍及 `upper_limit[]` / `lower_limit[]` 或其他對應表
- 變數是 `Slave_MCU_AD_DATA[x]`：查看 `x` 的值域
- 變數是 flag 陣列：查看陣列的各個 index 代表什麼

---

## 輸出格式

以 Markdown 表格輸出，欄位順序嚴格對應 Excel 欄位：

| A 產品 | B 版本起 | C 步驟代碼 | D 步驟名稱 | E 不良描述 | F ERR1 意義 | G ERR1 對照 | H ERR2 意義 | I ERR2 對照 |
|---|---|---|---|---|---|---|---|---|
| ... | ... | ... | ... | ... | ... | ... | ... | ... |

**各欄填寫規則**：

- **A 產品**：填 Excel 分頁名稱（如 `321-GL0Q A`）
- **B 版本起**：填韌體版本字串（如 `20260330_0904`）
- **C 步驟代碼**：填 Define.h 中的數值（整數）
- **D 步驟名稱**：填 Define.h 中的常數名稱
- **E 不良描述**：用繁體中文說明什麼情況下會觸發此 NG（觸發條件，不是步驟名稱直譯）
  - 保留/未啟用步驟：填 `【保留步驟】{功能描述}，本版韌體未啟用`
  - 純步驟標記（只有 pass 記錄）：填步驟作用說明，並註明「無獨立 NG 記錄」
- **F ERR1 意義**：ERR_DATA_1 代表什麼（中文）；無記錄則留空
- **G ERR1 對照**：若是代碼型填 `值=說明;值=說明`；量測值型**留空**
- **H ERR2 意義**：ERR_DATA_2 代表什麼；無或為 0 則留空
- **I ERR2 對照**：同 G 欄邏輯

**排序**：依步驟代碼數值由小到大排列，跳過 `PASS_STEP`。

---

## 補充說明區塊

在表格之後，輸出一個「**補充說明**」區塊，說明：
1. 哪些步驟 ERR 欄位留空的原因（例如：原始碼中只有 pass 記錄）
2. 保留步驟的說明（哪些在 Define.h 定義但本版未啟用）
3. 任何需要工程師手動確認的項目（例如腳位對照表需對硬體圖查詢）

---

## 注意事項

- 原始碼通常為 Big5/CP950 編碼，使用 `grep -a` 處理 binary 檔案，或以 `open(path, 'rb').read()` 後 `decode('latin-1')` 讀取
- 同一步驟可能在多個 `.c` 檔案中出現（例如 `Func.C` 和 `EL321xxx_Func.c`）
- 同一步驟可能有多個 NG 觸發條件，合併描述於 E 欄，使用頓號或「或」連接
- 若找不到 `Test_Process_Func`，改找 `main()` 或其他明顯的測試主流程函數