---
tags:
  - gantt
  - overview
year:
show_boards:
show_archived: true
---

# Task Dashboard
```dataviewjs
// ╔═══════════════════════════════════════════════════════════════╗
// ║ 設定來源                                                       ║
// ║  • 年份(YEAR)：本檔 frontmatter 的 `year`（空 = 今年）         ║
// ║  • 樣式/分類：同資料夾的 config.md（內含一段 ```json）          ║
// ║  • 根資料夾：自動偵測為本檔所在資料夾（可被 config 覆寫）       ║
// ║ 想改設定 → 編輯 config.md，不必動這支腳本。                    ║
// ╚═══════════════════════════════════════════════════════════════╝

// ── 內建預設值（config.md 缺失或解析失敗時的後備）──────
const DEFAULTS = {
    root_folder: '',   // 空 = 自動偵測為本檔所在資料夾
    board_abbrev: {},
    priority_emoji: { high: '🔴', medium: '🟡', low: '🟢' },
    type_config: {
        bug:      { emoji: '🐛', color: '#e74c3c', label: 'Bug' },
        feature:  { emoji: '✨', color: '#27ae60', label: 'Feature' },
        release:  { emoji: '🚀', color: '#8e44ad', label: 'Release' },
        research: { emoji: '🔬', color: '#2980b9', label: 'Research' },
        review:   { emoji: '🍰', color: '#f39c12', label: 'Review' },
    },
    type_edge: 5,
};

// ── 本檔所在資料夾（自動偵測的基準）──────────────────
const HERE = dv.current().file.folder;

// ── 讀取 config.md（與本檔同資料夾）──────────────────
let CFG = DEFAULTS;
try {
    const raw  = await dv.io.load(`${HERE}/config.md`);
    const json = raw?.match(/```json\s*([\s\S]*?)```/);
    if (json) CFG = { ...DEFAULTS, ...JSON.parse(json[1]) };
} catch (e) {
    dv.paragraph('> [!warning] config.md 讀取失敗，改用內建預設值');
}

// ── 套用設定 ───────────────────────────────────────────
// 年份：本 view 的 frontmatter `year`（找不到就用今年）
const YEAR = Number(dv.current().year) || new Date().getFullYear();
// 根資料夾：frontmatter > config.md > 自動偵測（本檔資料夾）
const ROOT_FOLDER    = dv.current().root_folder || CFG.root_folder || HERE;
const BOARD_ABBREV   = CFG.board_abbrev   ?? DEFAULTS.board_abbrev;
const PRIORITY_EMOJI = CFG.priority_emoji ?? DEFAULTS.priority_emoji;
const TYPE_CONFIG    = CFG.type_config    ?? DEFAULTS.type_config;
const TYPE_EDGE      = CFG.type_edge      ?? DEFAULTS.type_edge;
// ─────────────────────────────────────────────────────

const today      = dv.date('today');
const yearStart  = dv.date(`${YEAR}-01-01`);
const yearEnd    = dv.date(`${YEAR}-12-31`);
const yearSpanMs = yearEnd.toMillis() - yearStart.toMillis() + 86400000;

const showBoards   = [].concat(dv.current().show_boards ?? []).filter(Boolean);
const showArchived = dv.current().show_archived ?? false;

const fmt = (d) => {
    if (!d) return null;
    if (typeof d === 'object' && d.toFormat) return d.toFormat("yyyy-MM-dd");
    const m = String(d).match(/(\d{4}-\d{2}-\d{2})/);
    return m ? m[1] : null;
};

// 日期 → 年內百分比（0–100）
const pct = (d) => {
    if (!d) return 0;
    const ms = d.toMillis ? d.toMillis() : dv.date(String(d)).toMillis();
    return Math.max(0, Math.min(100, (ms - yearStart.toMillis()) / yearSpanMs * 100));
};

const sanitize = (text) => text
    .replace(/\[[\s\S]+?::[\s\S]+?\]/g, '')
    .replace(/\[\[.*?(?:\|(.*?))?\]\]/g, (_, a) => a || '')
    .replace(/:/g, '：')
    .replace(/\s+/g, ' ')
    .trim();

