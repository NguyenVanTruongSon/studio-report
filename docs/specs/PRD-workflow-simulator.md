# PRD — OpenProject Workflow Simulator (Orbit Workflow Builder)

**Version:** 2.0  
**Date:** 2026-05-22  
**Status:** Approved — synced với design `Workflow Admin.html`

---

## 1. Overview

### 1.1 Mục tiêu

Xây dựng tool nội bộ **Orbit Workflow Builder** — mô phỏng hệ thống Type → Role → Workflow Transition của OpenProject. Cho phép team thiết kế và kiểm tra bộ quy trình workflow trực quan trước khi triển khai thực tế.

### 1.2 Vấn đề cần giải quyết

- OpenProject không có giao diện trực quan để xem toàn bộ workflow matrix theo Type/Role cùng lúc.
- Cấu hình thủ công trên OP Admin tốn thời gian, dễ sai, không có diagram tự động.
- Không có công cụ thống kê số lượng transition theo Type/Role để review trước khi áp dụng.

### 1.3 Người dùng mục tiêu

| Người dùng | Nhu cầu |
|---|---|
| Project Manager | Thiết kế workflow phù hợp quy trình dự án |
| System Admin | Kiểm tra tính nhất quán trước khi config lên OP |
| Team Lead | Review và phê duyệt bộ rule transition |

---

## 2. Kiến trúc màn hình

### 2.1 App Shell

```
┌─────────────────────────────────────────────────────┐
│ SIDEBAR (232px)  │  MAIN AREA                       │
│ ─────────────    │  ─────────────────────────────   │
│ Logo "Orbit"     │  TOPBAR: breadcrumb + actions     │
│ Search settings  │  ─────────────────────────────   │
│ Nav: Project     │  PAGE HEADER:                     │
│  └ Settings      │    Chips: module code, workspace  │
│    └ Workflow ●  │    H1: "Workflow Builder"          │
│ HR               │    Subtitle + action buttons       │
│ Sale             │    TABS: Editor / Statistics /    │
│ CRM...           │          Catalogs                  │
│ ─────────────    │  ─────────────────────────────   │
│ User avatar      │  TAB BODY                         │
└──────────────────┴──────────────────────────────────┘
```

### 2.2 Ba Tab chính

| Tab | Nội dung |
|---|---|
| **Workflow editor** | 3-column layout: Left (selector/cards) + Center (chart+matrix) + Right (subtotals/summary) |
| **Statistics** | Heatmap type×role với drill-down theo condition |
| **Catalogs** | Side-rail + CRUD panel cho Types, Statuses, Roles |

---

## 3. Seed Data (chính xác từ design)

### 3.1 Types (7)

| id | Name | Code | Color |
|---|---|---|---|
| task | Task | TSK | oklch(0.58 0.12 260) |
| summary | Task summary | SUM | oklch(0.55 0.02 60) |
| ms | Milestone | MIL | oklch(0.58 0.12 35) |
| epic | Epic | EPC | oklch(0.58 0.12 320) |
| feat | Feature | FTR | oklch(0.58 0.12 195) |
| story | User Story | STO | oklch(0.58 0.12 150) |
| bug | Bug | BUG | oklch(0.58 0.14 25) |

### 3.2 Statuses (11)

| id | Name | Group | initial | closed |
|---|---|---|---|---|
| new | New | open | ✓ | — |
| spec | Specified | open | — | — |
| ins | In Specification | in_progress | — | — |
| conf | Confirmed | open | — | — |
| tbs | To be scheduled | open | — | — |
| sched | Scheduled | in_progress | — | — |
| prog | In progression | in_progress | — | — |
| wait | Waiting for Approve | in_progress | — | — |
| hold | On Hold | on_hold | — | — |
| closed | Closed | closed | — | ✓ |
| rejected | Rejected | closed | — | ✓ |

> Status có 2 flag đặc biệt: `initial` (INIT tag, màu blue border) và `closed` (END tag, màu green border).

