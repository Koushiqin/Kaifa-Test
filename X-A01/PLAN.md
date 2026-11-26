# X-A01 执行方案

- 状态：设计已修订，未执行
- 任务：X-A01 四角色登录、会话与退出
- 工作台 revision：1
- 被测站：https://exam.1mpower.com/（用户确认为非隔离共享环境；隔离边界待预检核实）
- 最后更新：2026-11-26
- 位置说明：本文件是 X-A01 任务执行方案/交付物；规格见 `.scratch/x-a01/spec.md`，具体执行步骤以本文件为准。

## 1. 结论

本轮只执行 X-A01 的 3 个用例。任务级有两个独立缺口：

1. **管理员账号未提供**：如果站点存在管理员角色/入口，管理员断言记 `blocked`；如果站点没有管理员角色/入口，记 `na` 并附证据。具体按预检结果。
2. **候选版本未知**：`build` 写 `unknown`，日期、响应头、静态资源 hash 写 `notes`；即使补管理员，本轮也不能形成可归因的 P0 通过结论。

因此，本轮预期结果是：X-A01-01/02/03 至少为 `blocked` 或 `na`（按判定树）；若实测发现权限泄漏、旧会话仍可用等缺陷，相关用例记 `failed`，优先级高于 `blocked`。只有在管理员角色存在且拿到账号、候选版本可归因、真实 TTL 可验证时，才有可能形成 P0 通过结论。

## 2. 范围与验收

- 范围：只做 X-A01，不扩展到其他 38 项自动任务。
- 用例：
  - X-A01-01 合法登录与默认入口
  - X-A01-02 无效与过期会话
  - X-A01-03 退出后旧请求
- 验收：全角色认证断言 + 会话失效断言通过。
- 时间盒：工作台给 X-A01 的是 15 分钟人工准备与复核；首次自动化执行 + 证据预计 45–70 分钟，这是整轮时间，不是 X-A01 的任务级预算。不要把 45–70 分钟说成“符合 15 分钟”。
- 阻塞边界：缺管理员、真实 TTL 不可验证、助教角色不符、站点异常、写入无法清理、候选版本未知时，按第 6 节记 `blocked`/`na`，或在 `notes` 标明任务级限制。

## 3. 环境与账号

### 3.1 被测站

- URL：https://exam.1mpower.com/
- 非隔离性来源：用户 2026-11-26 明确回答“非隔离环境”；工作台 X-A01 任务卡写的是“隔离测试环境 / 浏览器与接口自动化”，两者冲突。
- 预检记录判定依据：站点是否只有测试账号、是否有真实班级/学生数据、是否有站点所有者说明。
- 在隔离性证据取得前，`environment` 写 `exam.1mpower.com（测试环境，隔离性待核实）`，不直接断言“非隔离”。
- 不做压测、fuzz、越权扫描；只做最小写操作。
- 所有测试账号均来自 [UsableAccounts.md](../UsableAccounts.md)，不接触其他账号。

### 3.2 账号库

所有测试任务共用一套账号，统一放在仓库根目录 `.secrets/accounts.local.json`，并加入 `.gitignore`。

文件结构（只示意键；实际用户名/密码从 [UsableAccounts.md](../UsableAccounts.md) 抄入本机文件，不写入脚本、计划、报告或证据）：

```json
{
  "accounts": {
    "teacher.primary":   { "displayName": "测试账号老师", "username": "...", "password": "...", "expectedRole": "教师" },
    "teacher.backup":    { "displayName": "测试账号老师2", "username": "...", "password": "...", "expectedRole": "教师" },
    "assistant.primary": { "displayName": "测试账号助教", "username": "...", "password": "...", "expectedRole": "助教" },
    "student.primary":   { "displayName": "测试账号学生", "username": "...", "password": "...", "expectedRole": "学生" },
    "student.secondary": { "displayName": "测试账号学生", "username": "...", "password": "...", "expectedRole": "学生" }
  }
}
```

每个账号包含 `displayName`、`username`、`password`、`expectedRole`；脚本按账号键读取，`displayName` 仅用于报告。

