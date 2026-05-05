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

## index.html 工作流程

### 用法
1. Alex 发截图给 Claude
2. Claude 解析后输出标准格式（代码块）:
```
ORIGIN|当前位置地址
PICKUP|Job号|取件地址
DELIVER|Job号|送件地址
DELIVER_ONLY|Job号|送件地址（车上已有货）
HOME|16 Steeprock Cove, Winnipeg, MB R3Y 0L3
```
3. Alex 复制粘贴进工具 → 一键优化

### 算法流程 (index.html)
```
1. Mapbox Geocoding  → 所有地址转坐标
2. ORS Matrix API    → N×N 真实驾驶时间矩阵
3. 贪心初始解        → nearest-neighbor，遵守取送约束
4. 2-opt 改善        → 消除路线交叉，最多50次迭代
5. ORS Directions    → 计算最终路线总距离和时间
6. Google Maps 链接  → 分段导航（每段≤9个途经点）
```

### 取送约束
- PICKUP 必须在对应 DELIVER 之前
- DELIVER_ONLY = 货已在车上，无取件点，当普通 job 处理
- requiresPickup[deliverIdx] = pickupIdx 映射表

### 已验证
- 穷举对比：9站时算法结果与穷举最优差距仅 4 秒（0.1分钟）
- Mapbox geocoding 解决了 ORS 在 Winnipeg 找不到地址的问题

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

### 算法层面
1. **or-opt 未实现** — 2-opt 无法移动单个站点到最优位置，or-opt 可以解决孤立远端单问题
2. **多起点贪心未实现** — 现在只跑一次贪心，多起点取最优可提升初始解质量
3. **目标函数单一** — 只优化驾驶时间，未加 LATE 单权重

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
| 路线优化 | 自研 贪心 + 2-opt |
| AI 解析 | Anthropic-compatible API (mrafx.ca) |
| 数据存储 | localStorage |
| 部署 | GitHub Pages (私有 repo, Pro account) |

---

## 讨论中的未来方向

1. **or-opt + 多起点贪心** — 下一步算法改进
2. **MapLibre GL 地图可视化** — 在页面内显示路线图，不跳转 Google Maps
3. **Android Accessibility Service** — 自动抓取 Spoke app 订单，不需要截图
4. **圆心南移策略** — 路线末尾优先南边站点，影响派单系统给更多南区单
5. **Ruin-and-Recreate** — 参考 PyVRP/jsprit，新单加入时重新优化而不是简单插入

---

## Composio 连接

- GitHub: kkofor (ACTIVE)
- Gmail: alex.cici@gmail.com (ACTIVE)
- Google Calendar (ACTIVE)

---

生成时间: 2026-05-05
