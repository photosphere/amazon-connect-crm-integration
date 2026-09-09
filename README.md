# Amazon CRM · Amazon Connect CCP 集成示例

两个单文件（HTML + Tailwind CDN + amazon-connect-streams CDN）的 dummy CRM，演示如何把 Amazon Connect 的 CCP 嵌进自建 CRM，并把来电/进线转成弹屏和工单。

| 文件 | 定位 | CCP 呈现方式 |
| --- | --- | --- |
| `index.html` | 方案 A · 停靠式 CCP | 完整 CCP 面板停靠在右侧整列，主内容自动让位（三栏布局），不遮挡工单与客户资料；窄屏退回悬浮面板 |
| `index_pro.html` | 方案 B · 紧凑通话条 | 来电只显示顶部一条通话条（接听/拒接/静音/保持/挂断由 Streams API 驱动），完整 CCP 按需展开（转接、拨号盘、Chat） |

两个页面共用同一份演示数据与 localStorage 配置，可以来回切换而不用重填 CCP 地址。

- SDK：[amazon-connect-streams](https://github.com/amazon-connect/amazon-connect-streams) `2.18.5`（jsDelivr CDN 引入）

---
#### Login
<img width="1509" height="770" alt="Image" src="https://github.com/user-attachments/assets/6da577e0-d917-4279-984b-dcb58bb47e51" />

#### Inbound Call in Native CCP
<img width="1509" height="850" alt="Image" src="https://github.com/user-attachments/assets/5a12cb3c-b494-4f36-b257-6b31a3ee1023" />

#### Click-to-Call Outbound Call in Native CCP
<img width="1511" height="853" alt="Image" src="https://github.com/user-attachments/assets/a33d6345-6573-4062-8955-ac810c937560" />

#### Inbound Call in Customized CCP
<img width="1508" height="846" alt="Image" src="https://github.com/user-attachments/assets/6ac445b8-fb6f-4ed3-ad52-d3d1ed46f6a4" />

#### Click-to-Call Outbound Call in Customized CCP
<img width="1505" height="847" alt="Image" src="https://github.com/user-attachments/assets/11ff53a2-b542-4b66-8a92-0e6c46a31e43" />

---

## 1. 前置条件

### 1.1 Amazon Connect 实例侧

| 项 | 说明 |
| --- | --- |
| CCP 地址 | `https://<实例别名>.my.connect.aws/ccp-v2/`（旧域名形如 `https://<别名>.awsapps.com/connect/ccp-v2/`）。必须是 `ccp-v2`，Streams 不支持旧版 CCP |
| 区域 | 实例所在区域，例如 `us-west-2`，要与 CCP 地址一致 |
| **Approved origins** | 最关键的一步。把承载页面的来源（例如 `http://localhost:8080`、`https://crm.example.com`）加入实例设置的 *Approved origins*，否则 iframe 会被拒绝，表现为 CCP 一直白屏或反复要求登录 |
| 座席账号 | 一个 Connect 用户，安全配置文件需包含 CCP 访问权限；要点击外呼还需要 **Outbound calls** 权限 |
| 电话类型 | 座席设置里选 **Softphone**（浏览器内接听），本示例的 `allowFramedSoftphone: true` 才有意义 |
| 路由 | 路由配置文件里绑定队列（演示里用 `BasicQueue`）、可用的联系流 |
| 外呼 | 队列上要配置 **Outbound caller ID 号码**；目标国家/地区要在实例的外呼国家允许列表里，否则 `agent.connect()` 会以 failure 回调返回 |
| Chat | 若要演示 Chat 进线，实例需启用 Chat 并有对应联系流 |
| 联系属性（可选） | 联系流里设置 `customerId` / `customer_id` / `crmId` 或 `customerName` 属性，页面会优先用它来定位 CRM 客户 |

### 1.2 浏览器与宿主页面侧

- **必须通过 HTTP(S) 提供页面，不能直接双击用 `file://` 打开**。Streams 依赖真实 origin 做 `postMessage` 与 cookie 校验。本地起服务：
  ```bash
  # 打开 http://localhost:8080/index.html 或 http://localhost:8080/index_pro.html
  ```
- 生产环境必须 HTTPS：麦克风权限、第三方 cookie 都需要安全上下文。
- **允许弹窗**：首次登录会 `window.open` 一个 Connect 登录窗（本示例已把它居中显示，尺寸 480×640）。
- 允许**麦克风**权限，且同一时间只允许**一个页签**初始化 CCP，多个页签会互相抢占软电话会话。
- 桌面版 Chrome / Edge / Firefox。Safari 的第三方 cookie 策略可能导致会话保持不住。
- 页面里 CCP 的 iframe **不能被移动到别的父节点**（reparent 会重载 iframe 并拆掉 `initCCP`）。因此本示例的"停靠"是靠给 `<main>` 预留右边距实现的，面板本身始终 `position: fixed` 原地不动；收起时只用 `visibility/opacity` 隐藏，不用 `display:none`。

### 1.3 本项目自身的配置

登录门禁弹窗需要填 4 项，保存在 localStorage：

| 字段 | 默认值 | 说明 |
| --- | --- | --- |
| CCP 地址 (`ccpUrl`) | `https://connect-us.my.connect.aws/ccp-v2/` | 必填，必须 `https://` |
| 区域 (`region`) | `us-west-2` | 必填 |
| 登录地址 (`loginUrl`) | 空 | 可选，用于 SAML / 自定义 SSO 登录入口 |
| 外呼号码 (`outboundNumber`) | `+13072633584` | 所有演示客户共用的点击外呼号码，校验 E.164 格式 |

| localStorage key | 用途 |
| --- | --- |
| `auroraCrm.ccpConfig.v2` | 上面 4 项配置 |
| `auroraCrm.data.v1` | 工单与来往记录 |
| `auroraCrm.ccpDocked` | CCP 停靠/悬浮偏好 |
| `connectPopupManager::connect::loginPopup` | Streams 自己写入的弹窗去重标记，"重置"按钮会清掉它以便重新拉起登录窗 |

---

## 2. 用到的 SDK 方法与事件

标注 **A** = 仅 `index.html`，**B** = 仅 `index_pro.html`，未标注 = 两个页面都用。

### 2.1 初始化与会话

| API | 作用与用法 |
| --- | --- |
| `connect.core.initCCP(container, config)` | 把 CCP 挂到 `#ccpMount`。本示例传入：`ccpUrl`、`loginUrl`、`loginPopup: true`、`loginPopupAutoClose: true`、`loginOptions`（按屏幕算出的居中 `width/height/top/left` + `autoClose`）、`region`、`softphone: { allowFramedSoftphone, disableRingtone: false, allowFramedVideoCall }`、`pageOptions: { enableAudioDeviceSettings, enablePhoneTypeSettings }`、`ccpAckTimeout / ccpSynTimeout / ccpLoadTimeout` |
| `connect.agent(callback)` | 座席对象就绪回调 → 解锁 CRM、渲染座席信息与状态下拉 |
| `connect.contact(callback)` | 每个新联系的回调 → 在里面订阅联系事件 |
| `connect.core.onAuthFail(cb)` | 会话过期 / 座席在 CCP 内登出 → 重新锁回登录门禁 |
| `connect.core.onAccessDenied(cb)` | 记录到事件日志，通常意味着 origin 未批准或权限不足 |
| `connect.core.terminate()` | 退出登录与"重置"时拆掉 CCP |

配合使用的**非 SDK**手段：退出登录时以 `mode: 'no-cors'` + `credentials: 'include'` 请求实例的 `/connect/logout`（失败则回退到隐藏 iframe 加载），用来丢掉 Connect 会话 cookie，然后 `terminate()` + 刷新页面。

### 2.2 座席（Agent）

| API | 用途 |
| --- | --- |
| `agent.getName()` | 顶栏座席名、工单受理人、历史记录归属 |
| `agent.getRoutingProfile()` | 显示路由配置文件名 |
| `agent.getAgentStates()` | 填充状态下拉（Available / Offline / 自定义 Aux） |
| `agent.getState()` | 当前状态，同步状态指示灯颜色 |
| `agent.setState(state, {success, failure})` | 手动切换状态；退出登录前也用它先置 Offline，停止路由 |
| `agent.onStateChange(cb)` / `agent.onRefresh(cb)` | 状态变化时同步 UI |
| `agent.connect(endpoint, {success, failure})` | 点击外呼 |
| `agent.mute()` / `agent.unmute()` | **B** 通话条的静音按钮 |
| `agent.onMuteToggle(cb)` | **B** 座席在 CCP 内静音时，同步通话条按钮状态 |

### 2.3 联系（Contact）

方法：

| API | 用途 |
| --- | --- |
| `contact.getContactId()` | 联系 ID，作为活动联系与工单的关联键 |
| `contact.getType()` | 配合 `connect.ContactType` 区分 VOICE / CHAT |
| `contact.isInbound()` | 区分进线与外呼（失败时回退用 `getInitialConnection().getType()`） |
| `contact.getAttributes()` | 读联系属性做客户匹配 |
| `contact.getQueue()` | 队列名，写入弹屏与工单 |
| `contact.getConnections()` / `contact.getInitialConnection()` | 取客户号码（ANI），逐个连接找 `getEndpoint().phoneNumber` / `getAddress().phoneNumber` |
| `contact.getAgentConnection()` | **B** 拿座席连接做保持/挂断 |
| `contact.accept({success, failure})` | **B** 通话条接听 |
| `contact.reject({success, failure})` | **B** 通话条拒接（不支持时回退为销毁座席连接） |
| `contact.clear({success, failure})` | **B** 结束 ACW，关闭联系 |

事件：

| 事件 | 页面里的处理 |
| --- | --- |
| `onIncoming` | 进线：建活动联系记录、弹屏、自动建单（A 同时自动展开 CCP；B 只渲染通话条） |
| `onConnecting` | 外呼/进线连接中，同一套处理（已存在的联系会去重） |
| `onAccepted` | 记事件日志 |
| `onConnected` | 记录接通时间、工单状态置"进行中"（B 另外把通话条切到"通话中"） |
| `onMissed` | 归档为"未接" |
| `onACW` | 提示补充工单（B 通话条切到 ACW 并显示"结束 ACW"） |
| `onEnded` / `onDestroy` | 归档到历史时间线、写入通话时长、清理活动联系 |

### 2.4 连接（Connection）

| API | 用途 |
| --- | --- |
| `connection.getType()` | 配合 `connect.ConnectionType.AGENT` 找出座席连接 |
| `connection.getEndpoint()` / `getAddress()` | 取客户电话号码 |
| `connection.isOnHold()` | **B** 通话条实时读取保持状态 |
| `connection.hold()` / `resume()` | **B** 保持 / 恢复 |
| `connection.destroy()` | **B** 挂断（拒接的回退路径也走这里） |

### 2.5 枚举与工厂

`connect.ContactType.VOICE` / `.CHAT`、`connect.ConnectionType.AGENT` / `.INBOUND`、`connect.Endpoint.byPhoneNumber(e164)`。

---

## 3. 支持的场景

| 场景 | 行为 | 依赖 | 页面 |
| --- | --- | --- | --- |
| 登录门禁 | 打开 CRM 即被登录弹窗拦住，填配置 → 拉起居中的 Connect 登录窗 → 登录成功自动解锁 | `initCCP` + `connect.agent` | A / B |
| 会话失效重锁 | 座席在 CCP 内登出或会话过期，CRM 自动锁回门禁并提示重新登录 | `onAuthFail` | A / B |
| 座席状态管理 | 顶栏下拉切换 Available / Offline / 自定义状态，指示灯同步 | `getAgentStates` / `setState` / `onStateChange` / `onRefresh` | A / B |
| 来电弹屏 | 进线自动定位客户档案并滚动到顶部，显示号码、队列、联系 ID、实时计时 | `onIncoming` / `onConnecting` + `getAttributes` / `getQueue` / ANI | A / B |
| 客户匹配 | 优先级：联系属性 `customerId`→`customerName`→刚点击外呼的客户→号码匹配→稳定兜底 | `getAttributes` + ANI | A / B |
| 自动建单 | 进线即创建工单（主题/分类/优先级/摘要/处理结果/是否跟进），接通置"进行中"，可提交结单 | `onConnected` + 本地存储 | A / B |
| 点击外呼 | 客户列表和档案页的号码可直接呼出，号码取自登录页配置的外呼号码 | `Endpoint.byPhoneNumber` + `agent.connect` | A / B |
| 软电话通话控制 | **在 CRM 自己的界面上**接听 / 拒接 / 静音 / 保持 / 恢复 / 挂断 / 结束 ACW | `accept` / `reject` / `mute` / `hold` / `resume` / `destroy` / `clear` | B |
| 转接 · 拨号盘 · Chat 消息 | 不重复造轮子，一键展开完整 CCP 操作 | `initCCP` 的 iframe | A / B |
| 多联系并发 | 多个活动联系各占一行通话条 / CCP 徽标显示数量 | `connect.contact` 逐联系订阅 | A（徽标）/ B（多行） |
| ACW 收尾 | 进入 ACW 时提示补充工单，B 可直接结束 ACW | `onACW` + `clear` | A / B |
| 历史归档 | 通话/Chat 结束后自动写入客户时间线（渠道、时长、队列、结果），工单提交也归档 | `onEnded` / `onDestroy` | A / B |
| 座席退出登录 | 先置 Offline → 调 `/connect/logout` → `terminate()` → 刷新回门禁；有活动联系时二次确认 | `setState` + `terminate` | A / B |
| 布局不遮挡 | A：CCP 停靠右侧整列、主内容让位、可切悬浮/拖动；B：通话条在文档流内下推内容 | 纯前端布局 | A / B |
| 无实例演示 | "模拟进线"可在没有真实通话的情况下走通弹屏 → 建单 → 结单 → 归档（B 里还能模拟接听/保持/挂断） | 无 | A / B |
| 事件日志 | 左下角面板打印 CCP 生命周期、API 成功/失败，便于排查 | 全部回调 | A / B |

---

## 4. 已知限制

- CRM 数据是内置的假数据 + localStorage，不接任何后端；清缓存即丢。
- 所有演示客户共用同一个外呼号码，因此按号码反查客户时只能做"稳定兜底"选择，真实场景应按号码唯一匹配或用联系属性透传 `customerId`。
- 转接、拨号盘、Chat 消息收发、录音回放没有自建 UI，统一交给完整 CCP。
- `index_pro.html` 的通话控制已带 try/catch 和 success/failure 回调兜底，验证过完整状态机（振铃 → 接听 → 保持 → 静音 → ACW）。
- `index.html` 在视口宽度小于 1280px 时退回悬浮面板，此时仍会覆盖内容（与改造前一致）。
- 同一浏览器只应有一个页签初始化 CCP；`index.html` 与 `index_pro.html` 不要同时打开。

## 5. 参考

- [amazon-connect-streams（SDK 仓库）](https://github.com/amazon-connect/amazon-connect-streams)
- [amazon-connect-streams API 文档](https://github.com/amazon-connect/amazon-connect-streams/blob/master/Documentation.md)
- [Amazon Connect 管理指南：应用集成 / Approved origins](https://docs.aws.amazon.com/connect/latest/adminguide/app-integration.html)