### 3.3 Roles (8)

| id | Name | Short | Color |
|---|---|---|---|
| member | Member | MB | var(--ink-3) |
| leader | Leader | LE | oklch(0.58 0.12 35) |
| pm | Project Manager | PM | oklch(0.58 0.12 260) |
| po | Project Owner | PO | oklch(0.58 0.12 320) |
| gd | Game Designer | GD | oklch(0.58 0.12 195) |
| artist | Artist | AT | oklch(0.58 0.13 340) |
| dev | Developer | DV | oklch(0.55 0.13 220) |
| qa | QA Tester | QA | oklch(0.58 0.13 130) |

### 3.4 Conditions (3)

| id | Name | Hint | Ký hiệu |
|---|---|---|---|
| default | Default | applies to anyone with this role | DE / DEF |
| author | Role is Author | only when this role created the item | AUTH |
| assignee | Role is Assignee | only when this role is currently assigned | ASG |

---

## 4. Data Model

```typescript
// ─── Catalog entities ───────────────────────────────────────
interface WorkflowType {
  id: string
  name: string
  code: string            // 3-char, e.g. "TSK"
  color: string           // oklch(...)
  tint: string            // lighter oklch for background
  count: number           // number of items of this type (display only)
  description?: string
}

interface WorkflowStatus {
  id: string
  name: string
  color: string
  group: "open" | "in_progress" | "on_hold" | "closed"
  initial: boolean        // INIT tag in diagram & sidebar
  closed: boolean         // END tag in diagram & sidebar
  isDefault: boolean      // default status when creating new item
}

interface WorkflowRole {
  id: string
  name: string
  short: string           // 2-char abbreviation, e.g. "LE"
  color: string
  tint: string
}

type ConditionId = "default" | "author" | "assignee"

interface Condition {
  id: ConditionId
  name: string
  hint: string
}

// ─── Transition storage ─────────────────────────────────────
// Key: `${fromStatusId}→${toStatusId}`  (literal → character U+2192)
type TransitionKey = string

// Main transition map:
// trans[typeId][roleId][conditionId] = Set<TransitionKey>
type TransitionMap = Record<string, Record<string, Record<ConditionId, Set<TransitionKey>>>>

// ─── Per-type configuration ──────────────────────────────────
// participating[typeId] = Set<roleId>   — which roles participate in this type
type ParticipatingMap = Record<string, Set<string>>

// typeStatuses[typeId] = Set<statusId>  — which statuses are in this type's lifecycle
type TypeStatusesMap = Record<string, Set<string>>

// ─── localStorage persistence ────────────────────────────────
// Key: "orbit.workflow.v1"
// Sets are serialized as arrays before JSON.stringify
interface PersistedState {
  types: WorkflowType[]
  statuses: WorkflowStatus[]
  roles: WorkflowRole[]
  typeSel: string
  roleSel: string
  condSel: ConditionId
  showMode: "union" | "condition"
  trans: Record<string, Record<string, Record<string, string[]>>>
  participating: Record<string, string[]>
  typeStatuses: Record<string, string[]>
}
```

---

## 5. Chi tiết tính năng

---

### Module A — Workflow Editor Tab (3-column layout)

```
LEFT (260px, sticky)        CENTER (flex-grow)           RIGHT (300px, sticky)
────────────────────        ─────────────────────────    ──────────────────────
SelectorCard                PanelHead: "Workflow chart"  SubtotalsCard
  Work-item type list         [roles/N shown]              Condition × Type×Role
  (click to select)                                        DE / AUTH / ASG counts
                            FlowDiagram (primary)          [click row to edit]
ParticipatingRolesCard        dagre auto-layout            Total transitions
  Roles in {Type}             Role-colored arrows
  [✓] checkbox per role       INIT/END/Orphan nodes       RoleSummary
  click name = edit role      Union ↔ Condition toggle     Role rollup × Type
                              Edge count                   D / A / G columns
TypeStatusesCard                                           [click to select role]
  Statuses in {Type}        PanelHead: "Edit transitions"
  [✓] checkbox per status     Role badge + Condition chip  ConditionStackingHelp
  INIT / END badges           [Suggest linear] [Clear]     "How conditions stack"
                                                           DE → AUTH → ASG
                            TransitionMatrix
                              rows = from status
                              cols = to status
                              diagonal = disabled (—)
                              active = ✓ (project color)
                              empty = □ checkbox
```

