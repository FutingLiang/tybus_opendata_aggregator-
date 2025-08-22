# 桃園公車開放資料整合系統 (TYCG Bus OpenData Integration System)

本專案為一個專門針對桃園市政府開放資料平台的公車資料整合工具，採用 ETL (Extract, Transform, Load) 架構設計。系統能夠自動化地從多個 API 端點抓取公車路線、業者資訊與站點資料，並進行資料清洗、轉換與整併，最終產出結構化的資料集供後續分析使用。

**主要腳本**: `c:\Users\root\Desktop\tybus_opendata_aggregator.py`

## 系統架構概覽

### 資料來源
- **路線資料源**: 桃園市公車路線清單 API (`ROUTE_BASE_URL`)
- **業者資料源**: 桃園市市區公車動態資料 API (`OP_BASE_URL`)
- **站點資料源**: PDF 格式的站點地址檔案 (透過路線資料中的 `stopAddrUrl` 取得)

### 處理流程
```
API 資料抓取 → 資料解析與清洗 → 跨資料源整合 → 結構化輸出
     ↓              ↓              ↓           ↓
分頁式抓取      JSON/XML解析    業者ID對照    DataFrame格式
進度追蹤       PDF文字提取     地址匹配      CSV/Excel輸出
錯誤重試       正規表示式      資料合併      資料庫匯入
```

## 核心技術組件

### 1. 通用分頁抓取引擎 (`fetch_pages`)
- **分頁策略**: 基於 `offset` 與 `limit` 參數的標準分頁機制
- **容錯機制**: 
  - HTTP 狀態碼檢查與例外處理
  - JSON 解析失敗時自動切換至 XML 解析模式
  - 網路逾時與重試機制
- **效能最佳化**: 
  - 可配置的請求間隔防止 API 限流
  - `tqdm` 整合提供即時進度視覺化
  - 記憶體效率的資料累積策略

### 2. PDF 文件解析模組 (`fetch_and_parse_pdf`)
- **文件處理**: 
  - 透過 `requests` 進行二進位 PDF 下載
  - 使用 `io.BytesIO` 實現記憶體內文件處理
  - `pdfplumber` 引擎進行文字提取
- **資料萃取**:
  - 正規表示式模式匹配站牌與地址資訊
  - 容錯的文字格式處理 (處理空白、換行等格式問題)
  - 結構化資料輸出 (`[{'站牌': '...', '地址': '...'}]`)

### 3. 資料整合與對照系統
- **業者資訊對照**: 建立 `OperatorID` 與 `OperatorName` 的映射表
- **資料正規化**: 統一欄位命名與資料格式
- **關聯性建立**: 路線、業者、站點間的多對多關係處理

## 環境建置與部署

### 系統需求
- **Python 版本**: 3.9+ (建議 3.11+)
- **作業系統**: Windows 10/11, macOS, Linux
- **記憶體需求**: 最少 2GB (處理大量 PDF 時建議 4GB+)

### 依賴套件清單
```txt
requests>=2.28.0      # HTTP 請求處理
pandas>=1.5.0         # 資料處理與分析
openpyxl>=3.0.0       # Excel 檔案操作
xmltodict>=0.13.0     # XML 解析
tqdm>=4.64.0          # 進度條顯示
pdfplumber>=0.7.0     # PDF 文字提取
```

### 安裝與設定
1. **環境隔離**:
   ```bash
   python -m venv tycg_bus_env
   # Windows
   .\tycg_bus_env\Scripts\activate
   # macOS/Linux
   source tycg_bus_env/bin/activate
   ```

2. **套件安裝**:
   ```bash
   pip install -r requirements.txt
   ```

3. **設定檔調整**:
   編輯 `tybus_opendata_aggregator.py` 頂部的參數區塊：
   ```python
   ROUTE_BASE_URL = "https://opendata.tycg.gov.tw/api/v1/..."
   OP_BASE_URL = "https://opendata.tycg.gov.tw/api/v1/..."
   REQUEST_SLEEP_SEC = 0.2  # 調整請求頻率
   ```

## API 函式庫參考

### `fetch_pages(base_url, limit=1000, start_offset=0, max_pages=200, show_progress=True)`
**用途**: 通用分頁資料抓取器
**參數**:
- `base_url` (str): API 端點 URL
- `limit` (int): 每頁資料筆數上限
- `start_offset` (int): 起始偏移量
- `max_pages` (int): 最大抓取頁數
- `show_progress` (bool): 是否顯示進度條

**回傳**: `List[Dict]` - 所有頁面資料的合併列表

### `fetch_and_parse_pdf(url, route_name)`
**用途**: PDF 站點資料解析
**參數**:
- `url` (str): PDF 檔案的 URL
- `route_name` (str): 路線名稱 (用於錯誤追蹤)

**回傳**: `List[Dict]` - 解析出的站牌地址資料

### `build_operator_map(op_data)`
**用途**: 建立業者 ID 對照表
**參數**:
- `op_data` (List[Dict]): 業者原始資料

**回傳**: `Dict[str, str]` - 業者 ID 與名稱的對照字典

## 效能調校與監控

### 效能參數
- `REQUEST_SLEEP_SEC`: 控制 API 請求頻率，避免觸發限流機制
- `max_pages`: 限制最大抓取頁數，防止無限迴圈
- PDF 處理: 大型 PDF 檔案可能需要較長處理時間

### 錯誤處理策略
- **網路錯誤**: 自動重試機制 (建議實作指數退避)
- **資料格式錯誤**: JSON/XML 雙重解析策略
- **PDF 解析失敗**: 跳過問題檔案並記錄錯誤

## 擴展性設計

### 資料庫整合
```python
# SQLAlchemy 整合範例
from sqlalchemy import create_engine
engine = create_engine('postgresql://user:pass@localhost/tycg_bus')
df_final.to_sql('bus_routes', engine, if_exists='replace', index=False)
```

### 排程自動化
- **Windows**: 工作排程器設定
- **Linux/macOS**: crontab 設定
- **雲端**: GitHub Actions 或 AWS Lambda

### 即時監控
- **日誌系統**: Python `logging` 模組整合
- **效能監控**: 記錄處理時間與資料量
- **異常告警**: 整合 Slack 或 Email 通知

## 故障排除指南

### 常見問題
1. **HTTP 429 錯誤**: 增加 `REQUEST_SLEEP_SEC` 至 0.5-1.0 秒
2. **PDF 解析失敗**: 檢查 PDF URL 有效性與網路連線
3. **記憶體不足**: 減少 `limit` 參數或分批處理
4. **編碼問題**: 確保輸出檔案使用 `utf-8-sig` 編碼

### 除錯技巧
- 啟用詳細日誌輸出
- 使用小範圍資料進行測試
- 檢查 API 回應格式變更

## 使用範例

### 基本執行
```bash
python tybus_opendata_aggregator.py
```

### 自訂輸出
```python
# 在腳本末端加入
df_routes.to_csv("tycg_routes.csv", index=False, encoding="utf-8-sig")
df_operators.to_excel("tycg_operators.xlsx", index=False)
```

### 資料庫匯入
```python
import sqlite3
conn = sqlite3.connect('tycg_bus.db')
df_final.to_sql('routes', conn, if_exists='replace', index=False)
conn.close()
```

---

此系統設計為可擴展的資料整合平台，可根據需求調整資料來源、處理邏輯與輸出格式。如需客製化功能或技術支援，請參考程式碼註解或聯繫開發團隊。
