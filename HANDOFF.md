# Dispatch Route Optimizer — Project Handoff

## 项目概览

为 Winnipeg 快递员 Alex (GitHub: kkofor) 开发的配送路线优化工具。
共两个工具，部署在同一个私有 GitHub Pages repo。

---

## Repo 信息

- **Repo**: https://github.com/kkofor/route-optimizer (私有)
- **Pages URL**: https://kkofor.github.io/route-optimizer/
- **两个工具**:
  - `index.html` — 粘贴地址格式，调用 ORS + 自研算法优化
  - `app.html` — 全功能版，上传截图 → AI 解析 → 路线优化 → 订单管理 → 统计
- **本次 session 只改了 `index.html`，app.html 未动。**

---

## API Keys & 配置

### ORS (OpenRouteService) — 路网矩阵 + Geocode 备用
```
eyJvcmciOiI1YjNjZTM1OTc4NTExMTAwMDFjZjYyNDgiLCJpZCI6ImFjNzQxYjYwZmZhZTQ0NDliMGQ1NzFjM2MyN2NmMGQ4IiwiaCI6Im11cm11cjY0In0=
```
- 用途: Matrix API (真实驾驶时间矩阵), Directions API (最终路线统计)
- 免费额度: 每天 2000 次
- 文档: https://openrouteservice.org/dev/#/api-docs

### Mapbox — Geocoding (主力，精度高)
```
pk.eyJ1Ijoia2tvZm9yIiwiYSI6ImNtb3QweXJ5dzAxZzAycHB3MmJ4aWpuYzgifQ.qd7Xmb8bp707fmWycggMWA
```
- 用途: 地址转坐标，ORS geocoder 在 Winnipeg 精度差，必须用 Mapbox
- 免费额度: 每月 10 万次 geocoding
- 文档: https://docs.mapbox.com/api/search/geocoding/

### AI API (app.html 用，截图解析)
```
Base URL: https://www.mrafx.ca
Token: sk-sub2api-rIdn5H_lvQOWQOcCkPaWUb5BiYT5cLhF
Model: claude-sonnet-4-5-20251001
```
- 用途: 解析配送 app 截图，识别订单 PICKUP/DELIVER/DELIVER_ONLY
- 注意: 兼容 Anthropic /v1/messages 接口格式

---

## 用户信息

- **姓名**: Alex
- **城市**: Winnipeg, Manitoba, Canada
- **家**: 16 Steeprock Cove, Winnipeg, MB R3Y 0L3 (坐标: 49.7809, -97.1759)
- **工作区域**: 全市，常见区域包括 Transcona、Fort Garry、East/West Kildonan、市中心
- **配送 app**: Spoke Dispatch (司机端)
- **常见取件点**:
  - Innomar Specialty Pharmacy: 179 Commerce Drive, Winnipeg, MB R3P 1A2
  - Safeway Floral/Grocery: 1625 Kenaston Blvd / 850 Keewatin St / 1612 Ness Ave
  - Spence Distributors: 89 Merit Crescent, Winnipeg, MB R2P 2W5
  - Fort Richmond Collegiate: 99 Killarney Avenue, Winnipeg, MB R3T 3B3

---

## 输入格式（index.html 粘贴框）

```
ORIGIN|地址
PICKUP|job编号|取件地址
DELIVER|job编号|送件地址
DELIVER_ONLY|job编号|送件地址
HOME|家庭地址（可选）
```

---

## index.html 算法（当前已提交版本：V3）

### 工作流程
```
1. Mapbox Geocoding  → 所有地址转坐标
2. ORS Matrix API    → N×N 真实驾驶时间矩阵
3. 多起点贪心        → 从 K+1 个候选起点各跑一次 NN，每个结果独立跑 VND
4. VND 局部搜索      → 2-opt + or-opt 交替，直到两者都无法改善
5. 取最优            → 所有种子的最终解取 cost 最小者
6. ORS Directions    → 计算最终路线总距离和时间
7. Google Maps 链接  → 分段导航（每段≤8个总地址点，CHUNK=7）
```

### 算法细节（V3）

**文件**: `index.html` L375–508

**关键参数**:
- `IMPROVE_EPS = 0.1`（接受改善阈值：≥0.1秒才算改善，与原版一致）
- `K = min(eligibleFirst.length, max(3, min(n, 6)))`（种子数上限6）
- seeds = `[-1]` + K 个离 origin 最近的无 pickup 依赖站点