#### A.1 SelectorCard — Work-item type

- List tất cả Types, click để chọn Type đang làm việc
- Active item: left border màu project, background tint
- Hiển thị: color dot + name + code + count

#### A.2 ParticipatingRolesCard — Roles in {Type}

- Header: "Roles in {Type}" + nút "All / None" toggle
- Mỗi role: checkbox (màu role khi checked) + badge circle + tên + trạng thái
- Trạng thái: "editing" (khi đang edit) | "click to edit" | "not participating"
- Số lượng transition tổng của role trong type này (tất cả conditions)
- Click checkbox: toggle tham gia
- Click tên: chọn role để edit
- Footer: "Configures which roles can transition this type."

#### A.3 TypeStatusesCard — Statuses in {Type}

- Header: "Statuses in {Type}" + "X/Y" counter + "All / None" toggle
- Mỗi status: checkbox + color dot + name + INIT/END badge
- Click: toggle status vào/ra lifecycle của type này
- Footer: "Builds the lifecycle for this type. Excluded statuses won't appear in the chart or matrix."

#### A.4 FlowDiagram — Chart View

**Được render bằng dagre auto-layout + custom SVG:**

- Header:
  - Type color dot + name + "· workflow chart"
  - Role buttons (mỗi role 1 button, click = set editing role)
  - Separator + mode toggle: **"Union of conditions"** / **"Editing condition"**
  - Edge count

- SVG nodes (mỗi status = 1 node):
  - Rect 160×48px, rx=6
  - Circle màu status (r=5) ở bên trái
  - Tên status (cắt bớt nếu > 19 ký tự)
  - Tag **INIT** (góc trên phải, màu blue) nếu `initial=true`
  - Tag **END** (góc trên phải, màu green) nếu `closed=true`
  - Text nhỏ góc dưới phải: `{incoming}↓ · {outgoing}↑`
  - **Orphan node** (không có edge nào): background `var(--bg-2)`, text mờ

- SVG edges (mỗi allowed transition = 1 edge):
  - Dagre tính toán waypoints
  - Mỗi edge có thể được vẽ bởi nhiều roles → offset polyline perpendicular
  - Mỗi role vẽ 1 đường, màu = `role.color`
  - **Editing role**: strokeWidth=2.2, opacity=1; roles khác: opacity=0.4
  - Badge label ở giữa edge: rect + text hiển thị `role.short`
  - Arrow marker: filled triangle, màu role
  - Smooth bezier path

- Footer legend: Initial / Closed / Orphan / ↓in ↑out + "dagre auto-layout"

- **Mode "Union"**: hiển thị tất cả conditions merged vào 1 diagram
- **Mode "Editing condition"**: chỉ hiển thị condition đang edit

#### A.5 PanelHead component

- Title (có thể là JSX) + Subtitle + Right slot (buttons)
- Dùng cho cả "Workflow chart" và "Edit transitions"

#### A.6 TransitionMatrix — Ma trận Transition

- Table với sticky corner header + sticky row headers
- Corner cell: "To status →" / "From status ↓" (diagonal text)
- Column headers: vertical text (writingMode), color dot + tên
- Row headers: color dot + tên
- Cells:
  - **Diagonal (same status)**: hatch pattern, "—", cursor not-allowed
  - **Active (allowed)**: background tint, icon check trong box màu project
  - **Empty**: small checkbox border
- Click cell: toggle transition on/off → update chart real-time
- Chỉ hiển thị statuses đang trong typeStatusList của type hiện tại

