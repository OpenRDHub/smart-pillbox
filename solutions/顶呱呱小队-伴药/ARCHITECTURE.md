# 醒伴 CareBox 架构设计

> 版本：V1.0｜日期：2026-10-16｜阶段：比赛样机联调与演示

## 1. 架构结论

醒伴采用“患者端微信小程序 + 家属端微信小程序 + 云端业务服务 + Device Gateway + ESP32-S3 日仓/家庭主体”的端云一体架构。

核心原则：

1. 日仓在时间、计划、供电和输出资格有效时本地提醒，不依赖手机在线。
2. 设备先产生不可变原始事实；业务层再基于规则生成开合推定、超时和风险状态。
3. 事件身份由 `producer + device_id + boot_epoch + seq` 组成，内容摘要用于冲突识别。
4. 云端先落库再回执；丢确认时重发原事件，不新造事件。
5. 设备身份、生产者、槽位、共享 4G 附件和会话身份必须分离。
6. 任何模拟能力必须带 `demo_simulated`，不可伪装成真实设备回执。

## 2. 总体架构

```text
患者端小程序                         家属端小程序
  ├─今日计划/倒计时                     ├─远程守护/今日进度
  ├─用药时间线                          ├─漏服升级/电话确认
  ├─设备与日仓                          ├─重复取药/风险事件
  └─设置与隐私                          └─家属权限/通知回执
           │ HTTPS / WebSocket / 订阅消息（按实际渠道能力）
           ▼
 API Gateway / 鉴权 / 幂等事件入口
           │
 业务服务：账户家庭、计划批次、提醒状态机、事件归档、库存估算、通知编排
           │
 PostgreSQL/Supabase + 对象/审计存储
           │
 Device Gateway：BLE 控制、POD UART、HTTPS 上行、4G 会话
       ┌───┴──────────────────────┐
 日仓 A / 日仓 B                  家庭主体
 ESP32-S3 + RTC + LID             ESP32-S3 + 屏幕 + LED + PCM5102/AUX
```

## 3. 技术栈选择及原因

| 层    | 比赛样机选择                                | 原因                   | 赛后升级                       |
| ---- | ------------------------------------- | -------------------- | -------------------------- |
| 双端   | 微信小程序原生或 Taro + TypeScript            | 适配患者/家属已有微信入口，减少安装成本 | 跨端壳或原生 App                 |
| 前端状态 | TanStack Query + Zustand/小程序 Store    | 服务端状态与本地演示/离线状态分离    | 统一状态与组件平台                  |
| UI   | 小程序组件、React、Design Tokens             | 快速演示并统一双端视觉口径        | Storybook、统一组件库            |
| API  | Node.js + TypeScript + NestJS/Fastify | 类型安全、适合快速构建模块化 API   | 设备、通知、账户服务拆分               |
| 数据库  | PostgreSQL / Supabase                 | 关系、事务、RLS、审计与快速验证    | 云 RDS、读写分离、归档库             |
| 实时   | Supabase Realtime / WebSocket         | 双端接收时间线和状态变化         | 独立 WebSocket Gateway + MQ  |
| 设备接入 | BLE GATT、POD UART、HTTPS、共享 4G         | 与现有硬件/固件事实一致         | Device Gateway；MQTT 仅评估后引入 |
| 固件   | ESP-IDF 5.4.2、paired-r7               | 与当前日仓和主体基线一致         | 量产固件、OTA、设备农场              |
| 通知   | 微信订阅消息 + 短信兜底                         | 比赛阶段可快速演示            | 正式供应商、外呼与回执聚合              |
| 测试   | Vitest/Jest、Playwright、真机             | 覆盖状态机、API、双端和真机链路    | 契约、混沌、离线、设备农场              |
| 部署   | Supabase Edge Functions / Node + CDN  | 低运维、适合样机阶段           | 容器化、灰度、可观测性                |

现阶段网络实现按 HTTPS 描述；不得将 MQTT 写成已实现能力。

## 4. 项目目录结构

