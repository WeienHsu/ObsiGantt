---
tags:
  - config
---

# Gantt 設定檔

> 這是 Gantt View 共用的設定。改完存檔後，回到 Gantt View 重新整理即可生效。
>
> - `YEAR` 不在這裡 —— 它放在 `Gantt View.md` 自己的 frontmatter（留空 = 今年）。
> - `root_folder` 預設「自動偵測」為 Gantt View 所在的資料夾，**保持空字串即可**。
>   只有當你想把 Gantt View 放到別的資料夾、指向另一個任務夾時才填。

```json
{
  "root_folder": "",

  "board_abbrev": {
    "Template": "ChangeProjectName"
    },

  "priority_emoji": {
    "high": "🔴",
    "medium": "🟡",
    "low": "🟢"
  },

  "type_config": {
    "bug":      { "emoji": "🐛", "color": "#e74c3c", "label": "Bug" },
    "feature":  { "emoji": "✨", "color": "#27ae60", "label": "Feature" },
    "release":  { "emoji": "🚀", "color": "#8e44ad", "label": "Release" },
    "research": { "emoji": "🔬", "color": "#2980b9", "label": "Research" },
    "review":   { "emoji": "🍰", "color": "#f39c12", "label": "Review" }
  },

  "type_edge": 5
}
```

## 怎麼用

- **新增分類**：在 `type_config` 加一筆，例如
  `"improve": { "emoji": "⚙️", "color": "#16a085", "label": "Improve" }`，
  任務寫 `[type:: improve]` 就會套用。
- **看板縮寫**：`board_abbrev` 留空 = 由腳本自動取名（取看板檔名第一個詞或前 5 字）。
  想自訂就填，例如 `{ "My Long Board Name": "MLB" }`。
- **改顏色**：改 `color` 的色碼即可。
- **改優先度 emoji**：改 `priority_emoji`。