#### A.7 Quick Actions trên TransitionMatrix

| Action | Behavior |
|---|---|
| **Suggest linear** | Bật tất cả transition giữa các status liền kề theo thứ tự danh sách |
| **Clear** | Tắt tất cả transition trong condition hiện tại |

#### A.8 SubtotalsCard — Right panel (condition picker)

- Header: "Condition · {Type} × {Role}" + "pick to edit"
- 3 rows (Default / Role is Author / Role is Assignee):
  - Icon (users/edit/user) + name + hint text + count
  - Click: đổi condition đang edit
  - Active: left border màu project, background tint
- Footer: "Total transitions" + số lớn

#### A.9 RoleSummary — Right panel (role rollup)

- Header: "Role rollup · {Type}" + "D / A / G" label
- Mỗi role: badge circle + name + 3 Mini chips (Default/Author/Assignee counts) + tổng
- Click row: chọn role để edit

#### A.10 ConditionStackingHelp — Right panel (info box)

- Title: "How conditions stack"
- Text: giải thích cơ chế Default → Author → Assignee stacking
- "A transition is allowed if *any* matching set permits it."

---

### Module B — Statistics Tab

#### B.1 Heatmap Type × Role

- Table: rows = Types, columns = Roles + Total column
- Header: button "Expand all / Collapse all" + "Type \ Role" label
- Mỗi role column: badge circle + tên
- Mỗi ô: tổng transitions của (type, role) qua tất cả conditions
- **Heatmap color**: `oklch(light 0.02..0.06 35)` — càng nhiều transition càng đậm màu
- Empty cell: "—"

#### B.2 Drill-down per condition

- Click row type → expand → hiện 3 sub-rows (Default / Author / Assignee)
- Sub-row: icon + name + "DEF"/"AUTH"/"ASG" + counts per role + row total
- Heatmap scale khác (max/2) cho sub-rows

#### B.3 Totals row

- "Role total" row ở cuối
- Role columns: background `var(--ink)`, text `var(--bg)`
- Grand total ô: background project color, text white, font size 17

---

### Module C — Catalogs Tab

#### Layout

```
LEFT (260px, sticky)     RIGHT (flex-grow)
────────────────────     ─────────────────────────────────
Catalogs side rail         Header: title + desc + Filter btn
 ● Types (7)               [TypeManager / StatusManager /
   Statuses (11)            RoleManager content]
   Roles (8)
```

#### C.1 TypeManager

- List mỗi type: color square + name + code + count + Edit/Delete buttons
- AddRow: input + "Add" button, Enter để thêm
- Auto-generate id từ tên (lowercase, replace special chars với `-`)
- Mặc định color khi thêm mới: `oklch(0.58 0.12 60)`

#### C.2 StatusManager

- List mỗi status: color circle + name + DEFAULT badge (nếu isDefault) + Group/Initial/Closed chips + Edit/Delete
- Chips: tone tương ứng group (info/warn/neutral/ok)
- AddRow: tạo status mới với group="open", initial=false, closed=false

#### C.3 RoleManager

- List mỗi role: avatar circle (tint bg, color border, short text) + name + id + Edit/Delete
- Avatar: 26×26px, border-radius 50%
- AddRow: auto-generate `short` từ initials của tên

#### C.4 AddRow component (dùng chung)

- Input field + "Add" button
- onKeyDown Enter → thêm
- Background `var(--bg)`, border top

---

### Module D — Page Header & Actions

- **H1**: "Workflow Builder"
- **Chips**: `P.SET.WF · Workflow`, `Workspace A`, `{N} types · {N} statuses · {N} roles`
- **Auto-save indicator**: green dot + "Auto-saved locally"
- **Action buttons** (top right):
  - `Reset` — xóa localStorage và reload (có confirm dialog)
  - `Duplicate set` — (UI stub, v2)
  - `Export JSON` — download `orbit-workflow.json` với toàn bộ state
  - `Saved` (primary) — indicator

