# 项目总览与详细映射（合并文档）

下面是把我们当前对话里所有设计/规则/实现细节 **按文件→功能→接口→数据流** 的完整、可操作化汇总。把这份文档保存为 `PROJECT_OVERVIEW.md`，切换工作台后按这里的清单继续工作即可。

---

# 目录

1. 快速摘要（当前阶段）
2. 项目文件结构（按文件列出并说明）
3. 每个功能由哪些文件 / 函数/变量 实现（逐项详细说明）
4. 数据库 schema（表 + 字段 + 说明）
5. 后端 API 规范（端点、方法、请求体、示例响应）
6. 前端接口与组件契约（api.js、各组件 state/props/关键方法）
7. 业务规则与约束（前端显示 / 后端最终判断）
8. 本地运行与测试命令（快速上手）
9. Debug & 常见问题检查清单
10. 你需要提供的文件 / 下次新对话需要粘贴的内容清单
11. 下一步优先任务（短期计划）

---

# 1. 快速摘要（当前阶段）

* Console / Stats / Logs 三个页面的 UI 已实现并联调到本地 mock/部分后端接口（大多数交互存在，但仍需后端最终验收）。
* Settings 页面已完成基础 Modal 框架（工作时间、黑白名单、休息日、清零策略），正在重写：要求在 Modal 内完成所有编辑、保存失败时不关闭并允许重试；保存是否生效由后端决定。
* 后端：FastAPI + monitor 线程 + SQLite 基本跑通（监控、log 写入、黑白名单匹配等已实现部分）。
* 目标：把 Settings 的完整交互 + 后端 `/settings/*` 接口补全并做联调测试，确保所有页面在离线/打包场景下均能正确加载（pywebview / Electron）。

---

# 2. 项目文件结构（当前、详尽）

```
D:\APPS\AT\1.1\
├─ backend\
│  ├─ main.py                # FastAPI app, 静态挂载(frontend/dist), 启动 monitor 线程
│  ├─ monitor.py             # 获取前台应用，黑/白名单匹配，写入日志 (get_foreground_app, monitor_foreground, start_monitor_thread)
│  ├─ db_utils.py            # SQLite 初始化与操作 (init_db, log_usage, get_blacklist, get_whitelist, get_blocked_apps_count, etc)
│  ├─ api_status.py          # 独立路由模块（/status、/logs、/stats、settings 相关占位）
│  ├─ time_tracker.db        # SQLite 文件（运行时生成）
│  └─ test.py                # 启动脚本 / 测试脚本（可同时启动后端 & 前端 dev / 模拟）
│
├─ frontend\
│  ├─ package.json
│  ├─ vite.config.js
│  ├─ dist\                   # 生产构建产物 (index.html + assets/)
│  └─ src\
│     ├─ main.jsx
│     ├─ App.jsx              # 路由 / page switch（Sidebar 控制 activePage）
│     ├─ api.js               # 前端 fetch 封装（startTask/stopTask/getTaskStatus/getLogs/getStats, 新增 settings endpoints）
│     ├─ index.css
│     └─ components\
│        ├─ Sidebar.jsx
│        ├─ Console.jsx       # 控制台 UI（大色块、四个横向模块、刻度尺）
│        ├─ Stats.jsx         # 统计页（卡片、应用列表、饼/柱/折线图）
│        ├─ Logs.jsx          # 日志页（拉 /logs，过滤，隐藏滚动条）
│        └─ Settings.jsx      # 设置页（Modal: WorkTimeModal, AppManagerModal, RestDaysModal, ResetPolicyModal）
│
└─ start.bat                  # 可选：pythonw backend/main.py / or test runner
```

---

# 3. 每个功能由哪些文件 / 函数实现（详尽映射）

> 下面按功能列出：**实现文件 → 关键函数 / 关键 state 或 DB 字段 → 行为说明 / 交互契约**。

---

## A. 前台应用监控（监控线程）

**文件**：`backend/monitor.py`
**关键函数/变量**：

* `get_foreground_app()`

  * Windows: 使用 `win32gui.GetForegroundWindow()` + `win32process.GetWindowThreadProcessId()` + `psutil.Process(pid)` 获取 `exe_name`（进程名）和 `title`（窗口标题）。
  * 返回：`(exe_name, title)`，若失败返回 `("Unknown","Unknown")`。