const boardAbbrev = (name) => {
    if (BOARD_ABBREV[name]) return BOARD_ABBREV[name];
    const words = name.trim().split(/\s+/);
    return words.length > 1 ? words[0] : name.substring(0, 5);
};

const extractTitle = (text) => {
    const m = text.match(/\[title::\s*([\s\S]+?)\]/);
    return m ? m[1].trim() : null;
};

// 摘要 = 卡片內文「第一行」，且僅當它是文字時才採用。
// 只看緊接在卡片主行下方的那一行（定點規則）：
//   非縮排 / 空白 / inline 欄位 / 子勾選 → 一律不顯示，避免誤抓 BUGID、[type::] 等
const extractSummary = (lines, lineNo) => {
    const raw = lines[lineNo + 1];
    if (raw == null || !/^[\t ]/.test(raw)) return null;   // 沒有內文（只有標題）
    const trimmed = raw.trim();
    if (!trimmed) return null;                             // 第一行空白 → 不顯示
    if (/^\[[^\]]*::/.test(trimmed)) return null;          // inline 欄位 → 不顯示
    if (/^[-*+]\s*\[[ xX]\]/.test(trimmed)) return null;   // 子勾選 → 不顯示
    const text = sanitize(trimmed);
    return text ? text.substring(0, 120) : null;
};

const normalizeStatus = (heading) => {
    const s = (heading ?? '').toLowerCase().replace(/[\s_-]/g, '');
    if (s === 'inprogress')              return 'In Progress';
    if (s === 'todo')                    return 'To Do';
    if (s === 'awaitingrelease')         return 'Awaiting Release';
    if (s === 'done')                    return 'Done';
    if (s === 'archive' || s === 'archived') return 'Archive';
    return 'Backlog';
};

const STATUS_ORDER = ['In Progress', 'To Do', 'Backlog', 'Awaiting Release', 'Done'];
const byStatus = Object.fromEntries(STATUS_ORDER.map(s => [s, []]));
let taskCount = 0;
const unscheduled = [];

const boards = dv.pages(`"${ROOT_FOLDER}"`)
    .where(p => p["kanban-plugin"] === "board")
    .where(p => !showBoards.length || showBoards.includes(p.file.name))
    .sort(p => p.file.name);

// 預先讀取各看板原始行（直接用縮排判斷是否為頂層卡片，
// 因為 Dataview 對 TAB 縮排的 task.parent 解析不可靠）
const rawBoardLines = {};
for (const board of boards) {
    const raw = await dv.io.load(board.file.path);
    rawBoardLines[board.file.path] = raw ? raw.split('\n') : [];
}