```text
carebox/
├── apps/
│   ├── patient-miniapp/
│   │   ├── pages/             # 首页、今日计划、记录、设备、计划、我的
│   │   ├── components/        # 倒计时、时间线、设备状态、风险提示
│   │   ├── stores/            # 页面本地状态与隐私设置
│   │   └── services/          # API、BLE 会话、同步
│   └── family-miniapp/
│       ├── pages/             # 守护首页、动态、事件详情、成员
│       ├── components/        # 新鲜度、来源、回执、电话确认
│       ├── stores/
│       └── services/
├── services/
│   ├── api/
│   │   ├── modules/auth-family/
│   │   ├── modules/plans-batches/
│   │   ├── modules/occurrences/
│   │   ├── modules/events/
│   │   ├── modules/notifications/
│   │   └── modules/audit/
│   ├── device-gateway/
│   │   ├── ble/
│   │   ├── pod-uart/
│   │   ├── https-ingress/
│   │   └── shared-4g/
│   └── notification-worker/
├── firmware/
│   ├── daybox/
│   │   ├── boot-recovery/
│   │   ├── plan-manager/
│   │   ├── fill-batch-manager/
│   │   ├── clock-manager/
│   │   ├── lid-tracker/
│   │   ├── reminder-scheduler/
│   │   ├── evidence-store/
│   │   ├── sync-manager/
│   │   ├── output-manager/
│   │   └── diagnostics/
│   └── base/
├── packages/
│   ├── contracts/             # API、typed codec、事件合同
│   ├── domain/                # 状态机、窗口、安全间隔、幂等
│   ├── ui/                    # Design Tokens 与共享组件
│   └── test-fixtures/         # Demo 与验收夹具
├── database/
│   ├── migrations/
│   ├── schema/
│   └── policies/
├── tests/
│   ├── unit/
│   ├── contract/
│   ├── e2e/
│   ├── device/
│   └── acceptance/
├── README.md
├── PRD.md
└── ARCHITECTURE.md
```

## 5. 核心模块说明

### 5.1 前端模块

- `PatientApp`：患者低打扰自我管理。
- `FamilyApp`：家属远程守护、风险处理和权限。
- `ServerState`：`family`、`patient`、`todayOccurrences`、`deviceSnapshot`、`timelineEvents`、`notificationReceipts`。
- `LocalState`：`currentTab`、`demoScenario`、`pendingBleSession`、`offlineOutbox`、`privacySettings`。
- 所有状态更新带 `updated_at`、`source`、`freshness`；按钮不能直接写“已服”。

### 5.2 云端模块

- `AuthFamily`：微信登录、家庭、成员角色、邀请和撤权。
- `PlanBatch`：计划草稿、版本、整仓换装和批次采用。
- `OccurrenceEngine`：提醒实例、窗口、重提、超时、迟到开盖和安全间隔。
- `EventIngest`：身份验证、规范化、摘要、幂等、冲突隔离、精确回执。
- `NotificationOrchestrator`：订阅消息、短信、电话任务与渠道回执。
- `Audit`：权限、查看、处理、设备命令和数据变更审计。
- `Freshness/Risk`：最后同步、离线、数据过期、通知未达、重复取药风险。

### 5.3 设备与固件模块

日仓固件：

- `Boot/Recovery`：恢复 `boot_epoch`，检查 NVS/Flash 完整性、RTC 和外设资格。
- `PlanManager`：计划版本、有效期、采用状态。
- `FillBatchManager`：维护、批次、四格数量与核对。
- `ClockManager`：校时、时间质量和 `ClockHold`。
- `LidTracker`：五触点去抖、轨迹、稳定档、异常。
- `ReminderScheduler`：提醒实例、重提和升级历史。
- `EvidenceStore`：原始事件、摘要、离线队列。
- `SyncManager`：BLE、POD、WiFi 候选、4G 和 HTTPS。
- `OutputManager`：蜂鸣、振动、屏幕、灯和音箱。
- `Diagnostics`：STATUS、STACK、BLE、CFG、TRACE。

家庭主体：供电、POD 通信、屏幕、两颗 WS2812B-V6 和 PCM5102/AUX 声光增强；两颗 LED 不能映射为七格独立灯控。

## 6. 数据模型设计

### 6.1 领域对象

```ts
interface DeviceIdentity {
  deviceId: string;
  producer: "day" | "base";
  slot?: number;
  accessoryId?: string;
  sessionId?: string;
}

interface Occurrence {
  id: string;
  planVersion: string;
  fillBatchId: string;
  targetCell: 1 | 2 | 3 | 4;
  dueAt: string;
  expiresAt?: string;
  status: "scheduled" | "due" | "reminding" | "completed_inferred" | "timeout_pending_review" | "missed_recorded";
  source: "device_inferred" | "patient_reported" | "family_confirmed" | "demo_simulated";
}

interface DeviceEvent {
  producer: string;
  deviceId: string;
  bootEpoch: string;
  seq: string;
  type: "lid_transition" | "lid_trajectory" | "reminder" | "sync" | "diagnostic";
  occurredAt: string | null;
  monotonicMs: number;
  timeQuality: "trusted" | "stale" | "unknown" | "invalid";
  evidence: Record<string, unknown>;
  digest: string;
}
```