- 脚本只接收账号键（如 `student.primary`），不接收明文密码。
- `UsableAccounts.md` 保留为人类可读索引；按 Q2 的选择，保留明文密码，但已加入 `.gitignore`，不提交。实际密码同时抄入本机 `.secrets/accounts.local.json`，供脚本读取。
- `.gitignore` 已创建，包含：

```gitignore
.secrets/
*.local.json
UsableAccounts.md
X-A01/evidence/
node_modules/
.claude/settings.local.json
*.har
*.trace.zip
.scratch/pw/
.scratch/fetch/
```

- 为什么用 JSON 而不是 `.env`：账号需要角色、备注和多账号映射，JSON 的结构更稳定；所有任务的脚本共用同一套账号键。
- 为什么不在脚本里解析 `UsableAccounts.md`：Markdown 表格解析脆弱，且会把密码带进更多处理链路。`UsableAccounts.md` 只做人类索引。

### 3.3 账号分配

| 账号键 | 账号表角色 | 本轮用途 | 备注 |
|---|---|---|---|
| `teacher.primary` | 教师 | 教师登录、默认入口、退出重放 | 主教师 |
| `teacher.backup` | 教师 | 备用，不默认使用 | 防锁定/对比 |
| `assistant.primary` | 助教 | 助教登录、实际角色核对 | 可能实际不是助教 |
| `student.primary` | 学生 | 学生登录、合法读写、退出重放 | 主学生 |
| `student.secondary` | 学生 | 错误凭据测试 | 只试 1 次错误密码 |
| 无 | 管理员 | 预检确认角色/入口是否存在 | 存在但无账号 → `blocked`；不存在 → `na` |

### 3.4 操作边界

- 每个角色独立 browser context，不共享 cookie/storage。
- 不访问其他账号数据；不把作业发布给真实学生。
- 错误密码只试 1 次，优先 `student.secondary`，避免锁定账号。
- 结果先本地；不导入工作台，除非用户明确授权。
- 发现站点异常、账号锁定或数据风险，立即停止并记 `blocked`。

## 4. 领域模型

已按 grill-with-docs 的要求落到 [GLOSSARY.md](../GLOSSARY.md)。本轮统一使用：任务、用例、运行、结果、候选版本、预期版本、阻塞、不适用、证据、账号、角色、账号库。

关键区分：

- `build` 是候选版本；`expectedRevision=1` 是工作台任务记录的版本，两者不是一回事。
- 账号 ≠ 角色；`assistant.primary` 登录后实际角色可能不是助教，必须以站点返回或界面为准。
- 管理员“账号缺失” ≠ “角色/入口不存在”：前者 `blocked`，后者 `na`（附证据）。
- `blocked` ≠ `failed` ≠ `na`；缺账号只能记 `blocked`，产品不提供角色/能力才可记 `na`。
- 一个用例内只要关键断言没验证完，整个用例记 `blocked`，即使其他断言已经通过。

## 5. 测试设计

### 5.1 X-A01-01 合法登录与默认入口

**准备**：教师、助教、学生各一个 fresh context；管理员按预检结果处理。

**步骤**：

1. 打开登录页，记录标题、语言、表单字段。
2. 分别用 `teacher.primary`、`assistant.primary`、`student.primary` 登录。
3. 预检确认实际身份接口：预检已记录 `GET /api/v1/me` 未认证返回 401（2026-11-26 16:48:58 +08:00，`UNAUTHENTICATED`，本地预检证据）；登录后返回结构待确认。预检用带超时的 fetch 直接调用；页面自身的请求在 headless 下可能挂起，不作为证据。若不可用，改用会话 claims + 页面身份元素交叉验证。
4. 每个角色记录：登录后 URL、默认落地页、可见导航项、身份/角色。
5. 跨角色对比：学生是否看到教师/管理入口；教师是否看到管理入口。助教入口与教师/学生的差异只记录为观察项，不判失败。
6. 管理员：预检确认站点是否存在管理员角色/入口。存在但无账号 → `blocked`；不存在 → `na` 并附 UI/接口证据。不猜账号、不尝试登录。

