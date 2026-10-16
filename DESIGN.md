# credit-card-hub 架構設計（Muse 排程版）

研究日期：2026-10-16。原始藍圖來自使用者提供的 Gemini 對話計畫書，本版把 GitHub Actions＋Gemini API 換成 Muse 排程＋browser 巡檢。

## 架構總覽

```
[1] 前端 (GitHub Pages 靜態站)
  - 情境試算器 / 額度進度條 / 記帳 / 卡片與銀行等級維護
  - 寫回：GitHub REST API PUT /repos/{owner}/{repo}/contents/data/user_state.json
    （使用者自己的 fine-grained PAT，只存瀏覽器 localStorage）

[2] 資料（同一 repo，讀寫分離）
  - data/user_state.json      ← 只有前端寫（卡片、銀行等級、記帳）
  - data/benefits_db.json     ← 只有排程寫（權益知識庫）

[3] 排程引擎（Muse cron，取代 GitHub Actions）
  - 主排程 cc-hub-daily-sync：每天 09:00（Asia/Taipei）
    1. git pull 讀最新 data/user_state.json（enabled 卡片＋bank_tiers）
    2. browser 開各銀行官方公告頁，核對 8 張卡當季權益
    3. 比對 benefits_db.json，有變動才更新＋寫 changelog
    4. 跑網站產生器 → git commit & push → Pages 自動重部署
  - 守衛排程 cc-hub-sync-guard：每天 10:00
    檢查主排程狀態檔（checked_at 是否為今天）；異常時代查＋回報，正常時安靜
    （沿用 tokyo-flight-watch-guard 模式）
  - 一鍵觸發：不需要 workflow_dispatch；使用者在對話說一聲就手動跑一次
```

## 為什麼不用 Gemini API

排程跑的是 agent（就是我），直接讀 browser 回來的頁面內容寫 JSON，不需要外部 LLM key。
省掉 GEMINI_API_KEY，少一個要保管的秘密，也少一個計費點。

## 每日巡檢覆蓋率（2026-10-16 實測結論，台新部分當日複測更新）

- 可自動巡檢：永豐（DAWHO、幣倍卡，靜態頁）、中信 LINE Pay minisite（靜態頁）、星展（AOV、eco，需 JS 渲染，中等成本）、台新（見下方監控組合）
- 成本高：國泰 CUBE 產品權益頁是 Angular SPA，解析成本高；但國泰活動 promo 頁多為靜態，可巡

### 台新 Richart 監控組合（2026-10-16 複測：taishinbank.com.tw 並未封鎖自動化瀏覽器，先前那次阻擋是暫時性連線問題）

- 每週（官方）：https://mkp.taishinbank.com.tw/TsCms/marketing/expose/WM_20260708095300794/index.html（權益總覽頁，異動會打 NEW 標籤）＋ https://www.taishinbank.com.tw/TSB/personal/credit/intro/overview/cg047/card001/（活動連結中樞，連結增減即為異動）
- 每週（第三方交叉）：https://www.money101.com.tw/blog/台新richart卡 ＋ https://www.cardu.com.tw/message/detail.php?58136、https://www.cardu.com.tw/mpay/detail.php?32368（兩家 2026-10-14 剛更新過，頁面有日期欄位可判斷新舊）
- 每日（新活動發現）：新聞關鍵字「台新 Richart」監控（台新發新活動當日即有媒體報導：10/6 旅遊優惠、10/8 愛出國專案、10/15 加碼消息皆當日見報）
- 變動確認：官方活動辦法 XML 用純文字讀取（https://www.taishinbank.com.tw/TSB/personal/credit/intro/overview/cg047/card001/.elements/TSBankCreditCardFeatures-00016.xml）核對條文，再以 Money101＋cardu 兩家比對
- 每月：https://re-news.tw/creditcard/326915 比對指定通路名錄增刪
- 不適合的來源：Facebook 粉專（需登入）、ShopBack（沒有 Richart 權益頁）、搜尋引擎索引（官方頁爬取間隔以週/月計，不能作每日監控依據，僅適合發現新活動 URL）
- 2026-10-16 追查：多家引用台新官網的第三方頁面（Money101、beurlife）皆寫 7+1 權益適用至 2027/12/31，與先前條文 2026/7/1–2027/3/31 不同，官網亦出現「好評延長」連結，看來是延長了；最終以官方條文為準，benefits_db 初始資料先標註待官方條文確認

## 權益資料基準（2026-10-16 查得，seed 用）

完整逐卡 8 項欄位見 workspace/research_notes/tw-credit-card-q4-benefits-20261016-0303/report.md。
重點摘要：

