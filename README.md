# HRV_APP_V4.0 · SHIELD

以 **Streamlit** 建立的 ECG／心率變異度（HRV）分析與休克風險推論原型。應用程式從指定的 FHIR Observation 讀取 ECG 波形，利用既有 Keras 模型辨識心搏類別、產生 10 項特徵，再交由 XGBoost 模型輸出分數，並在網頁呈現波形、特徵與風險分級。

> 本專案為研究與展示用途。模型輸出及介面門檻不應直接作為臨床診斷、治療或分流依據；儲存庫未提供足以在此確認效能、校準或臨床適用範圍的驗證資料。

## 主要功能

- 輸入 Bearer token 與 Observation URL，擷取單筆 FHIR JSON。
- 將 `valueSampledData.data` 轉換為單欄 ECG CSV。
- 使用 `model_focalloss.h5` 進行心搏分類，依程式中的正常心搏篩選邏輯計算特徵。
- 使用 `xgb_model.json` 推論，顯示百分比與 LOW／MODERATE／HIGH RISK。
- 顯示 Observation JSON、可調整起始索引的 ECG 波形與 10 項特徵。
- 以 session state 保留分析結果，避免單純調整波形滑桿時重跑完整流程；token 或 URL 改變時清除舊結果。

## 分析流程

```text
FHIR Observation
  → parse_fhir_ecg_to_csv.py
  → ECG CSV（II 欄）
  → generate_HRV_10_features.py + function_all.py + Keras 模型
  → HRV／ECG 特徵 CSV
  → shock_rate.py + XGBoost 模型
  → Streamlit 視覺化
```

目前送入 XGBoost 的 10 項特徵依序為：

```text
aFdP, fFdP, ARerr, DFA.Alpha.1, Mean.rate,
Poincar..SD2, shannEn, LF.HF.ratio.LombScargle,
Grid Counting, SgridTAU
```

特徵 CSV 另包含第 11 欄 `time_min`。目前 `shock_rate.py` 以欄位位置移除第 11 欄，因此不得任意重排或增減輸入欄位。

## 專案結構

```text
.
├── app.py                       # Streamlit 入口、FHIR 讀取與視覺化
├── parse_fhir_ecg_to_csv.py      # FHIR sampled data → II 欄 CSV
├── generate_HRV_10_features.py   # 分段、模型分類及特徵產生
├── function_all.py              # ECG、RR 間期與特徵計算函式
├── shock_rate.py                # XGBoost 推論
├── models/
│   ├── model_focalloss.h5        # Keras 心搏分類模型
│   └── xgb_model.json            # XGBoost 模型
├── requirements.txt
├── runtime.txt                  # python-3.9.18
└── Dockerfile                   # Python 3.9.18、Streamlit port 10000
```

兩個模型檔目前均列在儲存庫中。這個專案的主要入口是推論介面，並非完整模型訓練與評估流程。

## 環境與安裝

建議先使用與 `runtime.txt`、Dockerfile 一致的 **Python 3.9.18** 環境。主要套件包含 TensorFlow／Keras 2.10.0、XGBoost 1.7.6、NumPy 1.24.4、pandas 1.5.3、SciPy 1.9.3、BioSPPy、pyHRV、NeuroKit2 與 HeartPy；完整版本以 `requirements.txt` 為準。

以下為依現有程式整理的操作方式，尚未在本次文件更新中完成依賴安裝、Docker 建置或端到端推論驗證。

```bash
git clone https://github.com/lenkun25369-ui/HRV_APP_V4.0.git
cd HRV_APP_V4.0
python -m venv .venv
```

啟用環境，依作業系統擇一：

```bash
# Linux / macOS / WSL
source .venv/bin/activate
```

```powershell
# Windows PowerShell
.venv\Scripts\Activate.ps1
```

安裝並啟動：

```bash
python -m pip install --upgrade pip wheel
python -m pip install -r requirements.txt
python -m pip install setuptools==75.1.0
python -m pip check
python -m streamlit run app.py
```

請從專案根目錄執行，因為模型與子程式路徑使用相對路徑。依終端顯示的本機網址開啟介面，通常為 `http://localhost:8501`。

### Docker

```bash
docker build -t hrv-app .
docker run --rm -p 127.0.0.1:10000:10000 hrv-app
```

開啟 `http://localhost:10000`。上述指令僅將服務映射至本機；公開部署前請先處理下方安全事項。

## 輸入資料與操作

1. 準備有權限存取的 **單筆 Observation URL** 與有效 token。
2. 在畫面輸入 `Token` 和 `Observation URL`；兩者皆非空白時自動開始分析。
3. 完成後查看分數、ECG 波形、10 項特徵及展開的原始 JSON。

