# 專案開發規則

## 版本管理規則（強制）

**每次修改任何 `.py` 檔案後，必須同步更新該檔案開頭的 `__version__`。**

格式：`西元年_月日_時分`，例如：

```python
__version__ = "2026_0413_1702"
```

**重要：版本號必須記錄當下的真實時間（時:分），不可估算或四捨五入。**
更新前請先用 `Get-Date -Format "yyyy_MMdd_HHmm"` 取得正確時間再填入。

## 軟體資訊填寫規則

當使用者貼上一條 hex 路徑（例如
`D:\yuanyu\735\TMC1\9U\M\Main_BOARD_TESTER_20260714_1412\Main_BOARD_TESTER_20260714_1412_0x245b_unlock.hex`）
並附上修改內容時，將軟體資訊填入 Software Change Log。

> **單一主檔原則（2026-07 導入自動化後）：** 全流程只維護一份主檔
> `Software Change Log.xlsx`（固定名、無時間戳）。**不再每次另存 `_AI`/時間戳副本**；
> 改為「先備份到 `備份/`、再就地更新主檔」，以根治重複與副本堆積。

**流程：**

1. **備份主檔**：先把 `Software Change Log.xlsx` 複製一份時間戳備份到 `備份/` 子資料夾
   （可直接呼叫 `pipeline_common.backup_excel()`），再就地編輯主檔本身。
   - 若尚未有 `Software Change Log.xlsx`（遷移前），以資料夾內最新、檔名不含
     `_副本_`/`_AI` 的 `Software Change Log_*.xlsx` 為準（Phase F 會正式建立主檔）。
2. 從 hex 路徑拆出各欄位，寫進對應分頁：
   - A 機種名稱：分頁名稱
   - B 修改內容：使用者提供的文字
   - C 軟體名稱：hex 檔名（含 `.hex`）
   - D 修改日期：取檔名中的 `YYYYMMDD`
   - E 備註：空白（除非使用者指定）
   - F 路徑：hex 所在「資料夾」路徑（不含檔名）— **一定要填**，這是 scanner/ng_prepare
     跳過全碟掃描的關鍵。
3. **分頁判斷（強制）：**
   - 分頁已存在 → 填入該分頁「下一個空白區塊」（本檔為 4 列一筆的合併儲存格區塊），保留既有資料。
   - 分頁不存在（新案子）→ **先停下來詢問使用者分頁名稱**，不可自行猜測命名；
     預設複製一個結構相同的現有分頁當範本建立新分頁，再填入第一個空區塊
     （使用者若指定範本分頁則依指定）。
4. **登記交辦清單（強制）**：在專案根目錄 `pending_tasks.txt` **append 一行**
   `分頁名<TAB>韌體資料夾絕對路徑`（F 欄那個資料夾路徑）。
   後續 `ng_prepare.py` / `scanner.py` 只處理清單內項目，成功發布後由 scanner 自動清空。
5. 完成後保留所有其他分頁不動，就地存回主檔，並告知使用者已備份與已登記交辦。

## 自動化流程與指令（2026-07）

- **NG 準備（可排程、無視窗）**：`python ng_prepare.py --main --no-input`
  依交辦清單找新步驟、插入 NG_Reasons 空白列；接著由 Claude 讀 `Func.C` 分析填入
  不良描述/ERR，**給使用者摘要人工確認後**才發布。
- **產出並發布**：`python scanner.py --main --no-input`（或雙擊 `update_and_push.bat`）
  → 重建 `firmware_db.json` → git push。成功後自動清空 `pending_tasks.txt`。
- **NG 去重維護**：`python ng_prepare.py --dedup-report`（檢視）/ `--dedup-apply`（收斂，
  先自動備份、只刪空白或內容完全相同的重複列，不動人工內容）。
- **排除不再維護的分頁**：在 `exclude_sheets.txt` 每行放一個分頁名或關鍵字。
- 這些本機檔（`pending_tasks.txt`/`exclude_sheets.txt`/`備份/`）皆已 gitignore，不上 GitHub。
