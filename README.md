# 巡展作品光照预算 Web 工作台

为巡展作品管理展期曝光预算的本地 Web 工具：Node.js + SQLite + 浏览器页面，单机本地运行。

- **作品维度**：录入展期曝光上限与既有累计量，实时显示已用 / 预占 / 余额（单位 lux·min）。
- **共享额度组**：易损作品可共用一笔曝光额度。策展人维护额度组的名称、**非负上限**与成员；**每件作品至多归属一组**，成员重复、不存在或已归属其他组一律拒绝。组已用 = 成员既有累计量 + 已结束时段实际曝光；组预占 = 成员进行中冻结计划 + 未开始计划曝光；**同一时段有多件组内作品时逐件计入**。单件上限与组上限须**同时**满足。
- **时段维度**：为展厅时段录入起止时间、照度（lux）和展出作品，按 **实际经过分钟 × 照度** 计算曝光。
- **多段照度**：巡展现场实际照度可能随时间变化。进行中时段可按**实际起止 + 多段照度**结算，每段含起止与非负照度，曝光 = Σ（段内实际分钟 × 照度）；分段必须按时间顺序首尾衔接、无重叠且完整覆盖实际区间，否则**整批拒绝**。
- **整批排期**：未开始时段在草稿中统一编辑后整批提交；同一作品时段重叠、或“已用 + 预占”超上限时 **拒绝整批**，逐条指出冲突作品与时段。
- **生命周期**：时段开始即冻结原排期并保留预占；结束时补记实际起止与照度（单一或多段），**实际曝光替换该时段预占**；结算超量后按开始时间/编号顺序标出不可启动时段与缺口并禁止启动。
- **结算更正**：已结束时段可凭理由提交**完整分段更正**（起止沿用原实际区间），系统记录更正前后的曝光、时间与分段；**新总量替换旧总量，不重复累计**，并同步重算作品/额度组已用量、后续时段不可启动状态与缺口；更正历史只增不改。
- **组配置调整**：结算、更正或调整额度组（上限/成员）后，按现有时序重算组/作品余额与不可启动时段；**组上限低于已用量时仍保留历史与进行中状态，仅阻止后续启动**，页面与接口展示相应缺口。组配置变更同样携带版本号，过期返回 409，失败不推进版本、不改变记录。
- **乐观并发**：所有调整（含作品/额度组、多段结算与更正）携带当前版本号，过期版本返回 409 与最新预算，失败不改变结算与版本。

## 一键启动（Docker Compose，推荐）

```bash
docker compose up -d --build
```

启动后打开：

- **页面地址**：http://localhost:8080
- 初始化：首次启动自动建表并写入演示数据（3 件作品 + 已结束/进行中/未开始时段）。

如需更换宿主机端口：

```bash
PORT=9090 docker compose up -d --build   # 页面变为 http://localhost:9090
```

停止与重置初始数据：

```bash
docker compose down            # 停止，数据保留在命名卷中
docker compose down -v         # 同时删除数据卷，下次启动重新初始化
```

## 本地直接运行（需要 Node.js ≥ 20）

```bash
npm install
npm start
# 打开 http://localhost:8080
```

## 配置（环境变量）

| 变量 | 默认值 | 说明 |
| --- | --- | --- |
| `PORT` | `8080` | HTTP 监听端口 |
| `HOST` | `0.0.0.0` | 监听地址（本地直跑可设 `127.0.0.1`） |
| `DB_PATH` | `./data/lightbudget.db`（Docker 内为 `/app/data/lightbudget.db`） | SQLite 数据库文件路径；文件不存在时自动建库并初始化 |

## 计算与校验规则

- **曝光量** = （结束 − 开始，分钟）× 照度 lux，单位 **lux·min**。
- **多段结算曝光量** = Σ各段（段结束 − 段开始，分钟）× 该段照度；每段照度须为非负数。
- **分段完整性**（多段结算 / 更正共用）：
  1. 每段时间格式合法、结束晚于开始、照度非负；
  2. 分段按时间顺序排列，第 *n* 段开始必须等于第 *n−1* 段结束（首尾衔接、无缺口、无重叠）；
  3. 首段开始 = 实际区间开始、末段结束 = 实际区间结束（完整覆盖）。
  
  任一条不满足返回 `422 SEGMENTS_INVALID`，`segmentErrors` 逐条说明，整批不落库。
