本專案用於批次抓取並整併「桃園市公車」相關開放資料，包含：

依資源 API 分頁抓取（JSON 為主，必要時回退 XML）
建立業者對照表（從市區公車動態資料集）
下載節流與進度列顯示
後續可擴充為匯出 CSV/Excel、資料清洗與彙整
特性
分頁抓取通用函式：fetch_pages()（支援進度列）
容錯處理：非 JSON 時自動嘗試 XML 解析
下載節流：避免過快被限流
依你提供的兩個資料源運作
資料來源
路線清單（含 stopAddrUrl）
ROUTE_BASE_URL = https://opendata.tycg.gov.tw/api/v1/dataset.api_access?rid=06073bd9-7ce3-4107-899a-ae63d29e0267&format=json
桃園市市區公車動態（建立「業者對照表」）
OP_BASE_URL = https://opendata.tycg.gov.tw/api/v1/dataset.api_access?rid=4303aab9-ec73-49a1-8420-f477a45df87f&format=json
環境需求
Python 3.9+
套件：
requests
pandas
openpyxl
xmltodict
tqdm
pdfplumber（如需解析 PDF 附檔）
安裝
在 Google Colab
原始檔首行已有：
!pip -q install requests pdfplumber pandas openpyxl xmltodict tqdm
在本機（Windows）
建議先建立虛擬環境，然後：
pip install requests pdfplumber pandas openpyxl xmltodict tqdm
主要參數
ROUTE_BASE_URL：路線清單 API
OP_BASE_URL：市區公車動態 API（用於建立「業者對照表」）
REQUEST_SLEEP_SEC：每次請求間隔秒數（預設 0.2）
使用方式
Colab：
在 Colab 開啟此檔案，執行所有儲存格。
依進度列觀察抓取狀況。
視需求在末段程式補上輸出 CSV/Excel 的邏輯。
本機（Windows）：
安裝依賴套件。
將此檔另存為 main.py（或你偏好的檔名）。
執行：
python main.py
輸出
本檔側重資料抓取與整併邏輯。若你需要固定輸出檔（CSV/Excel），請在資料彙整完成後新增：
python
import pandas as pd
# 假設 df_routes, df_operators 為你整理好的 DataFrame
df_routes.to_csv("routes.csv", index=False, encoding="utf-8-sig")
df_operators.to_excel("operators.xlsx", index=False)
如需我直接幫你補上輸出欄位與檔名，告訴我要輸出的表格與欄位清單。
常見問題
無法連線或 429：調高 REQUEST_SLEEP_SEC，例如 0.5 或 1.0。
亂碼：CSV 用 utf-8-sig，Excel 請用 openpyxl 匯出。
來源格式不一：fetch_pages() 已在 JSON 失敗時嘗試 XML；仍失敗時請貼錯誤訊息。
後續可擴充
自動合併「路線—站位—業者」完整管線（ETL）
例外與重試策略（指數回退、最大重試次數）
加入快取與增量更新
定期排程（Windows Task Scheduler / GitHub Actions）
若你要，我可以：

建立專案骨架於 C:\Users\root\CascadeProjects\TYCG-Bus-OpenData-Aggregator\
加入 requirements.txt、README.md、基本輸出程式碼與資料夾結構。