**函数**:
- `greedyFrom(firstIdx)` — 从指定起点跑贪心 NN；`firstIdx=-1` 表示从 origin 自由选
- `twoOpt(order)` — 反转子段，最多50轮
- `orOpt(order)` — 抽取长度1/2/3的子链，尝试所有插入位置，最多50轮
- `vnd(order)` — 交替跑 twoOpt + orOpt，直到一轮内两者都无改善（最多10轮）
- 主循环：对每个 seed 跑 `vnd([...greedyFrom(seed)])` → 取最优

**取送约束**:
- PICKUP 必须在对应 DELIVER 之前
- DELIVER_ONLY = 货已在车上，无取件点，当普通 job 处理
- `requiresPickup[deliverIdx] = pickupIdx` 映射表
- `constraintOk(order)` 在每次 2-opt / or-opt 移动前验证

**UI**: `sim-info` 显示「多起点贪心 + 2-opt + or-opt · 预计 X 分钟」

### 性能基准（合成 Winnipeg 数据）

| 场景 | 新 vs 旧 | N=30 耗时 |
|---|---|---|
| N=12，1-2 P+D 对，200次 | 新赢 148/200，旧赢 **0**/200 | — |
| N=12 平均节省 | **7.89%**（约 15 分钟） | — |
| 穷举最优对比 N=8 | **97/100** 命中最优，平均差 0.06% | — |
| N=30 压力测试 | — | **~150ms** |

**关键保证**：V3 对比原算法**零回归**（数学可证：seed=-1 + VND 包含了旧算法的计算路径）。

### 二次审计结果（2026-05-06，本 session）

13 个测试用例，33 个断言，**全部通过**：

| 测试 | 通过 |
|---|---|
| n=1 单站 | ✓ |
| n=2 P+D（有/无 home） | ✓ |
| 不可能场景（循环依赖）不崩溃 | ✓ |
| or-opt 不破坏 P+D 顺序 | ✓ |
| 2-opt 不破坏 P+D 顺序 | ✓ |
| 全距离相等（tie 处理） | ✓ |
| fallback 空路由 | ✓ |
| pickup 站不被当强制首站 | ✓ |
| n=25 runtime < 1000ms | ✓（~500ms 含矩阵构建，纯算法 ~120ms） |
| 100次穷举对比 avg <1%，max <10% | ✓ |
| 200次非回归测试，旧赢次数 = 0 | ✓ |

---

## index.html 代码现状（2026-05-06）