- **已用** = 作品既有累计量 + 所有已结束时段的**实际**曝光（多段结算取分段求和值；更正后为最新总量，旧值不保留在累计中）。
- **预占** = 进行中时段的（冻结）计划曝光 + 所有未开始时段的计划曝光之和。
- **余额** = 上限 − 已用 − 预占；余额为负即超量。
- **共享额度组**：
  - 组**已用** = Σ 成员（既有累计 + 已结束实际曝光）；组**预占** = Σ 成员（进行中冻结预占 + 未开始计划曝光）；组**余额** = 组上限 − 组已用 − 组预占。
  - 同一时段若同时展出多件同组作品，每件各计一份曝光（逐件计入）。
  - 每件作品至多归属一个额度组；单件上限与组上限**同时**生效，任一不足即阻断。
  - 组配置（名称 / 非负上限 / 成员）保存即按现有时序重算；组上限低于组已用量时，历史已结束时段与进行中时段保持不变，仅把后续未开始时段按顺序标记为不可启动，缺口为负值可用余额下的差额。
  - 删除额度组仅解除成员归属（`group_id` 置空），作品、单件预算与时段历史均保留。
- 排期整批保存时校验：
  1. 时间格式合法、结束晚于开始、照度/累计量非负（无效时序或负值一律拒绝）；
  2. 同一作品在任意两个未开始时段、以及与进行中（冻结区间）/已结束（实际区间）时段之间不得重叠；
  3. 每件作品「既有 + 已结束实际 + 进行中预占 + 本批未开始预占」不得超过其单件展期上限；
  4. 每个额度组「组内成员已用（含进行中冻结）+ 本批各成员未开始预占之和」不得超过组上限（同槽多件逐件计入）。
- 任一条不满足则整批不落库，响应中给出作品、冲突时段对、或额度组（含每个成员的贡献与整组超额量）。
- **结算更正规则**：
  - 仅 `finished` 时段可更正；更正必须带 `reason`（1–500 字）；
  - 更正分段的实际起止**沿用该时段原实际起止**（更正只调整照度分段，不改变展出区间）；
  - 每次更正写一条 `slot_corrections` 记录：理由、更正前后起止/照度/分钟数/曝光/完整分段、时间戳与序号（只增不改）；
  - 新总量**替换**旧总量（不重复累计），随后重算作品已用量、未开始时段不可启动标记与缺口；
  - 更正同样需要携带当前版本，过期返回 409 与最新预算。
- **不可启动核算**：每次调整/启动/结算/更正、以及额度组配置变更后，未开始时段按开始时间（再按编号）顺序尝试占用预算；任一关联作品**单件余额不足或其所属额度组余额不足**即标记为“不可启动”，并分别在 `blockedReason.gaps`（单件）与 `blockedReason.groupGaps`（组，含组成员贡献）中列出可用余额、需求与缺口，启动接口对该时段返回 422。调整排期或组配置保存后自动重新核算。
- **并发版本**：每次成功变更版本号 +1。请求必须携带当前 `version`；不匹配时返回 `409 VERSION_STALE` 与最新完整预算（含 `groups`），前端自动刷新后需重新调整提交。**校验失败不会推进版本、不改变任何结算/配置数据。**

## 接口一览

| 方法 | 路径 | 说明 |
| --- | --- | --- |
| GET | `/api/state` | 最新版本、**额度组汇总 `groups`**、作品预算汇总、全部时段（含 `actualSegments` 与 `corrections` 更正历史）及不可启动核算结果 |
| POST | `/api/artworks` | 录入作品（body 带 `version`） |
| PUT | `/api/artworks/:id` | 修改作品上限/既有量等 |
| DELETE | `/api/artworks/:id` | 删除作品（被时段引用时拒绝） |
| POST | `/api/groups` | **新建共享额度组**：`{version, name, limit, note?, members:[artworkId,...]}` |
| PUT | `/api/groups/:id` | **整组更新**名称/非负上限/备注/成员（成员整组替换；每件作品至多一组） |
| DELETE | `/api/groups/:id` | 删除额度组（成员解除归属，作品与历史保留） |
| POST | `/api/schedule` | **整批替换未开始时段**（body：`{version, slots:[...]}`）；单件与组上限同时校验 |
| POST | `/api/slots/:id/start` | 启动时段：冻结原排期、保留预占（单件/组缺口均 422 阻断） |
| POST | `/api/slots/:id/finish` | 结束补记（单一照度，兼容语义不变）：`{actualStart, actualEnd, actualLux}`，实际曝光替换预占并重新核算 |
| POST | `/api/slots/:id/settle` | **多段照度结算（进行中时段）**：`{version, actualStart, actualEnd, segments:[{start,end,lux}]}` |
| POST | `/api/slots/:id/correct` | **已结束时段完整分段更正**：`{version, reason, segments:[{start,end,lux}]}`，起止须完整覆盖原实际区间 |