**断言**：

- 登录成功，无 5xx。
- 身份/角色与账号实际角色一致；记录与账号表的差异。
- 默认落地页是该角色的合法入口，不是“无权限/未配置”空页。
- 低权限角色看不到高权限入口；教师看不到管理入口。
- “角色单列阻塞”操作化定义为：该角色默认落地页不是空壳/无权限页，且能看到自己的核心入口；无法判定时记录观察或 `blocked`，不硬判失败。
- 管理员状态按第 6 节判定树。

**证据**：`01-login-<role>.png`、`01-identity-<role>.json`（脱敏）、`01-nav-matrix.json`。

**限制**：工作台引用了 `docs/95-assistant-role-and-shared-work.md` 四端权限矩阵，但该 GitHub 链接当前 404；A-A01 前置条件摘要可作为唯一可用基线：同机构全部学生与作业、不限发布人；允许发布/查看/修改/撤回/结束同机构作业、上传批改人工确认；账号创建/角色/密码仅管理员。X-A01 本身没有逐入口矩阵；没有基线的入口只记录，不判失败。

### 5.2 X-A01-02 无效与过期会话

**预检**：枚举会话载体：cookies、`localStorage`、`sessionStorage`、IndexedDB（如有）、`Authorization` 头；确认哪个是实际认证载体。

**子场景**：

1. **错误凭据**：fresh context，`student.secondary` + 错误密码，只试 1 次。断言反馈通用、不泄露账号是否存在、不下发有效会话。
2. **无会话**：fresh context 直接访问受保护页面/接口 + 身份接口。API 断言 401/403；页面断言 302 跳登录或登录页；不返回业务数据。
3. **过期会话**：X-A01-02 开始时先登录一次，记录 cookie `exp`/JWT `exp` 和所有会话载体，然后去做错误凭据、无会话、其他角色等子场景；到时间再回来验证过期。等待预算 5 分钟；TTL > 5 分钟则篡改/清空会话载体模拟无效，明确标注“模拟无效，不是真实过期”；服务端无可观测 TTL 则记 `blocked`。
4. 至少覆盖学生 + 教师；助教实际角色有效时覆盖；管理员按第 6 节判定树。

**断言**：

- 错误凭据被拒，反馈可理解，不泄露账号是否存在，无敏感数据，不下发有效会话。
- 无会话访问被拒，不返回业务数据。
- 过期/无效会话被拒，不返回业务数据。
- 不要求“没有任何新 cookie”：匿名会话 cookie、CSRF cookie 是正常的；核心是“没有有效认证会话”。
- 页面请求允许 302 跳登录；API 请求允许 401/403。
- 真实 TTL 是否验证，单独在 note 标明。

**证据**：`02-wrong-creds.*`、`02-no-session.*`、`02-expiry-<role>.*`、`02-session-carriers.json`。

### 5.3 X-A01-03 退出后旧请求

**预检（合法读写前置）**：

1. 优先由 `teacher.primary` 给 `student.primary` 新建一个最小作业/草稿（只发给测试学生），记录 ID；或使用教师账号自己的未发布草稿。
2. 只有在确认现有作业属于测试教师/测试班级且可安全修改时，才使用 `student.primary` 已有的作业/草稿；不碰真实教师/班级数据。
3. 选择天然幂等的写：带客户端 ID 的草稿保存、无变化更新，优先 PUT/PATCH；避免每次 POST 新建。
4. 结束后清理；清理不了就在 `notes` 写明“未清理”，并评估是否记 `blocked`。

**步骤**：

1. 用 `student.primary` 登录，做一个合法读（如作业列表）和一个最小写（保存草稿）。
2. 记录请求方法/路径/必要头/幂等 body、响应、资源 ID、所有会话载体（cookie、localStorage、sessionStorage、IndexedDB、Authorization）。
3. UI 退出；记录退出响应与各载体变化。
4. 分别用每种旧载体重放读请求；API 断言 401/403，页面断言 302 跳登录或登录页；核心断言“无业务数据”。
5. 分别用每种旧载体重放写请求（同一幂等 body）；断言 401/403，数据无变化。
6. 退出后先检查旧页面是否仍显示数据；再刷新，断言不能继续操作、不残留可操作身份。如果退出后旧页面仍显示业务数据，单独记为残留数据风险。
7. 用新登录核对草稿状态；必要时清理。
8. 教师重复一次读/写重放；助教按实际角色可选；管理员按第 6 节判定树。