- 中信 LINE Pay（JCB 晶緻／VISA 鼎極共用同一計畫）：2026/7/1~12/31，國內外一般 1% LINE POINTS；
  國外實體 2.8%（1%+1.8%，限面對面，排除網路/條碼/第三方支付），皆無上限，免登錄。
  次級來源稱另有限時登錄加碼至 5%（每歸戶每季上限 450 點，每週二 3000 名），官方頁未找到，待確認。
- 國泰 CUBE：2026/1/1~12/31；一般 0.3%；玩數位/樂饗購/趣旅行 L1 2%／L2 3%／L3 3.3%；
  每月指定消費上限＝消費日當日信用額度＋50 萬；每日限切換一次；小樹點效期 3 年。
  秋日遊快閃 10/1~11/30：日本指定實體 3.5%、機場免稅 4%、IP 專賣店/人氣愛店/餐廳預約平台 5%、日系名店最高 8%。
  固定回饋 8/3~12/31 可變更一次：一般 1.2%／海外實體 2.5% 至 12/31。
- 台新 Richart：2026/7/1~2027/3/31；L2：Chill 刷 3.3~10%、Pay 著刷（台新 Pay 系 3.8%、LINE Pay/全盈+Pay 2.3%）、
  其餘五刷 3.3%、假日刷 2%；L1 多數最高 1.3%；一般 0.3%、保費 1.3% 免任務；
  L2 需台新/Richart 帳戶自動扣繳；新戶核卡 60 天先享 L2；台新 Point 效期 2 年。
- 永豐 DAWHO：2026/7/1~12/31；國內 1%／國外 2%（限外幣）無上限；
  任務（DAWHO 帳戶自扣＋電子/行動帳單）加碼：大戶 Plus +4%（上限 1000/月帳單週期）、大戶 +2.5%（上限 400）。
  使用者為大戶（非 Plus）：適用 +2.5%，上限 400/月帳單週期。
- 永豐幣倍卡：2026/7/1~12/31；基本國內 1%／國外 2% 無上限；精選通路 +4%（上限 800/帳單週期，封頂 20000），
  海外合計最高 6%；海外實體加碼不限日幣、不限幣別（官方定義：交易地區非台灣且非新台幣之實體商店）；
  9/1~12/31 全球狂歡季需登錄：單月海外滿 3 萬回 500、滿 5 萬回 1000。
  注意：原始藍圖寫「海外實體限刷日幣」是錯的，官方不限幣別。
- 星展 AOV：存戶升級（指定星展帳戶自動轉帳）：國內/外一般 1% 無上限；
  生活玩家精選通路最高 10%（加碼每月上限 500 點）；
  日韓泰/新美歐實體最高 5%（加碼每月上限 500 點，須商店國別為指定地區、非台幣、帳單列示國外交易手續費）；
  星耀理財 LINE Pay +10%（2026/7/16~12/31，需登錄，月資產 30 萬＋每月一筆換匯/轉帳，每月最高 NT$500）。
  使用者存款未滿 30 萬：星耀加碼不適用。曆月結算。
- 星展 eco：2026/1/1~12/31；國內外一般 1% 無上限；日韓泰/新美歐實體最高 5%
  （每月 15000 以內滿額 5%，加碼約 600 點/月）；星願集刷 9/1~12/31 需登錄
  （每筆滿 888 集滿 5 筆筆筆最高 2%，上限 500/月）；1 點＝NT$1。
- 海外交易手續費：六家銀行信用卡皆 1.5%（台新、永豐幣倍卡為官方明載；其餘為銀行標準公式，中高信心）。

## 使用者待辦（上線前）

1. 在 GitHub 建立 credit-card-hub repo（公開，免費 Pages）
2. 把現有 fine-grained PAT 的 Repository access 加選 credit-card-hub（Contents：Read and write）；同一個 PAT 就好，VM 上的 ~/.git-credentials 不用動
3. 開啟 Pages（Settings → Pages → Deploy from a branch → main / root）；或等我建好後用 API 開（PAT 需 Pages: write）

## 待確認事項

- 中信「海外 5% 限時加碼」官方活動頁找不到，只有媒體報導；上線前我再追一次，找不到就先不列入 benefits_db
- 國泰 CUBE「固定回饋切換後不可切回」只有媒體說法，官方只寫期間內可變更一次；以官方為準
- 星展 AOV 注意事項公式（歐洲 ×5%）與標題（最高 5% 含原 1%）表述有出入；以銀行最終解釋為準，benefits_db 內並陳