### 额度组数据结构

`GET /api/state` 的 `groups[]` 每项：

- `id` / `name` / `limit` / `note` / `memberCount`；
- `used`（组已用）、`reservedRunning` / `reservedScheduled`、`reserved`、`committed = used + reserved`、`balance`、`over`；
- `members[]`：逐成员给出 `artworkId`、`artworkName`、`used`、`reservedRunning`、`reservedScheduled`、`reserved`、`committed`。

作品对象新增 `groupId` / `groupName`（无组时为 `null`），其余字段保持不变。未开始时段被组额度阻断时，其 `blockedReason` 为：

```json
{
  "gaps": [],
  "groupGaps": [
    { "kind": "group", "groupId": 1, "groupName": "纸本/水彩易损作品共享额度",
      "has": -100400, "need": 21600, "gap": 122000,
      "contributions": [ { "artworkId": 1, "artworkName": "《山水轴》纸本水墨", "need": 21600 } ] }
  ]
}
```

> 单件缺口仍在 `gaps[]`（字段与历史一致）；两类缺口可能同时出现。整批保存因组超限被拒绝时返回 `422 BATCH_REJECTED`，错误体新增 `groupBudgetExceeded[]`（含 `limit/used/scheduledNeed/projected/overflow` 与逐成员 `contributions`），原 `budgetExceeded[]`（单件）保持不变。

`GET /api/state` 中已结束时段的字段：

- `actualStart` / `actualEnd` / `actualMinutes` / `actualExposure`：实际区间与总量；
- `actualLux`：仅单段时为该段照度，多段时为 `null`；
- `actualSegments`：`[{start,end,lux,minutes,exposure}, ...]` 完整分段；
- `corrections`：更正历史数组，每条含 `seqNo`、`reason`、`createdAt` 与 `before`/`after`（各自含 `start,end,lux,minutes,exposure,segments`）。

## HTTP 调用示例（多段结算 / 更正）

以下示例使用初始演示数据（`http://localhost:8080`）。先取当前版本：

```bash
curl -s http://localhost:8080/api/state            # 读取 "version"
```

### 1. 多段照度结算（进行中时段 → 已结束）

把进行中的时段 #2（计划 2026-09-30 09:00–13:00）按三段实测照度结算：
09:00–11:00 为 180 lux（120 分）、11:00–12:30 为 240 lux（90 分）、12:30–13:00 闭灯为 0 lux（30 分）。
曝光 = 120×180 + 90×240 + 30×0 = 43200 lux·min。

```bash
curl -s -X POST http://localhost:8080/api/slots/2/settle \
  -H 'Content-Type: application/json' \
  -d '{
    "version": 1,
    "actualStart": "2026-09-30T09:00",
    "actualEnd":   "2026-09-30T13:00",
    "segments": [
      {"start": "2026-09-30T09:00", "end": "2026-09-30T11:00", "lux": 180},
      {"start": "2026-09-30T11:00", "end": "2026-09-30T12:30", "lux": 240},
      {"start": "2026-09-30T12:30", "end": "2026-09-30T13:00", "lux": 0}
    ]
  }'
```

成功返回完整最新预算（与 `/api/state` 同构），并附带：

```json
{
  "version": 2,
  "justFinishedExposure": 43200,
  "justFinishedMinutes": 240,
  "justFinishedSegments": [ {"start": "...", "end": "...", "lux": 180, "minutes": 120, "exposure": 21600}, "..."],
  "settlement": { "actualStart": "...", "actualEnd": "...", "actualMinutes": 240, "actualExposure": 43200, "segments": ["..."] },
  "blockedAfterFinish": [ {"id": 3, "blockedReason": {"gaps": ["..."]}} ]
}
```