---

### Module E — Sidebar Navigation

```
Logo "O" + "Orbit" + "Admin · v2.4"
Search: "Search settings… ⌘K"
Nav:
  [P] Project ▾
    Projects / My Tasks / Kanban Board / Templates
    Settings ▾
      Workflow  ← active (left border)
      Types / Statuses / Roles & permissions / Custom fields
  [H] HR
  [S] Sale
  [C] CRM
  [K] Knowledge Hub
  [W] Workspace Connect
Footer: avatar (LK) + name + "Admin · Workspace A" + bell icon
```

---

### Module F — Topbar

- Breadcrumb: "Project → Settings → Workflow"
- Right: module code `P.SET.WF` + divider + bell + search icons

---

## 6. Cơ chế Condition Stacking

```
Khi một user cố transition một work package:
  1. Kiểm tra set "Default" của role user → nếu cho phép → PASS
  2. Nếu user là Author → kiểm tra thêm set "Author" → nếu cho phép → PASS
  3. Nếu user là Assignee → kiểm tra thêm set "Assignee" → nếu cho phép → PASS
  4. Nếu không set nào match → DENY
```

---

## 7. Transition Seed Data (mẫu từ design)

```
Task — Leader — default:   new→spec, spec→ins, ins→conf, conf→tbs, tbs→sched, sched→prog, prog→wait, wait→closed, prog→closed
Task — Leader — author:    wait→prog, spec→ins, closed→prog
Task — Leader — assignee:  sched→prog, prog→wait, wait→closed
Task — PM — default:       new→spec, spec→conf, conf→sched, sched→prog, prog→closed
Task — PM — author:        closed→prog, prog→sched
Task — PO — default:       new→spec, conf→sched, wait→closed, prog→closed
Task — Member — assignee:  sched→prog, prog→wait
Task — Dev — assignee:     sched→prog, prog→wait, prog→hold
Task — Artist — assignee:  sched→prog, prog→wait
Epic — PM — default:       new→ins, ins→spec, spec→conf, conf→prog, prog→closed
Epic — PO — default:       new→ins, ins→spec
Milestone — PO — default:  new→sched, sched→prog, prog→closed
Milestone — PM — default:  new→sched, sched→prog
Feature — Leader — default: new→spec, spec→conf, conf→sched, sched→prog, prog→closed
Feature — GD — default:    new→ins, ins→spec, spec→conf
Feature — GD — assignee:   sched→prog, prog→wait, wait→closed
Feature — Dev — assignee:  sched→prog, prog→wait, prog→hold, hold→prog, wait→closed
Feature — QA — default:    wait→closed, wait→prog
User Story — GD — default: new→ins, ins→spec, spec→conf
User Story — Leader — default: conf→sched, sched→prog, prog→closed
User Story — Member — assignee: sched→prog, prog→wait, wait→closed
Bug — QA — default:        new→conf, conf→sched, sched→prog, prog→wait, wait→closed, wait→prog, new→rejected
Bug — Dev — assignee:      sched→prog, prog→wait, prog→hold, hold→prog
Bug — Leader — default:    new→conf, conf→sched, sched→prog, prog→closed, wait→closed, wait→rejected
```

---

## 8. Tech Stack

| Layer | Công nghệ |
|---|---|
| Framework | Next.js 15 (App Router) |
| Language | TypeScript |
| UI | Tailwind CSS + CSS custom properties |
| State | Zustand (với localStorage middleware) |
| Diagram | dagre (graph layout) + custom SVG render |
| Export | Blob/URL.createObjectURL (JSON) |
| Fonts | JetBrains Mono (mono), system-ui (sans) |

---

## 9. CSS Design Tokens (từ design)