* `monitor_foreground()`

  * 循环：每秒检查一次前台窗口。
  * 逻辑：

    * 如果 `exe_name/title` 与上一次不同：计算上一段持续时长并 `log_usage(last_exe, last_title, duration)`。
    * 黑名单检查（匹配 `exe_name` 或 `title` 的通配/精确匹配规则）：立即尝试终止进程（`psutil.Process(...).kill()`），并写入一个事件日志：`log_usage(exe_name, title, 0)`（或专门写 error/kill 日志）。
    * 白名单逻辑：当白名单存在时，只有白名单内的应用计为“奖励休息时间”；若白名单为空则所有非黑名单应用计入“工作时长”。
  * 全局状态字典（示例）：

    ```py
    monitor_data = {
      "current_app": title,
      "current_start": start_datetime,
      "blocked_count": N,
      "work_time_elapsed": seconds,
      "rest_time_remaining": seconds
    }
    ```
* `start_monitor_thread()`

  * 用于在 `main.py` 中用 `threading.Thread(..., daemon=True).start()` 启动监控线程。

**注意**：黑名单匹配支持通配符（前端/后端存储 exe\_name 或 window pattern，后端在匹配时用 `fnmatch` 或正则）。`monitor_foreground` 要及时写入 log（避免重复写入 - 只有切换时写）。

---

## B. 日志（写入与读取）

**文件**：`backend/db_utils.py`（写入） & `backend/api_status.py`（读取 /logs 路由）
**关键函数**：

* `init_db()` — 建表（参见第 4 节 schema）
* `log_usage(app_name: str, start_time: datetime, duration: int)` — 插入 `app_usage` 表（注意：本函数签名需匹配 monitor 调用）
* `get_logs(limit)` (可在 `api_status.py` 实现) — SELECT 从 `app_usage` 或 `events` 表返回最近 N 条日志。
  **输出格式（建议 /logs）**：

```json
[
  {"time":"2025-09-18T09:00:01","type":"info","exe":"monitor.exe","message":"监控启动成功"},
  {"time":"2025-09-18T10:12:03","type":"error","exe":"monitor.exe","message":"无法写入数据库"}
]
```

前端 `Logs.jsx` 会：`fetch("http://127.0.0.1:8000/logs")` → JSON → map 到页面行。

---

## C. 统计（/stats）

**文件**：`backend/api_status.py` → `/stats` 路由；前端 `frontend/src/components/Stats.jsx`
**后端职责**：

* 返回以秒为单位的 app 使用时间映射：`{ "Chrome.exe": 3600, "VSCode.exe": 1200 }` 或更丰富结构（数组）。
* 也可支持 `?period=day|week|month` 参数。
  **前端职责（Stats.jsx）**：
* `fetchStats()`：调用 `api.js` 的 `getStats()`（`/stats?period=day`）
* 处理数据：

  * 把 <15s 的 app 合并为 `Other`（前端做聚合，如果后端没有）
  * `processedApps`：每项 `{ exe, window, seconds, included }`
  * `chartData` / `barData`：基于 `processedApps.filter(included)` 生成
* 控件交互：

  * Toggle include → 更改 `apps` 状态（前端），并即时影响饼图、柱图、进度条、百分比显示
  * 如果 `included=false` → 进度条灰色，百分比显示 `"-"`

---

## D. 控制台（Console）

**文件**：`frontend/src/components/Console.jsx`
**关键 state/方法**：

* `workPeriods`（数组），显示最多 3 个（当前、下一个、下下一个）；若结束，顶部滚动替换并保留渐隐效果。
* `countdown`（秒） → 页面样式倒计时（每秒递减，用于视觉效果）
* `restMinutes` / `blockedCount` / `isWorking` → 来自后端 `/status` 或本地 mock
* `renderTimeRuler()` → 画 24 小时刻度（每 10 分钟小刻度、30 分中刻度、60 分大刻度），并在当前分钟计算 `left` 百分比显示红线
* 大色块渐变动画：工作与休息分别使用不同的主色系；动画关键帧在 `<style>` 动态插入（组件 mount 时只插入一次）

**数据来源**：

* 推荐通过 `api.js.getTaskStatus()` 或 `api.js.getConsole()`（后端 `/task_status`）拉取 `monitor_data` 并渲染。

---

## E. 设置（Settings）

**文件**：`frontend/src/components/Settings.jsx`（当前正在重写） + 支撑后端路由 `/settings/*`
**主要 modals / 组件**：