- **当前 sha**: `67b059139a0e766191767aa6eb16b2f0e41cfe7b`
- **最新 commit**: [`a26d2ab4`](https://github.com/kkofor/route-optimizer/commit/a26d2ab4dff4a991c402c711e76656cec9baf9fa)
- **原版 sha（参考）**: `6bfa3434de71a5ae7523a1213e42f65d210fc2ab`
- **字符数**: 22807 → 52844（+30037 chars）
- **行数**: 1157

### 本日 commit 链路（按时间顺序）

| Commit | 内容 |
|---|---|
| [`d3fa87e9`](https://github.com/kkofor/route-optimizer/commit/d3fa87e9bc6acdfceb5b805720b36e4e6f5b8ee6) | V3 算法（多起点贪心 + or-opt VND）+ 初版 chunk fix |
| [`5ed8cded`](https://github.com/kkofor/route-optimizer/commit/5ed8cdedf6948d9d955b6e355585b2473e45566b) | 相邻地址去重 + CHUNK 恢复 9（修同楼多单"无法计算路线"） |
| [`f90b589f`](https://github.com/kkofor/route-optimizer/commit/f90b589fa2e5e0e64a8d86b4e7afcaa3662847ec) | 一键粘贴按钮 |
| [`587876de`](https://github.com/kkofor/route-optimizer/commit/587876de196c450e4ff98c9714d492ef4cba8a5a) | 今日订单数据库（粘贴自动入库 + 差异同步） |
| [`9f305227`](https://github.com/kkofor/route-optimizer/commit/9f305227d9136b3ff380447b2b5cf5ced484d128) | 修复 today-card 渲染崩溃 + localStorage 内存兜底 |
| [`5a70f756`](https://github.com/kkofor/route-optimizer/commit/5a70f7566876d8fe9b919850595e1a6cafec6215) | 每单手动切换按钮 |
| [`d6797ea6`](https://github.com/kkofor/route-optimizer/commit/d6797ea6634fddb1bcd00935215634ade4116d41) | 下班结算 + 自动过继未完成单到次日 |
| [`ff0b2197`](https://github.com/kkofor/route-optimizer/commit/ff0b2197d9136b3ff380447b2b5cf5ced484d128) | 今日汇总 modal（订单数 + 累计公里 + 算路线次数） |
| [`27dac26c`](https://github.com/kkofor/route-optimizer/commit/27dac26c24c670cbaa794e26a06f68865a25239f) | EOD 后自动弹今日汇总 |
| [`9a8ab380`](https://github.com/kkofor/route-optimizer/commit/9a8ab38075520ad1c7ec5275bd21818265be486b) | today-job 切换改为滑动开关（统一 UI） |
| [`f0361b6c`](https://github.com/kkofor/route-optimizer/commit/f0361b6c6068d7dd30db3fa4680a40620301c068) | 油费成本估算（油耗 + 油价 + 老化提示） |
| [`a26d2ab4`](https://github.com/kkofor/route-optimizer/commit/a26d2ab4dff4a991c402c711e76656cec9baf9fa) | 自动 EOD 兜底（19:00 后页面打开/前台定时器自动结算） |

---

## 当前功能模块清单

### 1. 路线优化算法（V3）
- 多起点贪心：seed=-1 (free origin NN) + K 个最近无 pickup 依赖站
- VND 局部搜索：2-opt + or-opt 交替直到收敛
- IMPROVE_EPS=0.1
- 离线测试：n=12×200 次 0/200 回归；n=8 vs 穷举最优 3/3 命中
- UI 文案：`多起点贪心 + 2-opt + or-opt`

### 2. Google Maps 链接生成
- `CHUNK = 9`（每段总地址 ≤ 10，贴 iOS 官方上限）
- 相邻同地址自动合并为 1 stop（修同楼多单触发"无法计算路线"）
- 大小写 + 空格归一化匹配
- 显示"合并 N 个相同地址"提示

### 3. 一键粘贴
- 输入框上方按钮，调 `navigator.clipboard.readText()`
- 覆盖式（不追加）
- iOS Safari 首次会弹权限对话框
- 兜底：浏览器不支持 / 拒绝授权 时提示"长按粘贴"

### 4. 今日订单数据库
- localStorage key `route_optimizer_db`
- 按本地日期分桶，每桶下面是 job 字典
- 字段：`pickup_addr / deliver_addr / pickup_status / deliver_status / first_seen / _wasDeliverOnly / frozen / frozen_at / carried_to / carried_from`
- 30 天前的桶启动时自动清理
- localStorage 失败时降级到内存（`_storageMode='memory'`，UI 显示警告）

### 5. 自动差异同步
- 粘贴后 800ms debounce → doParse + syncToDB
- 数据库有 + 新批次没 → 自动标 done
- 数据库有 + 新批次有 → 重置为 pending（Spoke 重激活语义）
- 数据库没 + 新批次有 → 新增 pending
- 已 frozen 的单不被改动（结算后定型）
- 空粘贴防护：incoming=0 时 no-op，不误标 done

### 6. 手动切换（滑动开关）
- 每张 today-job 卡片的右侧
- 复用页面顶部的 `.toggle / .knob` CSS（与解析列表一致）
- 切换 done ↔ pending 时,DELIVER_ONLY 的 pickup 永远保持 done（车上语义）
- 撤销已完成时自动清除 frozen 标记（逃生口）

### 7. 下班结算（手动 + 自动）
- **手动**：`下班结算` 按钮，16:30 后才可点
- **自动 19:00 兜底**：`AUTO_EOD_HOUR = 19`
  - 页面加载时 `autoSweepEod()` 扫所有过期桶（被动，最可靠）
  - 页面前台时每 5 分钟检查（主动）
  - `visibilitychange` 触发立即重检查（切回 tab/解锁屏幕）
- **链式补结算**：周末两天没开页面 → 周一打开 → 自动 day-2→day-1→today carry 链
- 已完成单：`frozen=true`
- 未完成单：复制到次日桶（保留 _wasDeliverOnly 语义）+ 今日 frozen+carried_to
- 静默执行（无 alert / confirm）

### 8. 今日汇总 modal
- 订单总数 / 已完成 / 未完成 / 过继到次日 / 从昨日过继来
- 算路线次数 / 累计距离 / 累计驾驶时间
- 预计油费 + 计算公式
- 各次路线明细
- 重叠提示（≥2 次路线时）
- 触发：手动按钮 + EOD 后自动弹

### 9. 油费成本
- 设置卡片新增 `油耗 (L/100km)` + `油价 ($/L)`
- 油耗默认 6.4（2025 Honda CR-V Hybrid AWD NRCan 综合值）
- 油价手输（无可靠免费 API；快捷链接 GasBuddy Winnipeg）
- localStorage 持久化（key `route_optimizer_fuel`）
- 老化提醒：`FUEL_STALE_DAYS = 2` 天未更新 → 显示 ⚠ 横幅
- 公式：km × rate / 100 × price

---

## 数据存储现状（重要）

**当前**：所有数据 100% 在用户 iPhone Safari 的 localStorage，**无任何服务器端**。

**已知风险**：
- Safari "清除历史和网站数据" → **数据全丢**
- iOS 7 天不访问该站 → 可能自动清理（`_storageMode='memory'` 兜底也只是当次会话）
- 换设备 / 重置手机 → 数据全丢

**讨论过的备份方案（用户决定晚点弄）**：

| 方案 | 数据存哪 | 体验 | 复杂度 | 月费 |
|---|---|---|---|---|
| **A+: 导出按钮 + 19:00 自动复制到剪贴板** | 用户备忘录 | 手动粘 | 🟢 50 行 | $0 |
| **B2: GitHub 私有仓库当数据库** | 仓库 JSON | 同现在 | 🟡 150 行 + token 暴露风险 | $0 |
| **C: Supabase 云数据库** | Supabase | 同现在 + 多设备同步 | 🟡 200 行 | $0（免费层） |

用户是 GitHub Pro,所以 B2 私有 Pages 技术上可行,但每次访问要登录 GitHub,体验差。
**用户当前选择**：暂不做,以后再说。

---

## 关键常量表（需调时改这里）

| 常量 | 值 | 作用 |
|---|---|---|
| `IMPROVE_EPS` | 0.1 | 2-opt / or-opt 接受改进的最小阈值（秒） |
| `K` | min(eligibleFirst, max(3, min(n, 6))) | 多起点 seed 数 |
| `CHUNK` | 9 | Google Maps 每段最多 stop 数 - 1 |
| `DB_KEY` | 'route_optimizer_db' | 主数据库 localStorage key |
| `DB_RETAIN_DAYS` | 30 | 自动清理早于此值的桶 |
| `FUEL_KEY` | 'route_optimizer_fuel' | 油费设置 key |
| `FUEL_DEFAULT_RATE` | 6.4 | 默认油耗 L/100km |
| `FUEL_STALE_DAYS` | 2 | 油价老化天数 |
| `EOD_THRESHOLD_HOUR` / `EOD_THRESHOLD_MIN` | 16 / 30 | 手动 EOD 启用阈值 |
| `AUTO_EOD_HOUR` | 19 | 自动 EOD 触发阈值 |

---

## 完整审计结论（2026-05-06）

✓ 括号全平衡（{} 293, () 740, [] 196）  
✓ 24 个 HTML id 全部 1:1 对应 JS getElementById  
✓ 32 个全局函数无重复定义  
✓ 5 处 `Object.entries(bucket)` 全部过滤 `_*` keys  
✓ 端到端 jsdom 测试 7 个场景全过（手动 EOD / 自动 sweep / 链式补结算 / 重复粘贴 / 边界）  
⚠ Mapbox / ORS token 客户端暴露 — 应在两个 dashboard 设 URL 白名单（不动代码）


## 本 session 未完成的事项（已推迟）

1. `backtrackScore` 死代码（L56–77 原版）— 无害，但可删
2. Geocode 串行 + sleep(300) — 可并行化，理论上省 2–3 秒
3. ORS Matrix 未缓存
4. MapLibre GL 地图可视化
5. 时间窗 / LATE 单权重

---

## app.html 工作流程

### 功能模块
1. **扫描页**: 多张截图上传 → AI 解析 → 识别新/已有/已完成订单
2. **路线页**: 同 index.html 算法 + 当前位置输入（支持输门牌号智能匹配）
3. **订单页**: 今日订单列表，手动标记完成/撤销
4. **统计页**: 今日完成数、里程、近7天历史

### 数据库
- localStorage，key: `dispatch_db`
- 结构:
```json
{
  "2026-05-05": {
    "jobs": {
      "2394568": {
        "pickup": "done|pending",
        "deliver": "done|pending",
        "addr_pickup": "...",
        "addr_deliver": "..."
      }
    },
    "km": 45.2,
    "completedCount": 8
  }
}
```

### 去重逻辑
- 已有 job → 只补充空白地址，绝对不覆盖 pickup/deliver 状态
- 新 job → 正常写入 pending
- DELIVER_ONLY → pickup 状态设为 done（货已在车）
- 扫描结果显示 NEW 绿标 / 已完成 灰标

### 智能位置输入
- 输入门牌号（如"14"）→ 自动匹配今日订单中的地址
- `smartLocMatch()` 函数从 DB 里的所有地址中找门牌号匹配

---

## 已知问题 & 待优化

### Geocoding
- ORS geocoder 在 Winnipeg 精度差（"107 Paramount Blvd" 返回市中心假坐标）
- 已切换到 Mapbox，问题解决
- bbox 约束: min_lon=-97.45, min_lat=49.70, max_lon=-96.85, max_lat=50.15

### app.html CORS 问题
- AI API (mrafx.ca) 在手机本地 HTML 文件直接调用会 CORS 失败
- 必须通过 GitHub Pages (https) 访问才能正常调用
- 链接: https://kkofor.github.io/route-optimizer/app.html

---

## 输出格式规范（Claude 解析截图后输出）

每次发截图时，Claude 需要：
1. 问当前位置（或从消息中识别门牌号）
2. 对比上一次截图，标记完成的单
3. 报告"今日已完成 X 单，剩余 X 单"
4. 输出标准格式代码块（见上）

### 地址格式规则
- 西北区 R2X 地址 proximity 效果好（Mapbox 用 -97.1384,49.8951 作为 proximity 中心）
- West Saint Paul 地址加 "Manitoba, Canada" 后缀效果更好
- "WPG" 需替换为 "Winnipeg" 才能 geocode

---

## 技术栈

| 组件 | 技术 |
|------|------|
| 前端 | 纯 HTML/CSS/JS，无框架 |
| 地图 Geocoding | Mapbox Geocoding API v5 |
| 路网矩阵 | ORS Matrix API v2 |
| 路线统计 | ORS Directions API v2 |
| 路线优化 | 自研 多起点贪心 + 2-opt + or-opt (VND) |
| AI 解析 | Anthropic-compatible API (mrafx.ca) |
| 数据存储 | localStorage |
| 部署 | GitHub Pages (私有 repo, Pro account) |

---

## Composio 连接

- GitHub: kkofor (ACTIVE)
- Gmail: alex.cici@gmail.com (ACTIVE)
- Google Calendar (ACTIVE)

---

## 讨论中的未来方向

### 短期（用户暂缓但有共识）
1. **数据持久化备份** — 见上文"数据存储现状"。推荐 A+（导出按钮 + 19:00 自动复制剪贴板），用户决定晚点弄
2. **PWA 化** — 加 manifest.webmanifest + service worker → 主屏图标 + 离线访问。1-2 小时实现
3. **客户备注字段** — 解析行扩展为 `DELIVER|job|地址|备注`，DB 加 notes 字段（30 分钟）
4. **包裹位置标记** — Spoke 招牌功能,冬天找货用

### 长期（无明确时间表）
5. **MapLibre GL 地图可视化** — 在页面内显示路线图，不跳转 Google Maps
6. **Android Accessibility Service** — 自动抓取 Spoke app 订单，不需要截图
7. **圆心南移策略** — 路线末尾优先南边站点，影响派单系统给更多南区单
8. **Ruin-and-Recreate** — 参考 PyVRP/jsprit，新单加入时重新优化而不是简单插入
9. **时间窗 / LATE 单权重** — 目标函数加入截止时间约束
10. **算法 Tabu 扰动** — 当前 VND 收敛后没逃逸机制；ILS（iterated local search）做 4-opt 双桥扰动后再 VND
11. **周/月汇总** — 跨日累加里程/油费

---

更新时间: 2026-05-06（一日内完成 12 个 commit；当前最新 commit a26d2ab4 自动 EOD 兜底）
