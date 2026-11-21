# LabGuard AI — 區域教學醫院智慧醫檢平台（MVP）

## 快速開始
以 Chrome/Safari 開啟 `index.html`，不需要安裝套件、資料庫或連網。檔案可用靜態網站部署，但**公開部署時不得上傳真實病人資料**。

## 核心模組
- 戰情室：依生化、血液、微生物篩選合成數據 KPI；含 TAT 圖表、警示、去識別化 CSV 匯入與範本下載。CSV 限制 1MB／5000列及固定五欄。
- QC：Levey–Jennings、z-score、CV%、1₂s、1₃s、2₂s、4₁s，簡化 R₄s 跨兩個 level 示範；新增 QC 值與調查紀錄。
- 報告審核：六個合成案例；QC 狀態、檢體、分析儀、身分、危急值、Delta、溶血指數規則；審核結果與稽核軌跡 CSV。
- 角色：主任、組長、醫檢師、稽核員 UI 演示（非真實身分驗證）。

## 安全限制
1. 無後端、無身分驗證、無永久保存與不可竄改稽核紀錄。
2. 無 HIS/LIS/API 串接、無檢驗報告放行、無危急值通知。
3. 目前儀表板混合展示示範圖表與匯入 CSV 彙總，並未完成全流程資料轉換；所有展示數據需標示來源。
4. QC 參考值、SD、多規則適用條件與報告危急值／Delta／溶血門檻全部是**教學演示**；必須依儀器、試劑、檢驗方法及病患族群經醫檢科核定。
5. 本檔案瀏覽器中處理使用者輸入，不向外傳送；但如部署公開靜態主機，資料仍可能暴露於用戶端環境，禁止上傳 PHI。

## 正式系統資料設計
- `specimens`: specimen_id, patient_pseudonym, collection_time, receipt_time, sample_type, source_unit, priority, acceptance_state, rejection_code
- `lab_results`: result_id, specimen_id, test_code, numeric_result, unit, flag, analyzer_id, result_time, previous_result, previous_time, qc_id, validation_status
- `qc_measurements`: qc_id, analyzer_id, analyte, control_lot, reagent_lot, control_level, measured_at, result, target_mean, target_sd, applicable_rules, qc_disposition
- `reference_rules`: rule_id, analyte, method, population, unit, critical_low, critical_high, delta_type, delta_threshold, interval, effective_date, approver, version
- `audit_log`: event_id, actor_id, role, timestamp, object_id, action, before, after, reason, software_version, rule_version

重要設計：病人識別 token 需由院內管理，研究用途資料不應在用戶端自行產生重識別對照表。正式資料庫須以最小權限、加密、備援及保留期限管理。

## 核心流程
1. 受控收件／LIS 介接 → schema 與單位檢查 → 重複、遺漏及時間序檢查。
2. QC 判定（需能追溯到當批次）→ 禁止失控批次自動放行 → 儀器／檢體／身分檢查。
3. 關鍵值與 Delta rule（各檢驗／族群／方法學核定）→ 決策分流：保留待審、優先危急值工作流或候選規則通過。
4. 授權醫檢師人工複核 → 依 SOP 通報與追蹤 → 可稽核記錄 → 才由正式 LIS 層核發。
5. 每日分析 TAT、退件、規則警示、漏報／誤報、QC CAPA、模型漂移。

## 分工與權限（正式）
- 醫檢師：檢驗工作清單、QC 登錄、複核、通知狀態記錄（依授權）。
- 組長／品管：QC 設定提案、CAPA、規則驗證、班表與 TAT。
- 主任：規則／版本核准、KPI、風險承擔、上線與退版批准。
- 稽核：受限唯讀、稽核匯出，不得修改。
- IT 系統管理員：部署、備份與權限，不得因管理者權限默認取得臨床核發權。

## KPI 精確定義
- TAT P50/P90/P95：需指定 clock start/end，如「收件→技術認可」「收件→LIS核發」；按急診/門診/住院及檢驗項目分類。
- TAT 達標率 = 完成且達到指定 SLA 的檢驗件數 / 同條件已完成檢驗件數。需同時監控未完成案件，防止選擇性偏差。
- 退件率 = 不合格被拒收檢體數 / 收到的全部檢體數；與重抽率分開。
- QC 失控率 = 觸發核定拒收規則的 QC run / 全部 QC run；追蹤 CAPA 關閉時間。
- 危急值通報及確認率、首次通知時效、超時比率分開統計。
- 審核錯誤攔截：FN（嚴重異常被放過）、FP（過多攔截）、敏感度、特異度、PPV；不同項目分層分析。

## 驗證方案
- G0：SOP、危急值／Delta／QC 閾值表，版本簽核；建立風險分析（FMEA）。
- G1：回溯資料，保留時序切分，確保同一病人／檢驗批次不洩漏於訓練與測試集；人工建立 gold-standard。
- G2：靜默模式，與既有 SOP 結果盲態比對，涵蓋急診／一般、不同班別、儀器、不同控制品批號、異常與高風險個案。
- G3：逐項上線准入：危急值零未通知由現行流程管控；對任何高風險規則設專屬安全門檻與停用條件；留存變更審核與回退 SOP。
- G4：每月 KPI 與安全事件檢視；模型資料漂移、儀器與 lot 變更觸發重新驗證。

## 延伸建置
後端建議內網 PostgreSQL + FastAPI（RBAC/SSO/稽核）、ETL 與排程、單向接 LIS，中介層；不讓瀏覽器直接接觸資料庫。AI 進一步可做工作量預測、QC 偏移辨識、異常結果風險排序，須完成臨床在地驗證。

## 主要參考
- ISO 15189:2022 — https://www.iso.org/standard/15189
- CLSI AUTO10 — https://clsi.org/shop/standards/auto10/
- CLSI AUTO15 — https://clsi.org/shop/standards/auto15/
- CLSI C24 — https://clsi.org/shop/standards/c24/
- IFCC AI committee 2026 opinion — https://ifccfiles.com/2026/02/10.1515_cclm-2025-1322.pdf


## iPhone / ChatGPT preview troubleshooting (v0.2)

Some ChatGPT file preview surfaces do not run embedded JavaScript; buttons in those previewers cannot function even when the HTML itself is correct. The top runtime status bar remains yellow when scripts are blocked and turns green when scripts run. Open with a regular browser from a trusted HTTPS host (e.g. GitHub Pages) for the most reliable phone experience. This prototype does not send data to a server. Mobile touch targets and sticky tabs were enlarged in v0.2.