* `WorkTimeModal`：可编辑时间段（`<input type="time">`）， 添加/删除/保存。前端检查：`isAfterLastWorkPeriod()`（仅 UI 提示，后端最终校验）。
* `AppManagerModal`（黑名单/白名单）：Tab 切换，列为 `exe` 与 `window`（**注意**：程序名称和路径**只能写入一种**，不要同时写两者），添加/删除/保存；删除仅允许在最后工作段结束（前端提示）。
* `RestDaysModal`：一个小日历格，支持选择未来某些日期，支持快速按钮：全选周五/周六/周日。只能在每周结束（后端校验）允许修改下一周/之后的休息日。
* `ResetPolicyModal`：单选清零策略（下个工作时间、每天4:00、每周一4:00、每月初4:00）。

**保存行为**：

* 保存点击后：前端 `POST` 到对应 `/settings/*` 接口。
* 如果后端 `200 OK` 并返回 accepted data → 前端关闭 modal 并更新本地显示。
* 否则（非 200 / 后端返回错误）：modal **不关闭**，底部显示红色错误信息 `saveError`，用户可以再次点击 保存（支持多次手动重试）。

**UI/UX 规则**（你明确要求）：

* 所有设置行为“前端只做显示/交互，实际是否允许变更由后端判断”。
* 程序路径/窗口标题二选一，保存 payload 仅包含 **exe OR window**（不要同时传两者作为单条记录）。
* 黑名单可随时添加；移除仅在最后一个工作时段结束后才可（前端提示，后端最终执行）。

---

# 4. 数据库 schema（建议最终版）

> sqlite 文件：`backend/time_tracker.db`（或 `time_tracker.db`）

```sql
-- 应用使用日志（主表）
CREATE TABLE IF NOT EXISTS app_usage (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  start_time TEXT,       -- ISO8601 字符串
  exe_name TEXT,         -- 进程文件名，如 "notepad.exe"
  app_name TEXT,         -- 窗口标题 / 人类可读名
  duration INTEGER       -- 秒数
);

-- 黑名单应用
CREATE TABLE IF NOT EXISTS blacklist_apps (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  exe_name TEXT,         -- 可为空，表示使用 app_name 匹配
  app_name TEXT          -- 可为空，表示使用 exe_name 匹配
);

-- 白名单应用
CREATE TABLE IF NOT EXISTS whitelist_apps (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  exe_name TEXT,
  app_name TEXT
);

-- 休息日列表
CREATE TABLE IF NOT EXISTS rest_days (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  date TEXT              -- 'YYYY-MM-DD'
);

-- 可选：settings 表（持久化 resetPolicy、rewardTime 等）
CREATE TABLE IF NOT EXISTS settings (
  key TEXT PRIMARY KEY,
  value TEXT
);
```

> 注意：`app_usage` 的 `exe_name` 与 `app_name` 允许只填其中一项（遵循你要求：**不要同时写入“名称+路径”，要单独写入其中一个**）。当 monitor 只知道窗口标题时写 `app_name`；当知道进程名写 `exe_name`。

---

# 5. 后端 API 规范（建议/占位，前端按此调用）

> 基本假设主机 `http://127.0.0.1:8000`

### 状态与操作

* `GET /task_status`
  返回：

  ```json
  { "status": "running"|"stopped",
    "app": "Notepad",
    "start_time": "2025-09-18T09:00:00",
    "work_time_elapsed": 3600,
    "rest_time_remaining": 1200,
    "blocked_count": 2
  }
  ```

* `GET /logs`
  返回数组：

  ```json
  [
    {"time":"2025-09-18T09:00:01","type":"info","exe":"monitor.exe","message":"监控启动成功"},
    {"time":"2025-09-18T10:12:03","type":"error","exe":"monitor.exe","message":"数据库锁"}
  ]
  ```

* `GET /stats?period=day`
  简单版（map）：

  ```json
  { "Chrome.exe": 3600, "VSCode.exe": 1200 }
  ```

  或更丰富版（推荐）：

  ```json
  {
    "apps": [{"exe":"Chrome.exe","window":"Chrome - tabs","seconds":3600}, ...],
    "hours": [{ "hour":"09:00","work":60,"active":50 }, ...]
  }
  ```

### Settings