### 6.2 核心数据库表

| 表                                             | 用途            | 关键约束        |
| --------------------------------------------- | ------------- | ----------- |
| `accounts`                                    | 账户与登录身份       | 最小化身份资料     |
| `families` / `family_members`                 | 家庭与角色         | RLS、邀请、撤权   |
| `patients`                                    | 被守护人          | 健康资料最小化     |
| `devices`                                     | 设备身份、角色、固件、绑定 | 身份与会话分离     |
| `device_sessions`                             | BLE/4G/主体会话   | 连接代次、旧会话失效  |
| `medications`                                 | 人工录入药品资料      | 不宣称自动识别     |
| `medication_plans`                            | 计划草稿、版本、有效期   | 采用状态可回读     |
| `fill_batches` / `fill_cells`                 | 整仓批次、四格数量     | 批次与计划关联     |
| `occurrences`                                 | 提醒实例与业务状态     | 同一实例处理迟到开盖  |
| `device_events`                               | 不可变原始设备事件     | 唯一身份 + 摘要冲突 |
| `event_corrections`                           | 人工纠正追加记录      | 不覆盖原始事实     |
| `notification_jobs` / `notification_receipts` | 通知任务与回执       | 送达、查看、处理分层  |
| `risk_events`                                 | 重复取药、离线、过期等   | 风险与事实分离     |
| `audit_logs`                                  | 权限、查看、命令、变更审计 | 可追溯         |

### 6.3 RLS 与角色

- 患者可读写自己的计划、设置和人工纠正申请。
- 紧急联系人可读授权范围并接收风险通知。
- 仅查看家属只读记录，不接收异常推送。
- 设备写入必须匹配绑定实体、当前会话和合法事件身份。
- 家属不能修改设备原始事件，只能追加确认或纠正。

## 7. 数据流与状态机

### 7.1 正常事件流

1. 设备或服务采用带版本的计划和装药批次。
2. 日仓校验身份、批次、目标格、时间质量和计划。
3. 到点创建持久提醒实例，本地输出并在通信成立时通知主体。
4. `LidTracker` 识别完整轨迹，`EvidenceStore` 保存原始证据。
5. `SyncManager` 以原事件 ID 进入本地或云端队列。
6. 云端验证、规范化、计算摘要、幂等查询、冲突隔离或落库。
7. 云端返回 durable receipt，业务层更新 occurrence，通知层异步处理。
8. 两端只展示有来源、有时间质量和新鲜度的状态。

### 7.2 提醒状态机

```text
scheduled
   ↓ due
reminding ──有效轨迹──> completed_inferred
   │
   ├─有限重提无轨迹──> timeout_pending_review ──升级──> missed_recorded
   │
   └─设备离线/时间未知──> needs_review
```

约束：重提按 `occurrence_id` 工作；迟到开盖更新同一实例；不因午夜自动判漏服；设备失时与提醒超时分开；安全间隔内开盖只生成风险事件。

### 7.3 时间质量

- `trusted`：有效校时且误差可接受。
- `stale`：可保时但校时过久或精度下降。
- `unknown`：无法证明连续性或校时年龄。
- `invalid`：RTC 失败、归零、冻结或明显跳变。

启动、校时、重连、提醒前和 MCU 独立周期都要检查。校时恢复后不重放已完成提醒，也不自动建议补服。

## 8. 接口与事件合同

建议业务 API：

- `GET /v1/devices/:id/state`
- `POST /v1/devices/:id/events`
- `POST /v1/plans`
- `POST /v1/plans/:id/adopt`
- `POST /v1/notifications/dispatch`
- `GET /v1/families/:id/timeline`
- `POST /v1/events/:id/corrections`

这些 REST 路径是业务层建议，不替代现有固件 typed 操作、HTTPS 服务、证书和正式回执材料；联调前必须由后端负责人确认实际路径。

事件原始内容至少包含：事件类型、格位/轨迹、发生 UTC 或 null、单调时间、时间质量、计划批次和证据。发送、接收、中继和重试元数据不得进入原始摘要。

处理规则：

- 同 ID、同规范化内容：幂等返回原内容确认。
- 同 ID、异内容：保留两份，记录摘要冲突，业务待核查。
- 云端先归档后回执，HTTP 200 不等于业务已持久化。
- 丢确认：重发原 ID 和原内容。
- 中继不冒充原生产者。
- `u64` 身份与版本使用十进制字符串；UTC 秒/毫秒不可混用；typed 字节、分片和 SHA 必须按真实合同编码。

