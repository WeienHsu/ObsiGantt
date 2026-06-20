# 📊 Gantt Board Template

一套用 **Obsidian Kanban + Dataview** 打造的年度任務甘特圖。
看板負責記任務，`Gantt View.md` 把所有看板的任務畫成一張時間軸。

---

## 1. 必要外掛（缺一不可）

到 **設定 → 社群外掛** 安裝並啟用：

| 外掛 | 用途 |
|------|------|
| **Dataview** | 跑甘特圖腳本（必裝） |
| **Kanban** | 編輯看板卡片（必裝） |

> ⚠️ **最重要、最常忘的一步**：
> 進 **設定 → Dataview → 打開「Enable JavaScript Queries」**。
> 沒開的話，Gantt View 會是**一片空白、也不報錯**，很難察覺。

---

## 2. 安裝（複製整包）

1. 把整個 `Gantt Board Template` 資料夾**複製**到你的 vault 任意位置。
2. **改名**成你的專案名稱（例如 `My Project`）—— 怎麼改都行，
   `Gantt View.md` 會**自動偵測**自己所在的資料夾，不必改任何設定。
3. 打開 `Gantt View.md`，切到閱讀模式，應該就看到甘特圖了。

---

## 3. 日常使用

### 開新看板
複製 `Template.md` → 改檔名成看板名稱（例如 `Backend`、`設計`）。
每個看板就是甘特圖裡的一條「來源」。

### 加任務
在看板卡片裡用 inline 欄位（這套系統靠這些欄位運作）：

```
- [ ] [title:: 任務名稱]
	[start:: 2026-05-07]
	[due:: 2026-05-14]
	[priority:: high]      ← high / medium / low
	[type:: feature]       ← bug / feature / release / research / review
```

- 只有填了 `start` 或 `due` 的任務才會進時間軸；沒日期的會列在下方「未排程」。
- 卡片**標題下一行**寫一句話 → 會變成滑鼠 tooltip 的摘要。

### 設定年份
打開 `Gantt View.md` 的 Properties，改 `year`（留空 = 今年）。
想同時看多年 → 複製 `Gantt View.md` 成 `Gantt View-2027.md`，各自設 `year`。

---

## 4. 自訂外觀（改 `config.md`）

所有顏色 / emoji / 分類 / 看板縮寫都在 `config.md` 的那段 `json` 裡，**不必動腳本**：

- `type_config`：分類的 emoji、顏色、圖例名稱（可自由增減）
- `priority_emoji`：優先度 emoji
- `board_abbrev`：看板顯示縮寫（留空 = 自動取名）
- `type_edge`：分類色邊寬度（px）

改完存檔 → 回 Gantt View 重新整理即可（有時需切離再切回讓它重讀）。

---

## 5. 檔案結構

```
你的專案夾/
├── Gantt View.md   ← 甘特圖引擎（自動偵測資料夾，通常不用改）
├── config.md       ← 共用外觀設定
├── Template.md     ← 空白看板，複製它開新看板
├── README.md       ← 本檔
└── Notes/          ← 卡片連結的細節筆記
```

---

## 6. 疑難排解

| 症狀 | 原因 / 解法 |
|------|------|
| 甘特圖一片空白 | 多半是沒開 Dataview 的 **Enable JavaScript Queries**（見第 1 節） |
| 顯示「config.md 讀取失敗」 | `config.md` 的 json 有語法錯（少逗號、引號不對）；會自動退回內建預設值 |
| 任務沒出現在時間軸 | 該任務沒填 `start` / `due`，或日期不在 `year` 那一年內 |
| 改了 config 沒變化 | 切離 Gantt View 再切回，或重新整理，讓 Dataview 重讀 |
| 新看板沒被掃到 | 確認該檔 frontmatter 有 `kanban-plugin: board`（用 Template 複製就會有） |
| 改名後，看板新增的筆記跑到錯的資料夾 | Kanban 外掛把筆記存放路徑寫死在看板裡（`new-note-folder`），改資料夾名後不會自動跟著變。請打開該看板 → 右上「⋮」→ **Open board settings** → 把 **Note folder** 改成你改名後的 `Notes` 資料夾 |