for (const board of boards) {
    const boardLines = rawBoardLines[board.file.path] ?? [];
    for (const task of board.file.tasks) {
        // 跳過縮排的子任務（以 TAB 或空白開頭的行 = 卡片內容中的勾選項目）
        const rawLine = boardLines[task.line] ?? '';
        if (/^[\t ]/.test(rawLine)) continue;
        if (!task.start && !task.due) {
            if (task.completed) continue;
            const abbrev  = boardAbbrev(board.file.name);
            const status  = normalizeStatus(task.section?.subpath);
            if (status === 'Done' || status === 'Archive' || status === 'Awaiting Release') continue;
            const customTitle = extractTitle(task.text);
            const uType    = TYPE_CONFIG[(task.type ?? '').toString().toLowerCase().trim()];
            const baseText = (customTitle ? sanitize(customTitle) : sanitize(task.text))
                             .substring(0, 80) || 'Untitled';
            const taskText = `${uType?.emoji ?? ''} ${baseText}`.trim();
            unscheduled.push({ abbrev, boardName: board.file.name, status, taskText });
            continue;
        }

        const s = task.start ?? task.due;
        const e = task.due ?? (task.start ? task.start.plus({days: 7}) : task.start);
        if (s > yearEnd || e < yearStart) continue;

        let status = normalizeStatus(task.section?.subpath);
        if (status === 'Archive' && !showArchived) continue;
        if (status === 'Archive') status = 'Done';

        taskCount++;
        const abbrev = boardAbbrev(board.file.name);
        const customTitle = extractTitle(task.text);
        const taskPart    = (customTitle ? sanitize(customTitle) : sanitize(task.text))
                            .substring(0, 58 - abbrev.length) || `Task ${taskCount}`;
        const priority  = (task.priority ?? '').toLowerCase();
        const prioEmoji = PRIORITY_EMOJI[priority] ?? '';
        const prioLabel = priority ? `${prioEmoji} ${priority.charAt(0).toUpperCase() + priority.slice(1)}` : '';
        const typeKey   = (task.type ?? '').toString().toLowerCase().trim();
        const typeCfg   = TYPE_CONFIG[typeKey] ?? null;
        const typeEmoji = typeCfg?.emoji ?? '';
        const isDone    = task.completed || status === 'Done' || status === 'Awaiting Release';
        const isOverdue = !isDone && !!task.due && task.due < today;
        const isActive  = !isDone && !isOverdue && !!task.start && task.start <= today;
        const label     = `${typeEmoji}${prioEmoji}[${abbrev}] ${taskPart}`.trim();
        const hasDue    = !!task.due;
        const startDate = task.start ?? task.due;
        const endDate   = task.due ?? (task.start ? task.start.plus({days: 7}) : task.start);
        const sp        = pct(startDate);
        const ep        = Math.max(sp + 0.5, pct(endDate));
        const summary   = extractSummary(boardLines, task.line);

        byStatus[status].push({
            label,
            startStr: fmt(startDate),
            dueStr:   fmt(endDate) + (hasDue ? '' : ' (+7d)'),
            sp, ep, isDone, isOverdue, isActive,
            typeColor: typeCfg?.color ?? null,
            typeLabel: typeCfg?.label ?? null,
            prioLabel,
            summary,
            boardName: board.file.name,
            sectionHeading: task.section?.subpath ?? '',
        });
    }
}

// ── 統計列 ──────────────────────────────────────────
const inProgressCount = byStatus['In Progress'].length;
const todoCount       = byStatus['To Do'].length + byStatus['Backlog'].length;
const doneCount       = byStatus['Done'].length;
const archivedNote    = showArchived ? '　　📦 **含封存任務**' : '';
dv.paragraph(
    `> 📅 **已排程**：${taskCount} 項　　🔄 **進行中**：${inProgressCount} 項　　📋 **待辦**：${todoCount} 項　　✅ **已完成**：${doneCount} 項　　⏳ **待排程**：${unscheduled.length} 項${archivedNote}`
);

// ── 分類圖例 ─────────────────────────────────────────
const legendItems = Object.values(TYPE_CONFIG)
    .map(c => `<span style="display:inline-flex;align-items:center;gap:4px;margin-right:14px;white-space:nowrap">`
        + `<span style="width:10px;height:10px;border-radius:2px;background:${c.color};display:inline-block"></span>`
        + `${c.emoji} ${c.label}</span>`)
    .join('');
dv.container.createEl('div').innerHTML =
    `<div style="font-size:11px;color:var(--text-muted);margin:.1em 0 .5em">分類：${legendItems}</div>`;