```css
/* Color palette — oklch */
--bg:            oklch(97% 0.01 80);
--bg-2:          oklch(94% 0.01 80);
--paper:         oklch(100% 0 0);
--ink:           oklch(25% 0.02 80);
--ink-2:         oklch(45% 0.02 80);
--ink-3:         oklch(62% 0.01 80);
--rule:          oklch(88% 0.01 80);
--rule-2:        oklch(82% 0.01 80);

/* Module accent — Project (blue-ish) */
--m-project:     oklch(0.58 0.12 260);
--m-project-tint: oklch(0.96 0.03 260);

/* Font stacks */
--mono: "JetBrains Mono", ui-monospace, monospace;
--sans: system-ui, -apple-system, sans-serif;
```

---

## 10. File Structure (implement)

```
src/
├── app/
│   ├── globals.css               ← CSS tokens + base
│   └── workflow/
│       └── page.tsx              ← Route: /workflow
├── components/
│   └── workflow/
│       ├── WorkflowApp.tsx        ← Root: state + 3 tabs
│       ├── EditorTab.tsx          ← 3-column layout
│       ├── CatalogTab.tsx         ← CRUD tab
│       ├── StatisticsTab.tsx      ← Heatmap
│       ├── FlowDiagram.tsx        ← dagre SVG
│       ├── TransitionMatrix.tsx   ← Checkbox matrix
│       ├── SubtotalsCard.tsx      ← Condition picker
│       ├── RoleSummary.tsx        ← Role rollup
│       ├── ParticipatingRolesCard.tsx
│       ├── TypeStatusesCard.tsx
│       ├── SelectorCard.tsx       ← Generic list picker
│       ├── PanelHead.tsx          ← Section header
│       ├── Sidebar.tsx
│       ├── Topbar.tsx
│       ├── icons.tsx              ← Icon component
│       └── managers/
│           ├── TypeManager.tsx
│           ├── StatusManager.tsx
│           └── RoleManager.tsx
└── lib/
    └── workflow/
        ├── types.ts               ← TypeScript interfaces
        ├── seed.ts                ← Seed data + seedTransitions()
        └── store.ts               ← Zustand store + localStorage
```

---

## 11. Acceptance Criteria

| # | Tiêu chí |
|---|---|
| AC-01 | Hiển thị 7 Types, 11 Statuses, 8 Roles mặc định |
| AC-02 | Chuyển tab Editor / Statistics / Catalogs hoạt động |
| AC-03 | Chọn Type → diagram và matrix cập nhật đúng statuses/roles |
| AC-04 | Tick/untick role participation → diagram cập nhật real-time |
| AC-05 | Tick/untick status trong type → matrix và diagram cập nhật |
| AC-06 | Tick cell trong matrix → edge xuất hiện trong diagram ngay |
| AC-07 | "Suggest linear" điền sequential transitions theo thứ tự status list |
| AC-08 | "Clear" xóa tất cả transitions của condition hiện tại |
| AC-09 | Mode "Union" hiển thị tất cả conditions; "Editing condition" chỉ hiện condition đang chọn |
| AC-10 | Orphan nodes (không có edge) hiện background mờ |
| AC-11 | Status có initial=true hiện tag INIT, closed=true hiện tag END |
| AC-12 | SubtotalsCard: click condition → chuyển editing condition |
| AC-13 | RoleSummary hiện D/A/G breakdown đúng |
| AC-14 | Statistics heatmap hiện đúng số, click expand → drill-down 3 conditions |
| AC-15 | Totals row trong Statistics hiện đúng tổng per role và grand total |
| AC-16 | CRUD: thêm/xóa Type/Status/Role trong Catalogs tab |
| AC-17 | Export JSON download file `orbit-workflow.json` |
| AC-18 | Reset xóa localStorage và reload về seed data |
| AC-19 | Reload trang → toàn bộ state persist từ localStorage |
| AC-20 | Build TypeScript không có lỗi (`npm run build` pass) |

---

*Tài liệu này được sync từ design file `Workflow Admin.html` (workflow-app.jsx + workflow-editor.jsx + workflow-shell.jsx).*