分段有缺口 / 重叠 / 未覆盖 / 负照度时：

```json
HTTP 422
{ "error": {"code": "SEGMENTS_INVALID", "message": "分段结算已拒绝：..."},
  "segmentErrors": ["第 2 段开始 2026-09-30T11:10 必须与第 1 段结束 2026-09-30T11:00 首尾衔接（无重叠、无缺口）"] }
```

### 2. 已结束时段分段更正

对已结束的时段 #1 提交带理由的完整分段更正（起止沿用原实际区间 09:10–11:50）：

```bash
curl -s -X POST http://localhost:8080/api/slots/1/correct \
  -H 'Content-Type: application/json' \
  -d '{
    "version": 2,
    "reason": "照度计复核：布光阶段 100 lux，巡场调亮后 170 lux",
    "segments": [
      {"start": "2026-09-29T09:10", "end": "2026-09-29T10:30", "lux": 100},
      {"start": "2026-09-29T10:30", "end": "2026-09-29T11:50", "lux": 170}
    ]
  }'
```

返回最新预算与本次更正记录（`correction.before/after` 含更正前后曝光、分钟数与完整分段），
时段的 `corrections` 历史追加一条；作品已用量、后续时段 `blocked` 与缺口已按新总量重算。

错误情况：

- 分段未完整覆盖原实际区间 / 衔接失败 / 负照度 → `422 SEGMENTS_INVALID`（`segmentErrors` 逐条说明）；
- 缺少理由 → `422 VALIDATION`；
- 时段不是 `finished` → `422 NOT_FINISHED`；
- 版本过期 → `409 VERSION_STALE`，响应带 `state`（最新完整预算）；用其中的 `version` 重试即可。

### 3. 单一照度结算（原接口，语义不变）

```bash
curl -s -X POST http://localhost:8080/api/slots/2/finish \
  -H 'Content-Type: application/json' \
  -d '{"actualStart": "2026-09-30T09:00", "actualEnd": "2026-09-30T13:00", "actualLux": 300}'
```

> `finish` 仍为原有无版本、单照度结算与返回语义（`justFinishedExposure`、`blockedAfterFinish`）；
> 内部以单段形式保存，因此该时段之后同样可以用 `/correct` 做多段更正。

### 4. 共享额度组（新建 / 调整 / 删除）

演示数据已自带组「纸本/水彩易损作品共享额度」（成员：山水轴 #1、星夜手稿 #2，上限 260000）。

新建额度组（非负上限 + 成员编号数组；每件作品至多一组，成员可留空）：

```bash
curl -s -X POST http://localhost:8080/api/groups \
  -H 'Content-Type: application/json' \
  -d '{
    "version": 1,
    "name": "水彩/纸本联合共享额度",
    "limit": 150000,
    "note": "巡展 B 厅共用",
    "members": [1, 2]
  }'
```

成员重复、不存在或已归属其他组时返回 `422 GROUP_INVALID`（示例中 #1、#2 已在初始组里）：

```json
HTTP 422
{ "error": {"code": "GROUP_INVALID", "message": "额度组保存已拒绝：成员存在重复、不存在或已归属其他组"},
  "memberErrors": [
    "第 1 个成员：作品「《山水轴》纸本水墨」(#1) 已归属额度组「纸本/水彩易损作品共享额度」，每件作品至多归属一组"
  ] }
```

调整组上限/成员（整组替换成员；成功即按现有时序重算余额与不可启动时段）。
下例把初始组上限调到 **50000**（低于成员既有已用之和 32000+ 已结束实际，组已用量约 54400），
历史与进行中时段保留，仅把后续未开始时段标为不可启动：

```bash
curl -s -X PUT http://localhost:8080/api/groups/1 \
  -H 'Content-Type: application/json' \
  -d '{"version": 1, "name": "纸本/水彩易损作品共享额度", "limit": 50000, "members": [1, 2]}'
```

返回的最新预算中：