## 9. 可靠性、故障与降级

| 故障     | 设备/服务行为             | 用户显示         |
| ------ | ------------------- | ------------ |
| RTC 无效 | `ClockHold`，不运行猜测药程 | 设备时间不可确认，待校时 |
| 无网络    | 本地继续，事件入队           | 设备离线、最后同步时间  |
| 低电量    | 提示低电，保留事实           | 可能影响后续提醒     |
| 日志压力   | 提示并禁止静默覆盖           | 需要回站同步       |
| POD 断链 | 日仓与主体分开             | 日仓可用/主体离线    |
| 音箱断电   | 不伪报已播放              | 外音箱输出待核查     |
| 滑盖错档   | 保留轨迹                | 未形成有效开合推定    |
| 4G 切换  | 停会话、隔离在途数据          | 当前日仓 4G 切换中  |

掉电/重启只恢复完整有效且校验通过的状态；不完整事务进入待核查，不全擦、不重新登记、不用旧镜像掩盖故障。

## 10. 安全与隐私

- BLE 使用安全配对和会话鉴权；旧会话、旧回调和旧权限必须失效。
- 家属角色至少区分紧急联系人与仅查看家属。
- 锁屏通知默认脱敏。
- 诊断材料不得暴露 PIN、密钥、完整 Flash/NVS 或患者身份。
- 原始事实、纠正、通知层级、查看和处理时间进入审计。
- 设备身份与附件身份分离；共享 4G 切换须隔离旧响应。

## 11. 代码规范

- TypeScript 开启严格模式，公共类型集中在 `packages/contracts`。
- 领域状态只能通过状态机/服务方法变更，禁止组件直接修改业务状态。
- 原始设备事件不可变；纠正使用追加表和追加 API。
- 所有写请求带 `client_request_id` 或业务幂等键；重试不得新造事件。
- 时间统一使用 UTC；展示层再按用户时区格式化；明确秒、毫秒和 `monotonic_ms`。
- `u64` 使用十进制字符串，禁止在 JavaScript 中直接用不安全整数承载设备身份。
- API 响应明确 `source`、`freshness`、`needs_review` 和 durable receipt 状态。
- UI 文案优先使用“开合推定”，模拟能力显示“演示模拟”。
- 日志禁止输出密钥、PIN、完整患者身份和完整设备持久区。
- 每个模块至少覆盖正常路径、断链/重试、冲突/异常和权限边界。
- 提交前执行格式化、类型检查、单元测试、契约测试和关键 E2E。

## 12. 测试与验收

### 单元测试

状态机、时间窗口、安全间隔、幂等、冲突、权限和事件规范化。

### 契约测试

typed codec、API 字段、`u64` 字符串、UTC 单位、请求分片、SHA 和 durable receipt。

### E2E 与真机

正常开盖、错档、预开未关、抖动、超时、迟到完成、断电、重启、乱序 ACK、离线补传、BLE 权限、后台、订阅消息、弱网和设备切换。

### 硬件验收

沿用硬件 PRD 的 T01-T20 矩阵，至少覆盖换槽、重启/掉电、换装、无手机无网、输出故障、双仓并发、错档、深开、抖动、迟到完成、冲突、补传、日志压力、校时异常、4G 换仓、电量续航、权限撤回和版本恢复。

蜂鸣/ERM、机械、续航、主体与双仓联调没有同版实测记录前，只能标记为“待实测”或“联调中”。

## 13. 当前风险与升级路线

高风险项：

- 稳定免 PIN 重连和认证/订阅顺序。
- RTC、真实计划、开合推定和取消/恢复闭环。
- 蜂鸣/ERM 实际输出、声音感知和外音箱供电。
- 两只日仓与家庭主体联调。
- 日志容量、掉电事务恢复、机械与 24 小时续航。
- 正式 HTTPS 服务、通知、短信和电话回执。

产品化方向：七只真实电子仓、物理时间锁、人体睡眠/起身检测、声音反馈诊断、药品识别和精确药量传感、免串口配网与换机恢复、家属推送/外呼、单手防撒与长期真实使用验证。

## 14. 版本记录

| 版本   | 日期         | 说明                                  |
| ---- | ---------- | ----------------------------------- |
| V1.0 | 2026-10-16 | 合并硬件 PRD、硬件方案、软件方案和软硬件结合方案，建立统一架构基线 |