**断言**：

- 没有有效认证会话，没有业务数据，没有状态变化。
- 旧读/旧写仍成功 → `failed`（会话未失效，安全缺陷），保留证据。
- 旧页面仍显示数据，或刷新后仍可操作 → `failed`（残留可操作身份/数据）。
- 如果写请求无法安全重放 → 该子断言 `blocked`，note 说明。

**证据**：`03-before-logout-<role>.json`、`03-old-session-replay.json`、`03-old-page-before-refresh.png`、`03-old-page-after-refresh.png`、`03-data-state.json`。

## 6. 判定口径

判定优先级：`failed` > `blocked` > `passed`。

- `failed`：至少一条已实际验证的断言不通过。
- `blocked`：没有失败断言，但关键断言因缺账号/环境/前置条件无法验证。
- `passed`：全部关键断言都已实际验证并通过。
- `na`：产品明确不提供该角色或能力，且测试计划允许；不能用于“账号缺失但角色存在”。
- `pending`：只存在于本地草稿；导入前必须改成实际状态，未执行用例按 `blocked` 记录。

**管理员/助教判定树**：

| 场景 | 用例状态 | 说明 |
|---|---|---|
| 管理员账号缺失，但站点存在管理员角色/入口 | `blocked` | 附“未提供管理员账号” |
| 站点没有管理员角色/入口 | `na` | 附 UI/接口证据，说明产品缺口 |
| 助教账号实际不是助教 | 先记录实际角色 | 再按“角色是否存在”判 `blocked`/`na` |
| 产品明确不提供某能力，且任务允许 | `na` | 不能用 `na` 掩盖没测到 |
| 候选版本未知 | 不直接改用例状态；`notes` 标明 | 即使断言通过，也不能形成可归因的 P0 通过结论 |
| 环境隔离性未确认 | 不直接改用例状态；`notes` 标明 | 不能作为隔离环境验收 |
| 旧读/旧写仍成功 | `failed` | 会话未失效，保留证据 |
| 旧页面仍显示数据，或刷新后仍可操作 | `failed` | 残留可操作身份/数据 |
| 写重放无法安全执行 | `blocked` | `note` 说明无法重放的原因 |
| 无真实 TTL 观测，仅模拟无效 | `blocked` | `note` 写“模拟无效，不是真实过期” |
| 错误凭据返回有效会话 | `failed` | 认证缺陷 |
| 无会话返回业务数据 | `failed` | 授权缺陷 |
| 错误凭据泄露账号是否存在 | `failed` | 用户枚举，安全缺陷 |

**预期结果**：

| 用例 | 可验证部分 | 主要缺口 | 预期状态 |
|---|---|---|---|
| X-A01-01 | 教师/学生/助教登录、身份、默认入口、导航对比 | 管理员账号/角色 | `blocked`/`na` 取决于预检；若发现权限泄漏则 `failed` |
| X-A01-02 | 错误凭据、无会话；过期可能只能模拟 | 管理员 + 真实 TTL | `blocked`；若拒绝失效则 `failed` |
| X-A01-03 | 学生/教师退出后读/写重放、旧页面刷新 | 管理员 + 合法读写前置 | `blocked`；若旧请求成功则 `failed` |
| 任务级 | 3 个用例 | 管理员 + 候选版本未知 + 隔离性待核实 | 不能形成可归因的 P0 通过结论 |

## 7. 结果文件

`X-A01/expected.json` 字段：