* `GET /settings`
  返回以下结构（建议）：

  ```json
  {
    "workPeriods": [{"start":"09:00","end":"12:00"}, {"start":"14:00","end":"17:00"}],
    "blacklist": [{"exe":"game.exe","window":""}, ...],
    "whitelist": [{"exe":"code.exe","window":""}],
    "restDays": ["2025-09-20", ...],
    "resetPolicy":"next_work"
  }
  ```

* `POST /settings/worktimes`
  请求 body:

  ```json
  { "workPeriods": [{"start":"09:00","end":"12:00"}] }
  ```

  成功响应应返回 accepted state 或 error message：

  ```json
  { "ok": true, "workPeriods":[...] }  // 或 { "ok": false, "error": "reason" }
  ```

* `POST /settings/blacklist`、`POST /settings/whitelist`
  body:

  ```json
  { "blacklist":[{"exe":"game.exe","window":""}, ...] }
  ```

* `POST /settings/rest-days`
  body:

  ```json
  { "restDays": ["2025-09-20","2025-09-21"] }
  ```

* `POST /settings/reset-policy`
  body:

  ```json
  { "resetPolicy": "daily_04" }
  ```

> **后端必须**校验所有设置操作的合法性（例如是否允许当前时间进行删除、是否生效），并返回明确 `ok:false` 与 `error` 字段以便前端显示错误提示。

---

# 6. 前端接口与组件契约（详细）

## A. `frontend/src/api.js`

当前主要函数（示例）：

```js
const API_BASE = "http://127.0.0.1:8000";

export async function startTask(appName) { return fetch(`${API_BASE}/start_task/${encodeURIComponent(appName)}`).then(r=>r.json()); }
export async function stopTask() { return fetch(`${API_BASE}/stop_task`).then(r=>r.json()); }
export async function getTaskStatus() { return fetch(`${API_BASE}/task_status`).then(r=>r.json()); }
export async function getLogs() { return fetch(`${API_BASE}/logs`).then(r=>r.json()); }
export async function getStats(period="day"){ return fetch(`${API_BASE}/stats?period=${period}`).then(r=>r.json()); }
export async function getSettings(){ return fetch(`${API_BASE}/settings`).then(r=>r.json()); }
export async function postWorkTimes(body){ return fetch(`${API_BASE}/settings/worktimes`, {method:"POST",headers:{"Content-Type":"application/json"},body:JSON.stringify(body)}).then(r=>r.json()); }
```

**说明**：Settings 的 POST/GET 函数需要补全到 `api.js`，并在前端 Settings 组件中使用。

---

## B. `Console.jsx`

**关键 state**：

* `workPeriods`：`[{start,end}]`（显示前三条）
* `countdown`（秒）→ 每秒自减用于样式
* `restMinutes`，`blockedCount`，`isWorking`
  **关键行为**：
* 首选从 `/task_status` 拉数据更新（或订阅 monitor\_data）
* 时间刻度绘制函数：`renderTimeRuler()` 负责根据当前分钟计算左偏移显示红线

---

## C. `Stats.jsx`

**关键 state/方法**：

* `apps`：raw apps `{exe,window,seconds,included}`
* `processedApps`：聚合 <15s 为 `Other`
* `fetchStats()`：调用 `api.js.getStats()` 拉取后端（或 fallback mock）
* `toggleInclude(idx)`：翻转 `included`，并影响 `chartData` 与 `barData`
  **图表**：使用 `recharts`（PieChart, BarChart, LineChart, Area）；所有图表数据应由 `processedApps` 派生，以确保联动一致。

---

## D. `Logs.jsx`

**关键 state/方法**：

* `logs`：`[{time,type,exe,message}]`
* `filter`：`all|info|error`
* `loadLogs()`：`GET /logs`，失败或空时显示“暂无日志”。
* UI：隐藏滚动条；每条 message 用 `<input readOnly />` 或 `<pre>` 使用户可复制。

---

## E. `Settings.jsx`（最终目标）

**结构**：

* 主页面：入口按钮（工作时间 / 黑白名单 / 休息日 / 清零策略）
* 每个按钮打开 Modal（见第 3 节与第 5 节）
  **重要 state（示例）**：
* `workPeriods`, `blacklist`, `whitelist`, `restDays`, `resetPolicy`（初始由 `GET /settings` 填充）
* 每个 Modal 内维护 `localCopy` + `dirty` + `saveError`。
  **保存流程（统一）**：

1. 点击 “保存” → 前端 `POST` 到后端对应接口。
2. 等待响应：

   * 成功（`ok:true` 或 200 + body） → 更新全局 state，关闭 modal。
   * 失败 → 设置 `saveError`（红色提示显示），保留 modal，用户可再次点击保存（多次重试）。