```json
{
  "version": 2,
  "groups": [ { "id": 1, "name": "纸本/水彩易损作品共享额度",
                "limit": 50000, "used": 54400, "balance": -149000, "over": true, "members": [ "…" ] } ],
  "slots": [
    { "id": 2, "status": "running", "blocked": false },
    { "id": 3, "status": "scheduled", "blocked": true,
      "blockedReason": { "gaps": [], "groupGaps": [
        { "groupId": 1, "groupName": "纸本/水彩易损作品共享额度",
          "has": -100400, "need": 21600, "gap": 122000,
          "contributions": [ {"artworkId": 1, "artworkName": "《山水轴》纸本水墨", "need": 21600} ] } ] } }
  ]
}
```

此时启动 #3 返回 `422 BLOCKED`（`blockedReason` 同上）；把上限改回充足值（携带新版本号）后缺口消除、可正常启动。

整批排期若使组预占超限，返回 `422 BATCH_REJECTED` 并在 `groupBudgetExceeded` 中指出组、每个成员贡献与超额量：

```json
{ "error": {"code": "BATCH_REJECTED", "message": "整批已拒绝：有作品或额度组预占后将超过上限"},
  "groupBudgetExceeded": [
    { "scope": "group", "groupId": 1, "groupName": "纸本/水彩易损作品共享额度",
      "limit": 180000, "used": 150400, "scheduledNeed": 48000,
      "projected": 198400, "overflow": 18400,
      "contributions": [
        { "artworkId": 1, "artworkName": "《山水轴》纸本水墨", "used": 82400, "scheduledNeed": 24000, "projected": 106400 },
        { "artworkId": 2, "artworkName": "《星夜手稿》水彩",   "used": 68000, "scheduledNeed": 24000, "projected": 92000 }
      ] }
  ] }
```

删除额度组（携带版本；成员解除归属，单件预算与时段历史保留）：

```bash
curl -s -X DELETE http://localhost:8080/api/groups/1 \
  -H 'Content-Type: application/json' -d '{"version": 2}'
```

错误情况：名称为空/重复、上限为负、成员非法 → `422 VALIDATION` 或 `422 GROUP_INVALID`（`memberErrors`）；
组不存在（更新/删除）→ `404 NOT_FOUND`；版本过期 → `409 VERSION_STALE` 并带最新 `state`；
任何失败均不推进版本、不改变成员归属。

## 页面用法

- **共享额度组**：顶部「共享曝光额度组」卡片可新建 / 编辑 / 删除额度组，维护名称、非负上限与成员（多选，已属其他组的作品不可选），实时预览成员已用+预占与保存后组余额；组余额为负时红色显示缺口。
- **作品归属**：作品表新增「所属额度组」列；阻断提示同时区分单件缺口与额度组缺口（含本时段逐件贡献）。
- **结束进行中时段**：在「进行中 / 已结束时段」点击 **结束补记**，弹窗内可切换
  「单一照度结算 / 多段照度结算」；多段模式下逐段填写起止与照度，弹窗实时显示各段时长/曝光与总量，
  提交时服务端再次校验衔接与覆盖。
- **查看分段与更正历史**：已结束时段点击 **分段/更正** 展开，展示实际照度分段与历次更正
  （理由、更正前后曝光与时间）。
- **更正已结束时段**：点击 **分段更正**，弹窗预填当前分段并显示原实际区间与更正前曝光，
  调整分段、填写理由后提交；成功后页面刷新作品已用量、缺口与不可启动标记。

## 快速体验超量拦截

初始演示数据以 2026-09-30 为当天，预算均充足。进入页面：

1. 对「一号厅 09:00–13:00（进行中）」点 **结束补记**；
2. 把实际照度改为 `300`（240 分钟 × 300 = 72000 lux·min），保存；
3. 后续二号厅时段会被标记为 **不可启动**，页面列出《山水轴》《星夜手稿》的缺口，启动按钮被禁止；
4. 在未开始排期草稿中缩短时长 / 降低照度 / 改展作品后 **整批校验并保存**，缺口消除即可重新启动。

体验共享额度组：在顶部「共享曝光额度组」卡片编辑初始组，把上限改到成员已用+预占之下（例如 `50000`），
保存后组余额变红、后续未开始时段按顺序标出 **额度组缺口（含逐件贡献）** 并禁止启动；进行中时段保持运行；
把上限改回充足值（携带新版本号）后恢复。也可新建只含部分作品的组，验证“每件作品至多一组”的成员拒绝与整批排期的组超限明细。