| 字段 | 值 |
|---|---|
| `schemaVersion` | `1` |
| `taskId` | `X-A01` |
| `runId` | `X-A01-20261126-<HHmmss>` |
| `expectedRevision` | `1`（导入前用浏览器上下文重读 `/api/records`，变了以最新为准） |
| `environment` | `exam.1mpower.com（测试环境，隔离性待核实）` |
| `build` | 用户提供的 commit；没有则 `unknown` |
| `startedAt` / `finishedAt` | ISO 含 `+08:00` |
| `measurementScope` | `functional` |
| `metrics` | `{}` |
| `caseResults` | 3 条，状态用 `passed` / `failed` / `blocked` / `na` |
| `thresholds` | `[]` |
| `artifacts` | `[]`（证据路径写 `notes`） |
| `notes` | 候选版本未知、环境隔离性待核实、账号范围、方法、证据索引、阻塞原因 |

`build` 写 `unknown`，不把日期放进版本字段；`notes` 记录 `buildDate=2026-11-26`、响应头、静态资源 hash 等可观察指纹，不冒充 commit。

**导入方式**：只能通过工作台 UI 或受支持的站点访问；不要用 curl 直接 POST `/api/results`（Cloudflare 会 403）。导入前确认当前工作台身份是测试员 B、任务已领取/进行中，并重读 `expectedRevision`。

## 8. 证据与脱敏

```
X-A01/
  expected.json
  report.md
  evidence/<runId>/
    01-login-<role>.png
    01-identity-<role>.json
    01-nav-matrix.json
    02-wrong-creds.json
    02-no-session.json
    02-expiry-<role>.json
    02-session-carriers.json
    03-before-logout-<role>.json
    03-old-session-replay.json
    03-old-page-before-refresh.png
    03-old-page-after-refresh.png
    03-data-state.json
    manifest.json
```

- 证据粒度：截图 + 脱敏请求/响应摘要；完整 HAR 如需要只放 `.secrets/raw-evidence/`（已被 `.secrets/` 忽略），不进入 `.scratch/` 或交付物。
- 脱敏：`password`、`cookie`、`set-cookie`、`authorization`、`token`、`session`、`csrf`、`username`、`displayName`、`studentId`、`email`、`phone`、班级/学校名称的值全部移除或打码；只保留字段名、状态码和属性。
- 截图对姓名、用户名、头像打码；JSON 只保留字段名、状态码和结论。
- 证据只存本地，不上传外部；`artifacts` 留空，路径写 `notes`。
- `manifest.json` 记录每个证据文件的时间、URL、状态码和 SHA256。

## 9. 执行流程与时间盒

| 阶段 | 时间 | 内容 |
|---|---|---|
| 预检 | 10 min | 站点可达、登录页、环境指纹、隔离性证据、工作台负责人/领取状态、revision、会话载体、身份接口、合法读写对象 |
| X-A01-01 | 15 min | 三角色登录、身份、默认入口、导航矩阵、管理员角色/入口预检 |
| X-A01-02 | 15 min | 先登录记 exp，再做错误凭据/无会话/其他角色，最后回来验过期（≤5 min 等待） |
| X-A01-03 | 15 min | 合法读写、退出、各载体旧请求重放、旧页面刷新、数据核对 |
| 证据与结果 | 10 min | 脱敏、写 `expected.json`、报告 |
| 复核 | 5 min | 自查断言与证据，交给用户审核 |

**停止条件**：站点不可达、连续 5xx、账号锁定、反爬挑战无法通过、出现影响他人数据的风险、写入无法清理。

**开跑前检查清单**：

- [ ] 向用户确认候选版本 commit；拿不到就 `build: "unknown"`，指纹写 `notes`。
- [ ] 确认站点隔离性，记录判定依据；`environment` 写“隔离性待核实”。
- [ ] 确认工作台负责人是测试员 B，任务已领取/进行中；不是则先协调。
- [ ] 用浏览器上下文重读 `/api/records`，确认 `expectedRevision` 仍为 1。
- [ ] 预检会话载体：cookie、`localStorage`、`sessionStorage`、IndexedDB、`Authorization`。
- [ ] 预检真实身份接口路径，用带超时的 fetch 直接调用，并记录未认证 401 的 URL/status/时间；找不到则用 claims + UI 交叉验证。
- [ ] 准备测试学生的合法作业/草稿，记录 ID 和清理方式。
- [ ] 确认管理员角色/入口是否存在；无法获取账号时按判定树记 `blocked`/`na`。
- [ ] 确认导入方式：工作台 UI 或申请机器访问；不直接 curl `/api/results`。
- [ ] 更新脱敏清单，补 PII 和用户枚举断言。
- [ ] 把“助教导航差异”等额外断言降级为观察项。