---

# 7. 业务规则与前端显示约定（必须清楚的运作规则）

1. **是否生效由后端判定**：前端只负责 UI 和发送修改请求。后端会返回是否接受修改（`ok:true/false` + message），前端据此决定是否关闭 modal / 更新页面。
2. **编辑/删除权限仅前端提示**：例如「仅在当天最后一个工作时间结束后可删除」，前端应显示为 disabled/提示，但最终权限永远由后端决定。
3. **黑白名单匹配规则**：

   * 支持两类条目：`exe_name`（程序文件名，如 `notepad.exe`）或 `app_name`（窗口标题或其部分）。
   * 前端输入时建议只填写**一种**（`exe` 或 `window`），不要同时写两者在同一行（以免后端混淆或误匹配）。
   * 匹配支持通配符（后端用 `fnmatch`/regex）。
4. **白名单为空行为**：若没有白名单配置，则默认所有非黑名单应用都可计入“工作时长”（并影响奖励逻辑）。
5. **日志写入策略**：只在“应用切换”或明显交互时写入数据库，避免每秒写入。
6. **统计合并策略**：前端把小于 15 秒的项合并为 `Other`，但如果后端已经返回分组，则前端应尊重后端提供的分组（前端只作防御性处理）。

---

# 8. 本地运行与测试命令

## 后端（Python）

1. 推荐创建虚拟环境并安装依赖：

```bash
cd D:\APPS\AT\1.1\backend
python -m venv .venv
.venv\Scripts\activate
pip install fastapi uvicorn psutil pywin32 webview sqlite3
# 注意：pywebview/pywin32 视运行方式选择性安装
```

2. 启动（开发）：

```bash
uvicorn main:app --reload --host 127.0.0.1 --port 8000
# 或直接：
python main.py   # 如果 main.py 启动 uvicorn 或 pywebview
```

3. 快速验证接口：

```bash
curl http://127.0.0.1:8000/logs
curl http://127.0.0.1:8000/stats?period=day
curl http://127.0.0.1:8000/task_status
```

## 前端（Node）

1. 安装依赖：

```bash
cd D:\APPS\AT\1.1\frontend
npm install
# 若有依赖安装问题（插件版本冲突），请贴 package.json 我来修正
```

2. 开发模式：

```bash
npm run dev
# 打开 http://localhost:5173/
```

3. 生产构建（用于嵌入 pywebview/electron）：

```bash
npm run build
# 检查 frontend/dist/index.html 与 assets 是否生成
```

## 启动桌面窗口（pywebview 方式）

* 用 `backend/main.py` 的 pywebview 方案启动（在 main.py 中先启动 uvicorn，然后 `webview.create_window("Time Tracker", "http://127.0.0.1:8000")`）。
* 也可以使用 Electron（需 main.js 指向 `frontend/dist/index.html`）。

---

# 9. Debug & 常见问题检查清单

* **白屏 / Le is not a function / destroy is not a function**

  * 通常来源于打包/模块不匹配或 React Hook 写法错误（例如 `useEffect` 返回 Promise）。查看 DevTools Console（F12）。如果使用 pywebview，开发模式下要用浏览器打开 `http://localhost:5173` 来更方便调试（因 pywebview devtools 有时不可见）。
* **`ERR_CONNECTION_REFUSED`**：后端没有运行或端口不通。先 `curl` 验证后端。
* **Vite 配置错误（vite.config.js 语法）**：修正 vite.config.js JSON/对象语法（之前报错是 `Expected "}" but found "outDir"`）。
* **依赖安装错误（@vitejs/plugin-react 版本）**：确保 package.json 里使用与本地 Node 兼容的依赖版本；如报 `No matching version`，请贴出 package.json 给我。
* **组件命名/大小写错误**（生产构建白屏常见）：确认 `export default function Settings()` 与 `import Settings from "./components/Settings"` 的大小写路径完全匹配（Windows 容易放过但生产打包敏感）。
* **数据库 schema 不一致**：如果 `test.py` 报 `no such column: exe_name`，说明 DB 不是最新 schema：删除旧的 time\_tracker.db 并重新 run `init_db()`，或运行 `ALTER TABLE`。
* **监控未实时杀掉黑名单程序**：确认监控逻辑里是否 `kill` 操作在检测到黑名单时立即执行，而不是只在切换应用时执行（我们之前发现 kill 在切换时才发生 —— 需要在 match 黑名单的分支内立刻执行 `proc.kill()`）。

