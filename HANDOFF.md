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

- **当前 sha**: `55be4b8c6ebdfa1f37c1cb935645cac61974aebd`（V3 + maps chunk fix）
- **提交 commit**: [`d3fa87e9`](https://github.com/kkofor/route-optimizer/commit/d3fa87e9bc6acdfceb5b805720b36e4e6f5b8ee6) — 2026-05-06 16:01 UTC
- **原版 sha（参考）**: `6bfa3434de71a5ae7523a1213e42f65d210fc2ab`
- **字符数**: 22807 → 25768（+2961 chars）
- **改动**:
  - 算法 V3（多起点贪心 + or-opt VND）— L375–514
  - Google Maps 分段 `CHUNK 9 → 7`（每段总地址 ≤ 8，避免 app 卡死）
  - UI 文案：`真实路网矩阵 + 2-opt优化` → `多起点贪心 + 2-opt + or-opt`
- **离线测试**：n=12×200 次 0/200 回归；n=8 vs 穷举最优 3/3 命中；n=25 stress ≤175ms

---

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

1. **MapLibre GL 地图可视化** — 在页面内显示路线图，不跳转 Google Maps
2. **Android Accessibility Service** — 自动抓取 Spoke app 订单，不需要截图
3. **圆心南移策略** — 路线末尾优先南边站点，影响派单系统给更多南区单
4. **Ruin-and-Recreate** — 参考 PyVRP/jsprit，新单加入时重新优化而不是简单插入
5. **时间窗 / LATE 单权重** — 目标函数加入截止时间约束

---

更新时间: 2026-05-06（V3 已部署 + Maps chunk 修复，commit d3fa87e9）