## 10. 交付物

- `X-A01/expected.json`（本地结果）
- `X-A01/report.md`（中文报告）
- `X-A01/evidence/<runId>/`（脱敏证据）
- `X-A01/run-x-a01.mjs`（方案确认后再写）
- `GLOSSARY.md`、`docs/adr/0001-x-a01-nonisolated-blocked.md`
- 不提交 git，不导入工作台，除非用户明确要求/授权。

## 11. 风险与应对

| 风险 | 应对 |
|---|---|
| 候选版本未知 | `build` 写 `unknown`，日期/指纹写 `notes`；任务级标明不能形成可归因的 P0 通过 |
| 环境隔离性冲突 | 记录用户确认与工作台任务卡冲突；预检记录证据；`environment` 写“隔离性待核实” |
| 非隔离环境，可能碰到真实数据 | 只用测试账号自己的数据；最小写；结束后清理；不压测 |
| 错误密码导致账号锁定 | 只试 1 次，用 `student.secondary`；随后验证正常登录 |
| 会话载体不是 cookie | 预检枚举 cookie/localStorage/sessionStorage/IndexedDB/Authorization；按实际载体重放 |
| 会话 TTL 太长，真实过期无法验证 | 先记 exp，5 分钟预算内就等；否则模拟无效并记 `blocked`，不冒充真实过期 |
| 管理员账号缺失 vs 角色不存在 | 按判定树：角色存在但缺账号 → `blocked`；角色不存在 → `na` 附证据 |
| `assistant.primary` 实际不是助教 | 记录实际角色；相关断言按角色是否存在判 `blocked`/`na` |
| 没有合法读写对象 | 优先由测试教师给测试学生新建最小作业/草稿；只有确认已有数据属于测试教师/测试班级且可安全修改时才使用；仍不可用则写重放记 `blocked` |
| 反爬/Cloudflare 挑战 | 真实 Chrome、降低操作频率；持续失败记 `blocked` |
| HAR/证据泄露 | 原始 HAR 只放 `.secrets/raw-evidence/`；`.gitignore` 加 `*.har`、`*.trace.zip`；交付物只放脱敏摘要 |
| 写操作无法清理 | 保留并写明“未清理”，不做更大改动 |

## 12. 决策记录

- 候选版本：`build` 写 `unknown`；日期、响应头、静态资源 hash 写 `notes`；候选版本未知是任务级限制，不直接改用例状态。
- 环境：用户确认非隔离，但工作台任务卡写“隔离测试环境”；预检记录隔离性证据，`environment` 写“测试环境，隔离性待核实”。
- 管理员/助教：账号缺失但角色存在 → `blocked`；角色/入口不存在 → `na` 附证据；助教实际角色不符时先记录实际角色。
- 会话载体：预检枚举 cookie/localStorage/sessionStorage/IndexedDB/Authorization；X-A01-03 按每种旧载体重放。
- 真实过期：先记 exp，最多等 5 分钟；超过预算则模拟无效并记 `blocked`。
- 合法读写：优先由测试教师给测试学生新建最小作业/草稿；只有确认已有数据属于测试教师/测试班级且可安全修改时才使用；优先幂等写；结束后清理。
- 证据：截图 + 脱敏摘要；完整 HAR 只放 `.secrets/raw-evidence/`；`.gitignore` 加 `*.har`、`*.trace.zip`、`.scratch/pw/`、`.scratch/fetch/`。
- 工作台：确认负责人/领取状态；导入走 UI，不用 curl 直接 POST；导入前重读 revision。
- 方案位置：`X-A01/PLAN.md` 是任务交付物，不是 `.scratch/` 的 issue spec；`.scratch/x-a01/spec.md` 只放指向它的索引。