// ── HTML Gantt 圖 ────────────────────────────────────
if (taskCount === 0) {
    dv.paragraph(`> [!tip] ${YEAR} 年無含日期的任務`);
} else {
    const MONTHS = ['1','2','3','4','5','6','7','8','9','10','11','12'];
    const monthData = MONTHS.map((m, i) => ({
        label: `${m}月`,
        p: pct(dv.date(`${YEAR}-${String(i + 1).padStart(2, '0')}-01`)),
    }));
    const todayP = pct(today);

    const css = `<style>
.dvg{font-size:12px;overflow-x:auto;margin:.4em 0}
.dvg-inner{min-width:480px}
.dvg-axis{display:flex;align-items:flex-end;padding-bottom:4px;margin-bottom:2px;
  border-bottom:1px solid var(--background-modifier-border);
  position:sticky;top:0;background:var(--background-primary);z-index:2}
.dvg-lcol{width:220px;flex-shrink:0;font-size:10px;color:var(--text-faint);padding-right:8px}
.dvg-tcol{flex:1;position:relative;height:18px}
.dvg-mname{position:absolute;bottom:2px;font-size:10px;color:var(--text-muted);
  transform:translateX(-50%);white-space:nowrap;pointer-events:none}
.dvg-sh{display:flex;align-items:center;margin:8px 0 2px}
.dvg-sh::before{content:'';width:220px;flex-shrink:0}
.dvg-sh-line{flex:1;display:flex;align-items:center;gap:6px;
  border-bottom:1px solid var(--background-modifier-border);padding-bottom:2px}
.dvg-sh-name{font-size:10.5px;font-weight:700;color:var(--text-muted);
  text-transform:uppercase;letter-spacing:.06em;white-space:nowrap}
.dvg-row{display:flex;align-items:center;min-height:22px;margin-bottom:1px;border-radius:3px}
.dvg-row:hover{background:var(--background-secondary)}
.dvg-lbl{width:220px;flex-shrink:0;padding-right:8px;overflow:hidden;
  text-overflow:ellipsis;white-space:nowrap;color:var(--text-normal);cursor:default}
.dvg-area{flex:1;position:relative;height:14px}
.dvg-ml{position:absolute;top:0;bottom:0;width:1px;
  background:var(--background-modifier-border);opacity:.5;pointer-events:none}
.dvg-tl{position:absolute;top:-5px;bottom:-5px;width:2px;
  background:var(--color-accent,#7c6ff7);opacity:.85;z-index:1;pointer-events:none}
.dvg-bar{position:absolute;top:0;height:100%;border-radius:2px;min-width:8px;cursor:pointer}
.dvg-bar:hover{filter:brightness(1.15)}
</style>`;

    let html = css + '<div class="dvg"><div class="dvg-inner">';

    // 月份軸
    html += '<div class="dvg-axis">';
    html += `<div class="dvg-lcol">${YEAR} 任務時間軸</div>`;
    html += '<div class="dvg-tcol">';
    for (const { label: ml, p: mp } of monthData) {
        html += `<span class="dvg-mname" style="left:${mp.toFixed(1)}%">${ml}</span>`;
    }
    html += `<div class="dvg-tl" style="left:${todayP.toFixed(1)}%;top:0;bottom:0"></div>`;
    html += '</div></div>';

    // 各 section
    for (const status of STATUS_ORDER) {
        const tasks = byStatus[status].sort((a, b) => a.sp - b.sp);
        if (!tasks.length) continue;

        const sectionLabel = (status === 'Done' && showArchived) ? 'Done + Archived' : status;
        html += '<div class="dvg-sh">';
        html += `<div class="dvg-sh-line"><span class="dvg-sh-name">${sectionLabel}</span></div>`;
        html += '</div>';

        for (const t of tasks) {
            const bg = t.isOverdue ? '#c0392b'
                     : t.isDone   ? '#888888'
                     : t.isActive ? '#2980b9'
                     :              '#5dade2';
            const w   = Math.max(0.5, t.ep - t.sp).toFixed(1);
            const tip = `${t.startStr} → ${t.dueStr}`;
            const typeMark  = t.typeColor ? `box-shadow:inset ${TYPE_EDGE}px 0 0 ${t.typeColor};` : '';
            const typeAttr  = t.typeLabel ? ` data-type="${t.typeLabel}"` : '';
            const prioAttr    = t.prioLabel ? ` data-prio="${t.prioLabel.replace(/"/g,'&quot;')}"` : '';
            const summaryAttr = t.summary ? ` data-summary="${t.summary.replace(/"/g,'&quot;')}"` : '';
            const boardAttr   = ` data-board="${t.boardName.replace(/"/g,'&quot;')}"`;
            const sectionAttr = t.sectionHeading ? ` data-section="${t.sectionHeading.replace(/"/g,'&quot;')}"` : '';

            html += '<div class="dvg-row">';
            html += `<div class="dvg-lbl" title="${t.label}">${t.label}</div>`;
            html += '<div class="dvg-area">';
            for (const { p: mp } of monthData) {
                html += `<div class="dvg-ml" style="left:${mp.toFixed(1)}%"></div>`;
            }
            html += `<div class="dvg-tl" style="left:${todayP.toFixed(1)}%"></div>`;
            html += `<div class="dvg-bar" style="left:${t.sp.toFixed(1)}%;width:${w}%;background:${bg};${typeMark}" data-label="${t.label.replace(/"/g,'&quot;')}" data-tip="${tip}"${typeAttr}${prioAttr}${summaryAttr}${boardAttr}${sectionAttr}></div>`;
            html += '</div></div>';
        }
    }

    html += '</div></div>';

    const el = dv.container.createEl('div');
    el.innerHTML = html;

    // ── 浮動提示卡（桌機 hover / 手機兩步點按共用）─────────
    // tt 掛在 body 並跨 Dataview refresh 重用；狀態存在 tt 節點上，
    // 避免每次 refresh 重複累積全域 listener。
    let tt = document.getElementById('dvg-tip');
    if (!tt) {
        const gs = document.createElement('style');
        gs.textContent = '#dvg-tip{position:fixed;background:var(--background-primary);'
            + 'border:1px solid var(--background-modifier-border);'
            + 'box-shadow:0 2px 12px rgba(0,0,0,.25);border-radius:6px;'
            + 'padding:6px 10px;font-size:11px;color:var(--text-normal);'
            + 'pointer-events:none;z-index:9999;display:none;line-height:1.65;'
            + 'max-width:280px;word-break:break-word}';
        document.head.appendChild(gs);
        tt = document.createElement('div');
        tt.id = 'dvg-tip';
        document.body.appendChild(tt);
    }

    // 每次 render 先歸零：清掉可能殘留的卡片與 armed 節點
    const hideTip = () => { tt.style.display = 'none'; tt._armedBar = null; };
    hideTip();

    const showTip = (bar) => {
        const summaryLine = bar.dataset.summary
            ? `<div style="color:var(--text-muted);margin-top:3px;font-style:italic">${bar.dataset.summary}</div>` : '';
        const typeLine = bar.dataset.type
            ? `<div style="color:var(--text-muted);margin-top:2px">🏷️ ${bar.dataset.type}</div>` : '';
        const prioLine = bar.dataset.prio
            ? `<div style="color:var(--text-muted);margin-top:2px">${bar.dataset.prio}</div>` : '';
        tt.innerHTML = `<div style="font-weight:600">${bar.dataset.label}</div>`
                     + summaryLine
                     + `<div style="color:var(--text-muted);margin-top:2px">${bar.dataset.tip}</div>`
                     + typeLine + prioLine;
        tt.style.display = 'block';
        const r    = bar.getBoundingClientRect();
        let   left = r.left + r.width / 2 - tt.offsetWidth / 2;
        let   top  = r.top - tt.offsetHeight - 8;
        left = Math.max(8, Math.min(left, window.innerWidth - tt.offsetWidth - 8));
        if (top < 8) top = r.bottom + 8;
        tt.style.left = left + 'px';
        tt.style.top  = top  + 'px';
    };

    // 跳轉前一定先收卡片，避免 body 層級的浮動卡片飄到新畫面上殘留
    const navigate = (bar) => {
        hideTip();
        if (!bar.dataset.board) return;
        const anchor = bar.dataset.section ? `#${bar.dataset.section}` : '';
        app.workspace.openLinkText(`${bar.dataset.board}${anchor}`, '', false);
    };

    // 桌機：滑鼠 / 觸控筆 hover 顯示、離開隱藏（觸控不走這條路）
    el.addEventListener('pointerover', (e) => {
        if (e.pointerType === 'touch') return;
        const bar = e.target.closest('.dvg-bar');
        if (bar) showTip(bar);
    });
    el.addEventListener('pointerout', (e) => {
        if (e.pointerType === 'touch') return;
        if (e.target.closest('.dvg-bar')) hideTip();
    });

    // 啟用：桌機點一下直接跳；手機第一點顯示卡片、第二點同一條才跳
    el.addEventListener('click', (e) => {
        const bar = e.target.closest('.dvg-bar');
        if (!bar) return;
        if (tt._lastPointerType === 'touch') {
            if (tt._armedBar === bar) {   // 第二次點同一條 → 跳轉
                navigate(bar);
            } else {                      // 第一次點 → 只顯示卡片，先不跳
                showTip(bar);
                tt._armedBar = bar;
            }
        } else {
            navigate(bar);               // 滑鼠 / 觸控筆：維持原本一點即跳
        }
    });

    // 全域 listener 只綁一次（首次 render 綁定，之後 refresh 重用同一個 tt）
    if (!tt._dvgGlobalBound) {
        tt._dvgGlobalBound = true;
        document.addEventListener('touchstart', () => { tt._lastPointerType = 'touch'; }, true);
        document.addEventListener('pointerdown', (e) => {
            tt._lastPointerType = e.pointerType || 'mouse';
            // 點在長條以外 → 收起卡片（手機點空白處即可關閉）
            if (!e.target.closest('.dvg-bar')) hideTip();
        }, true);
        window.addEventListener('scroll', () => hideTip(), true);
    }
}