---

# 10. 你需要提供的文件 / 下次新对话需要粘贴的内容

为便于我直接接手并继续开发/修复，请在新对话中把以下内容一次性贴上（越完整越快）：

**后端（backend/）**

1. `backend/main.py`（完整源代码）
2. `backend/monitor.py`（完整源代码）
3. `backend/db_utils.py`（完整源代码）
4. `backend/api_status.py`（或其它路由文件，完整源代码）
5. `backend/test.py`（如果存在，用于快速测试的脚本）
6. `time_tracker.db`（如果你愿意上传小数据库样本；或至少 `sqlite` 导出的 `PRAGMA table_info` 与前 20 条数据）

**前端（frontend/）**

1. `frontend/package.json`（完整）
2. `frontend/vite.config.js`
3. `frontend/src/main.jsx` 与 `index.html`（或 `dist/index.html` 的头部若没有 src）
4. `frontend/src/App.jsx`
5. `frontend/src/api.js`
6. `frontend/src/components/Sidebar.jsx`
7. `frontend/src/components/Console.jsx`
8. `frontend/src/components/Stats.jsx`
9. `frontend/src/components/Logs.jsx`
10. `frontend/src/components/Settings.jsx`（当前版本）
11. `frontend/dist/index.html`（若你正在使用 build 产物嵌入 pywebview）
12. `start.bat`、`test.py`（项目启动脚本）

**其他信息**

* 你希望使用的桌面运行方式（pywebview / Electron）。
* 如果你有 Windows 的 Edge WebView2、Electron 版本、Node 版本（`node -v`）、npm 版本 请一并给出。
* 如果出现错误：粘贴 **完整** 的浏览器 Console 堆栈、uvicorn 日志和 `npm run dev` 的错误输出。

---

# 11. 下一步优先任务（短期计划）

1. 我来完成并提交 **Settings.jsx 的最终实现**（Modal 内 `time` 输入、保存失败不关闭、重试、后端请求占位），并把 `api.js` 的 settings API 扩展好。
2. 你把上述第10项中能立刻提供的文件粘给我（尤其是 `main.py`、`db_utils.py`、`monitor.py`、`Settings.jsx`、`package.json`、`vite.config.js`）。
3. 我把 Settings 前端完成后，我们做一次联调：我会给出具体 curl / 浏览器步骤检查后端是否实现 `/settings/*`。
4. 修复任何生产构建中发现的白屏或依赖问题（必要时我会给出修正的 `package.json` 与 `vite.config.js`）。

---

# 附：示例检查命令（把它加入你的笔记）

* 验证后端是否运行：

  ```bash
  curl http://127.0.0.1:8000/task_status
  curl http://127.0.0.1:8000/logs
  curl -X POST http://127.0.0.1:8000/settings/worktimes -H "Content-Type: application/json" -d '{"workPeriods":[{"start":"09:00","end":"12:00"}]}'
  ```
* 检查 sqlite 表结构（在 backend 目录）：

  ```py
  import sqlite3
  conn=sqlite3.connect('time_tracker.db')
  print(conn.execute("PRAGMA table_info(app_usage)").fetchall())
  print(conn.execute("SELECT id, start_time, exe_name, app_name, duration FROM app_usage ORDER BY id DESC LIMIT 5").fetchall())
  ```
* 前端 dev 测试：

  ```bash
  cd frontend
  npm run dev
  # 打开 http://localhost:5173/ 观察 Console / Network / Errors
  ```

---

如果你现在就要切换工作台：

1. 把本 `PROJECT_OVERVIEW.md` 保存到项目根（`D:\APPS\AT\1.1\PROJECT_OVERVIEW.md`）。
2. 下次新对话把 **第 10 节** 列出的文件一次性粘过来（尤其是 `backend/main.py`、`db_utils.py`、`monitor.py`、`Settings.jsx`、`package.json`、`vite.config.js`）。我会在新对话里 **直接修改/提交具体代码文件**（并把修改过的文件完整地粘回给你），继续按优先级推进。

需要我现在把这文档生成为文件并放到项目（例如 `/mnt/data/PROJECT_OVERVIEW.md`）吗？或者现在就把 `Settings.jsx` 的最终实现直接生成并给出完整代码？