目前解析器直接讀取 JSON 最外層的 `valueSampledData.data`，格式為空白分隔的數字字串。它不會搜尋 Bundle、component 或其他巢狀位置，也不會處理非數字的缺失／特殊代碼。

```json
{
  "resourceType": "Observation",
  "valueSampledData": {
    "data": "0.10 0.12 0.08 0.11"
  }
}
```

上面僅示意解析器需要的欄位，**不是完整 FHIR 合規範例，也不是可供 HRV 推論的完整 ECG**。

### 波形條件

- 程式固定以 **125 Hz** 計算，沒有根據 FHIR metadata 自動偵測或重取樣。
- 解析後資料一律放入 `II` 欄；請提供與模型預期一致的單通道 ECG，而非心率 bpm 或多通道交錯資料。
- 程式不套用 SampledData 的 origin、factor、period 或 dimensions；數值尺度、單位與採樣率需事先確認。
- 分段目標為 **37,500 點，即 125 Hz 下的 5 分鐘**。目前不足一段或尾段長度不足時，索引邏輯可能出錯或取到非預期資料，應先準備完整 5 分鐘片段。
- 特徵計算需有足夠可辨識心搏與有效 RR 間期；程式進一步保留對應心率 50–150 bpm 的正常 RR 間期。此為實作篩選條件，不代表所有臨床情境均適用。
- 多段資料雖可能產生多列輸出，目前網頁只顯示第一列特徵及第一筆預測，不能將其視為完整時間序列警報。

### 命令列分步操作

對已獲授權、去識別化的資料，可從專案根目錄執行：

```bash
python parse_fhir_ecg_to_csv.py observation.json ecg.csv
python generate_HRV_10_features.py ecg.csv features.csv
python shock_rate.py features.csv
```

若直接提供 ECG CSV，需有標題 `II`，每列一個數值。沒有 GUI 檔案上傳入口；CSV 路徑是命令列用法。

`shock_rate.py` 可透過環境變數 `XGB_MODEL_PATH` 指定替代模型；但網頁啟動檢查仍要求預設的 `models/xgb_model.json` 存在。替代模型必須與輸入特徵的順序、前處理與預期維度相容。

## 結果解讀

介面將第一筆模型輸出乘以 100，並依下列程式門檻顯示：

| 顯示值 | 標籤 |
|---|---|
| 小於 20% | LOW RISK |
| 20% 至小於 40% | MODERATE RISK |
| 40% 以上 | HIGH RISK |

這些是目前 UI 的顯示規則，不等於經驗證的臨床決策門檻，也不能僅由百分比顯示推定模型已完成機率校準。

波形圖雖使用「Heart Rate」標題，實際畫的是解析出的 **ECG 振幅對樣本索引**，不是逐點心率 bpm。振幅是否為 mV 需由資料來源確認。

## 已知限制與安全事項

- **TLS 驗證目前停用：** `fetch_observation()` 使用 `verify=False`。正式使用前應改為驗證伺服器憑證並配置可信任 CA，不應在未受保護的網路處理醫療資料。
- **token 保護：** 程式支援從 URL query 讀取 `token`／`obs`，但不建議把真實 token 放在網址，避免洩漏到瀏覽歷史、代理或存取紀錄。token 會以 Bearer header 送至填入的 Observation URL，僅使用可信任且已授權的端點。
- **本機資料副本：** 即使主流程使用暫存目錄，兩個腳本仍會在目前工作目錄額外寫入 `ECG_5min.csv`、`h0.csv`，不會隨暫存目錄自動刪除。請勿提交至 GitHub；使用後依機構規範管理或清除。
- **多使用者部署：** 固定檔名可能相互覆寫；目前不應假設已具備多使用者隔離、身分驗證與存取控制。介面還可展開完整 Observation JSON。
- **相依套件未完全鎖版：** `streamlit` 等套件未固定版本，程式使用 `st.experimental_get_query_params()`，新版 API 相容性需另行確認。若出現 API 或匯入錯誤，請記錄實際版本，不要直接假設升級所有套件即可修復。
- 儲存庫尚未提供端到端測試資料、完整環境鎖定檔、模型訓練資料來源與完整驗證報告。README 的存在不代表已驗證可直接部署。

## 參考與授權

介面原有的特徵參考連結：[DOI: 10.1016/j.bspc.2024.106854](https://doi.org/10.1016/j.bspc.2024.106854)。此處保留既有連結，不據此主張本程式已完整重現該研究。

目前儲存庫未附獨立 LICENSE；程式與模型的使用、修改或再散布權利，請先向維護者確認。