// ── 未排程任務清單 ───────────────────────────────────
if (unscheduled.length > 0) {
    unscheduled.sort((a, b) => {
        const sd = STATUS_ORDER.indexOf(a.status) - STATUS_ORDER.indexOf(b.status);
        return sd !== 0 ? sd : a.boardName.localeCompare(b.boardName);
    });
    dv.paragraph(`---\n\n### ⏳ 未排程任務（${unscheduled.length} 項）`);
    dv.table(
        ['狀態', '看板', '任務'],
        unscheduled.map(t => [t.status, t.boardName, t.taskText])
    );
    dv.paragraph(
        '> [!tip] 加上日期即可進入時間軸\n' +
        '> `[start:: YYYY-MM-DD]` · `[due:: YYYY-MM-DD]`'
    );
}
```

# How to use
## 任務時間軸

> **切換年份**：改本檔上方 Properties 的 `year`（留空 = 今年；每個 view 一年）
> **根資料夾**：自動偵測為本檔所在資料夾，免設定；要指向別的資料夾才在 `config.md` 或 Properties 填 `root_folder`
> **樣式 / 分類 / 看板縮寫 / 優先度 emoji**：都在同資料夾的 `config.md` 調整，不必動腳本
> **篩選看板**：在上方 Properties 的 `show_boards` 填看板名稱，留空 = 顯示全部
> **日期**：`[start:: 2026-05-07]` `[due:: 2026-05-14]`
> **優先度**：`[priority:: High]` 🔴　`[priority:: Medium]` 🟡　`[priority:: Low]` 🟢
> **分類**：`[type:: bug]` 🐛　`feature` ✨　`release` 🚀　`research` 🔬　`review` 🍰
> 　（顯示為標籤 emoji + 長條左側色邊；分類清單在 `config.md` 的 `type_config` 自訂）
> **摘要**：卡片內文「第一行」會顯示在提示卡標題下方（淡色斜體）
> 　想要摘要 → 卡片第二行（標題下一行）寫一句話；不想要 → 留空白，或讓 `[type::]` 等欄位接在標題後
> **點長條**：電腦 = 滑鼠移上去看提示卡、點一下跳到看板；手機 = 先點一下看提示卡、再點同一條才跳（點空白處收起卡片）
> **紅色 bar** = 過期未完成任務
## Template
[title:: ]
[start:: ]
[due:: ]
[priority:: ]
[type:: ]
