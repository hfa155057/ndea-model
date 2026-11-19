# SORM-NDEA｜上市公司營運與市場效率分析

以 Python 和 Gurobi 實作兩階段網路資料包絡分析（Network DEA），評估各決策單位（DMU）的營運與市場效率。第一階段透過投入產生中間產出，第二階段以中間產出連接最終產出。Notebook 提供含 ESG 與不含 ESG 的模型，可用於比較納入 ESG 指標前後的效率結果。

## 執行環境

- 建議使用 Google Colab 開啟 `NDEA_SORM_(1).ipynb`。
- 需要 `pandas`、`openpyxl`、`gurobipy`，以及能支援模型規模的 Gurobi 授權。

在 Colab 新增儲存格，執行：

```python
%pip install -q pandas openpyxl gurobipy
```

## Excel 資料格式

主函式每次讀取一張工作表。每列代表一個 DMU，欄名需與下表一致。請確認 DMU 識別碼唯一，模型使用的數值欄位沒有空值。Notebook 直接讀取已拆分的欄位，請先依研究的資料定義完成正負值前處理。

| 欄位 | 用途 |
| --- | --- |
| `DMU` | 決策單位識別碼，例如公司代號 |
| `I1`、`I2` | 第一階段投入，對應資產總計與營業成本 |
| `G1`、`G2` | 跨階段中間指標的正負拆分欄位 |
| `G3` | ESG 指標，僅含 ESG 版本需要 |
| `O1`、`O2` | 最終產出指標的正負拆分欄位 |
| `O3`、`O4` | 其他最終產出指標 |

## 執行步驟

1. 在 Colab 上傳自己的 Excel 資料檔，確認要分析的工作表名稱。附件只有 Notebook，資料檔需另行提供。
2. 執行定義 `run_dea`（含 ESG）或 `run_dea1`（不含 ESG）的儲存格。Notebook 含早期試算及硬編碼路徑，請依所需模型逐格執行。
3. 在新儲存格執行下列程式。將檔名與工作表名稱改為實際值。

```python
from google.colab import files

excel_file = next(iter(files.upload()))

# 含 ESG
result = run_dea(excel_file, "esg")

# 不含 ESG 時，改用下列呼叫
# result = run_dea1(excel_file, "traditional")

output_file = "DEA_result.xlsx"
result.to_excel(output_file, index=False)
files.download(output_file)
```

若要比較 ESG 納入前後的結果，請使用相同 DMU 和可比的資料設定，並依 DMU 識別碼合併兩組結果。

## 模型與輸出

模型對每個 DMU 分別求解，兩階段的參考權重各自加總為 1，採可變規模報酬（VRS）設定。目標函數為最小化 `a - b`。回傳的 DataFrame 欄名以工作表名稱為前綴。

| 輸出欄位 | Notebook 的計算方式 |
| --- | --- |
| DMU | 來源工作表的決策單位識別碼 |
| 第一階段效率 | 投入收縮係數 `a`，限制為 `0 ≤ a ≤ 1` |
| 第二階段效率 | 原始產出擴張係數 `b`，限制為 `b ≥ 1`，以 1 為基準 |
| 總效率 | 依現有程式計算 `a × b` |

若要重現論文表格，請核對論文採用的第二階段效率轉換及總效率定義，再解讀輸出。函式直接回傳 DataFrame，需另外呼叫 `to_excel()` 才會產生結果檔。

## Notebook 注意事項

- `run_dea2` 是另一個試算版本。它讀入 `O1`，但產出迴圈目前只遍歷 `O3`、`O4`，且 ESG 約束已註解。原有批次儲存格呼叫此函式，將 `exclude_dmus` 中的值當作工作表名稱，實際未執行 DMU 排除。
- 求解若未達 `OPTIMAL`，現有程式仍可能在讀取 `a.X`、`b.X` 時中斷。遇到這種情況，先確認求解狀態、輸入欄位及 Gurobi 授權。
