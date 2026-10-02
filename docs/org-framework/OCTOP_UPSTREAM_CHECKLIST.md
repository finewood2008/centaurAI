# 半人马组织框架 · Octop 上游核实清单（给 GPT）

2026-10-02 · Claude 读上游源码整理 · 用途：请 GPT 对照万象 / 知君 0.4.2 的实际代码逐条核实

## 0. 背景

**框架**：同一套 WANX（即 Octop 版万象）形成两种节点。个人小盒子承载个人工作上下文、记忆、凭据、任务编排和局部执行；组织大盒子承载组织目标、制度、共享知识、公共业务能力、算力和协作调度。客户端只连自己的小盒子；节点之间走 A2A，并由对齐机制约束：身份与职责、目标与规则、知识与事实、任务与状态、成果与反馈。

**johny 已拍板（2026-10-02）**：

1. 账号与节点身份在盒子内闭环，断网可用。企业微信、飞书、钉钉 SSO 是可选项。
2. 默认私密：组织侧只看用量、组织空间和安全事件；查看个人内容必须留痕，并通知本人。

**核心判断**：Octop 的多用户按“一家人”设计，`README_CN.md:49` 原文是“一人管理，全家共用”。组织框架需要“同事”式的信任模型：节点之间按任务交换必要内容，不互相开放全部数据。下面大多数条目，都是这两种信任模型之间的差距。

## 1. 怎么用这份清单

- **读的版本**：上游 Octop 1.0.2b5（`e473dd3`，2026-09-29），以及 PyPI 上的 `octop-harness`、`octop-gateway`、`octop-memory`、`octop-browser` 1.0.0。0.4.2 基于哪个上游版本，Claude 不知道，请先确认。
- **每条请回答**：0.4.2 里是否已改（是 / 否 / 部分）、对应文件；如果没改，给出最小改法和影响面。**动代码前先和 johny 确认范围。**
- **标记**：
  - 【复核】Claude 本人对照源码看过。
  - 【读码】由读码子任务得出，未逐行复核。
  - 【推断】按代码路径推出，需要在隔离测试机上实测。
- **适用**：〔大〕组织大盒子，〔小〕个人小盒子，〔两者〕两种节点都适用。
- **路径**：`octop_harness/…`、`octop_gateway/…`、`octop_memory/…`、`octop_browser/…` 指 PyPI 包；其余相对 Octop 仓库根目录（`src/octop/…`、`docker/…`、`dashboard/…`、`docs/…`）。

## 2. 节点之间（A2A 与对齐）

**N1 Octop 没有节点之间的通用 A2A**〔两者〕【复核】
- **上游**：唯一实现 A2A 协议的是华为小艺 IM 通道，`octop_gateway/channels/xiaoyi/channel.py:1,224,369`。它是一个 IM 入口，不是 Octop 实例之间的协议。实例互联用的是 Octop 自有的 Bridge，见 `docs/bridge.md` 和 `src/octop/infra/bridge/`。
- **要定**：个人节点和组织节点之间的通道，是新建 A2A 端点（可参考小艺通道的收发），还是收窄 Bridge。无论选哪种，都不能沿用 Bridge 的默认权限（见 N2）。

**N2 Bridge 的权限是“以账号主人身份操作他名下 agent 的一切”**〔两者〕【复核】
- **上游**：连接以“对端地址 + 用户名 + 密码”建立；密码加密保存，用于自动重连；连接双向对称；可见范围等于该账号在对端的可见范围（`docs/bridge.md` 第 1 节）。隧道白名单允许对端对 `/api/agents/{id}/` 下的 `threads|history|files|workspace|memory|skills|config|cron|…` 执行 GET、POST、PUT、PATCH、DELETE（`src/octop/infra/bridge/tunnel_policy.py:10-17,66-67`）。
- **冲突**：组织节点经 Bridge 可以读写个人节点的记忆、工作区和配置。这违背“个人节点保留长期上下文，组织侧按任务接收必要内容”。
- **目标**：跨节点只开放任务级接口（下发任务或规则、提交结果、查询状态），不开放记忆、工作区、配置的通用读写；内容随任务携带，并注明保留方式。

**N3 任何登录用户都能建 Bridge**〔两者〕【读码】
- **上游**：建连只校验登录，`src/octop/api/routers/bridge.py:120-123`；设了自动重连的连接在启动时恢复，`src/octop/infra/server.py:529-534`。
- **目标**：节点之间的连接只能由组织大盒子签发、管理员建立，普通用户不能自建对外连接。另一个办法是不挂载 Bridge 路由（`src/octop/api/app.py:233`）、不启动 BridgeManager（`src/octop/infra/server.py:507-534`）。

**N4 节点身份**〔两者〕【设计约束】
- **依据**：已拍板“身份在盒子内闭环”。参照公司盒 Remote Agent 的设备证书 mTLS 做法，节点之间改用组织大盒子签发的节点证书，不用 Bridge 现在的用户密码。
- **要求**：节点私钥不能放在 agent 能读到的地方，见 E1、E6。

**N5 跨节点任务和网页对话挤同一个 4 名额队列**〔大〕【复核】
- **上游**：Bridge 远程回合进入网页对话的同一个通道，`src/octop/infra/bridge/manager.py:1337`（`channel_manager.enqueue(WS_CHANNEL_ID, inbound)`）。所有员工节点发来的跨节点任务，加上网页对话，合计同时只能跑 4 轮（见 C1）。
- **目标**：跨节点任务单独排队，有自己的并发上限，并能看到排队位置。

**N6 外部通道来的人，一律算作 agent 主人**〔大〕【复核】
- **上游**：`src/octop/infra/gateway/process/message_keys.py:79-95` 把 IM 等外部通道的会话一律记在 agent 主人名下；同一 agent 的外部成员共用主人的浏览器 profile，彼此不隔离。A2A 如果按通道方式接入，也会这样。
- **目标**：每个跨节点请求都要映射到“哪位员工的哪个节点”，按该员工的权限执行、按该员工记账，结果只回该节点。

## 3. 执行与密钥（两种节点都要改）

组织大盒子上要防两件事：员工之间互相越界，以及一个人的 agent 动到全公司的数据。个人小盒子上只有一个人，但 agent 可能读到藏了指令的网页或邮件，进而偷走节点私钥，冒充本人向组织下单。

**E1 默认执行后端是整台主机**〔两者〕【复核】
- **上游**：
  - 默认后端是 `local_shell`、根目录 `/`，源码注释原话是“full host access by default; restrict via backend.root_dir in production”（`octop_harness/backends/__init__.py:7-9,78-82`）。
  - `inherit_env` 默认为 True；根目录为 `/` 时只打警告（同文件 `:296-306`）。
  - `src/octop/infra/backend/resolver.py:23-31` 沿用这个默认值；新用户的默认 agent 写死主机根目录（`src/octop/infra/agents/experts/default_agent.py:27-38,81`）。
  - 命令直接在宿主执行（`octop_harness/backends/local_shell.py:289-300`）。bwrap 目录隔离只在“Linux、根目录不是 `/`、已安装 bwrap”三者同时满足时启用（`octop_harness/backends/bwrap_shell.py:46-64`）【读码】。
- **目标**：强制使用容器沙箱，只挂载本人工作区。上游已有 `octop_harness/backends/docker_sandbox.py`，默认 512MB 内存、1 核、最多 256 个进程、断网、单条命令 120 秒【复核】。也可评估 OpenSandbox 后端（`src/octop/infra/backend/opensandbox_deps.py`）。注意：让服务进程拿到 Docker 权限约等于给它 root，沙箱的调度方式需要单独设计。
- **验收**：普通员工让 agent 执行 `ls ~/.octop/agents`、`env`、读取 `~/.octop/octop.db`，都失败。

**E2 agent 主人能改执行后端和安全策略**〔两者〕【复核】
- **上游**：
  - 创建接口 `src/octop/api/routers/agents.py:281-295` 和 PATCH 接口 `:378-411` 都允许提交整个 config，只校验 backend 的根目录。
  - agent 的 `config.security` 会覆盖全局策略（`src/octop/infra/agents/manager.py:3099-3101`）。
  - Docker 参数原样透传（`src/octop/infra/backend/docker_spec.py:10-29`）【读码】。
- **目标**：`backend` 和 `security` 由管理员策略锁定，普通用户提交这两项一律拒绝。

**E3 安全策略默认不拦截**〔两者〕【复核】
- **上游**：
  - 审批默认关闭，护栏是 warn 模式（`src/octop/infra/agents/security/policy_store.py:18-23`）；warn 模式不阻断任何操作（`octop_harness/security/tool_guard/engine.py:15-22,58-62`）。
  - 审批由发起对话的用户自己批，还能设成本会话全部放行（`src/octop/api/routers/chat/routes.py:185-215`、`src/octop/infra/agents/security/hitl_session.py:54-61`）【读码】。
  - `README_CN.md:526` 的说法与默认值不符。
- **目标**：出厂设为 block 或 require_approval；涉及组织资源的高风险动作，可配置为由组织侧审批。

**E4 shell 继承服务进程的全部环境变量**〔两者〕【复核】
- **上游**：`octop_harness/runtime_env.py:159-173`。compose 会注入 `OPENAI_API_KEY`、`OCTOP_DATABASE_PASSWORD`（`docker/docker-compose.yml:49-54`）【读码】；Docker 沙箱只拿到精简后的环境（`runtime_env.py:176-195`）【读码】。

**E5 任何登录用户都能读到模型 key**〔大〕【复核】
- **上游**：`src/octop/api/routers/providers.py:189-196` 只要求登录，`:134` 返回 `api_key` 明文。
- **目标**：普通用户只拿到模型名和能力，不含 key 和 base_url。可参考公司盒的“平台托管模型”。

**E6 密钥明文存在数据库里**〔两者〕【复核】
- **上游**：JWT 签名密钥见 `src/octop/infra/server.py:627-629` 和 `src/octop/infra/db/repos/secrets.py:15-36`；模型 key 原样写入，见 `src/octop/infra/db/repos/providers.py:80-88`。
- **后果**【推断】：agent 读到 `octop.db` 后，可以伪造管理员 token。个人小盒子还多一条要求：节点私钥同样不能落在 agent 够得着的地方。

**E7 登录就能借自定义 MCP（stdio）在服务器上启动进程**〔大〕【复核】
- **上游**：`src/octop/api/routers/connectors.py:561-588,621-647` 只要求登录；`src/octop/infra/connectors/probe.py:536-552` 用 `stdio_client` 启动子进程。配置还能标成 shared 推给全员（`src/octop/infra/connectors/service.py:578-593`）【读码】。
- **目标**：stdio 类型只有管理员能建、能测。

**E8 网页文件接口能碰到主机路径**〔大〕【部分复核】
- **浏览**【复核】：`src/octop/api/routers/filesystem.py:101-220` 允许任何登录用户从 `/` 开始浏览、建目录、改名，只排除 `/proc /sys /dev /etc /root`（`src/octop/infra/utils/host_dirs.py:23`）。
- **读取与下载**【读码】：`src/octop/api/routers/workspace.py:86-110,156-169`、`src/octop/infra/gateway/media/backend_files.py:316-322,367-368`（显式放行 `/.octop/agents/`）、`octop_harness/backends/workspace.py:23-27,610-617`。
- **目标**：所有文件接口限定在“本人工作区 + 被授权空间”的虚拟根目录内。

**E9 网页终端开的是宿主 shell**〔两者〕【读码】
- **上游**：`src/octop/api/routers/terminal.py:81,180-181,310-319,525,552`。需要 terminal 权限且是 agent 主人；开的是宿主上的真 shell，带服务进程的全部环境变量；全进程最多 10 个。
- **目标**：对普通用户关闭，或者让终端进入本人的沙箱。

**E10 所有人的浏览器登录态在同一个系统账号下**〔两者〕【读码】
- **上游**：
  - 每个用户一个 profile（`src/octop/infra/utils/browser_media.py:34-41`、`src/octop/infra/agents/middleware/browser_profile.py:26-44`）。
  - 所有 profile 都在同一系统账号的 `~/.octop/browser-profiles` 下（`browser_media.py:84-102`）。
  - CDP 端口从 `localhost:9222` 起分配（`octop_browser/profile.py:151-174`、`octop_browser/settings.py:139-146`）。
- **风险**【推断】：别人的 agent 在宿主上执行时，能读取 cookie，或经 CDP 接管浏览器。
- **目标**：浏览器放进各自的沙箱。

**E11 桌面控制和手机控制**〔两者〕【读码】
- **桌面**：控制的是服务器本机的显示，最多 3 路，没有控制锁（`src/octop/infra/desktop/setup.py:285-296`、`src/octop/infra/desktop/session.py:17,82-103`）。
- **手机**：整台机器只绑一部手机，后连的人会把前一个顶掉（`src/octop/infra/mobile/agent_control.py:13-45`）；默认关闭（`src/octop/config.py:110-113`）。
- **目标**：组织大盒子上默认关闭；个人小盒子按需开启。

**E12 官方镜像以 root 运行**〔两者〕【复核】
- **上游**：`docker/Dockerfile` 没有 `USER` 指令，`docker/docker-entrypoint.sh` 也不降权。

## 4. 共享、身份与组织（组织大盒子）

**S1 共享只有私有、全员两档，没有部门**〔大〕【读码】
- **agent**：admin、主人或 `is_shared` 的 agent 可访问（`src/octop/api/common/agent.py:18-31`）；任何主人都能设 `is_shared`，不需要权限（`src/octop/api/routers/agents.py:435-436`、`src/octop/infra/db/repos/agents.py:213`）。
- **知识库**：`shared` 就是全员只读（`src/octop/infra/knowledge/service.py:324-346`）；成员表已在迁移 007 里删除。
- **目标**：组织空间按部门、项目授权；检索时把权限条件下推进向量检索，不要全库检索完再过滤。可参考公司盒 Data Engine 的授权检索。

**S2 共享 agent 的记忆、工作区、凭据全员共用**〔大〕【记忆部分复核】
- **上游**：
  - 记忆召回按 agent 的命名空间进行，不区分人（`octop_harness/middleware/memory.py:455-470`）。`octop_memory/service.py:76-90` 里的 `session_id` 只用来排除当前会话。
  - atoms 表没有 user 列（`octop_memory/storage/backends/sqlite.py:283-298`）【读码】。
  - 共享的连接器用的是主人的凭据（`src/octop/infra/connectors/service.py:596-614`）【读码】。
- **目标**：组织的公共业务 agent（例如报价 Agent）同时接多个节点的任务时，记忆和产出按任务、按发起人分开；内容要跨人沉淀，只能经人工发布。

**S3 记忆游标按线程 id 存**〔两者〕【复核】
- **上游**：`octop_harness/middleware/memory.py:268,315`（`self._cursors[threading.get_ident()]`）。所有异步回合跑在同一个事件循环线程里，同一个 agent 并发跑两轮时，游标会互相覆盖，记忆抓取的消息范围就错了。组织公共 agent 同时服务多个节点时，一定会触发。
- **目标**：改成按“thread_id + 回合 id”存。

**S4 会话键可以串进别人的线程**〔大〕【代码复核，后果待实测】
- **上游**：`src/octop/api/routers/chat/turn.py:60-79`。传 `thread_id` 时会校验归属；只传 `session_key` 时，直接返回已绑定的线程，不校验归属。
- **目标**：session_key 由服务端生成，或者至少校验其中包含本人的 user_id。

**S5 管理员能读所有人的对话和记忆**〔大〕【读码】
- **上游**：`as_user` 见 `src/octop/api/common/agent.py:51-58`；记忆读取见 `src/octop/api/common/memory_client.py:155`。
- **已拍板**：默认私密。新框架下，个人的长期上下文本来就在个人节点；组织大盒子上应该只有随任务带来的内容。
- **目标**：跨人读取改成“申请 → 留痕 → 通知本人”；审计记录管理员删不掉。

**S6 角色与权限**〔大〕【读码】
- **上游**：
  - 唯一的特权角色是 admin（`src/octop/infra/users/identity.py:29-31`）。
  - 自定义角色只是“23 个模块键 + 3 项策略”的模板，分配时复制给用户，之后改模板不回写（`src/octop/api/routers/user_roles.py:3-4`、`src/octop/api/routers/users.py:467-485`）。
  - “聊天用 agent 永不设闸”（`src/octop/infra/users/permissions.py:4-5`）。
  - 有 users 权限就能重置管理员密码（`src/octop/api/routers/users.py:528-538`）。
- **目标**：角色能限定可用的公共业务能力和模型；非 admin 不能操作 admin 账号。

**S7 SSO 与部门**〔大〕【读码】
- **上游**：
  - 支持 OIDC、飞书、钉钉、企业微信（`src/octop/infra/auth/sso/providers/base.py:9`）。
  - 首次登录自动开户（`src/octop/infra/users/manager.py:252-279`）。
  - 只取姓名和邮箱，不同步部门（`src/octop/infra/auth/sso/providers/wecom.py:143-160`）。
  - 每种类型只能配一个 IdP（迁移 015）。
  - 企业微信扫码登录需要服务器能出网（`wecom.py:18-20`）。
  - 不支持 LDAP、AD、SAML。
- **目标**：联网时能从企业微信、飞书、钉钉同步部门；断网时盒子账号照常登录（已拍板）。

**S8 技能和插件的来源**〔两者〕【读码】
- **插件**：在服务主进程里执行的 Python，启动时用 pip 装依赖（`octop_harness/plugins/loader.py:29-41,55-79`）；安装需要 plugins 权限（`src/octop/api/routers/plugins.py:108-167`）；预装之后主人可以关掉，管理员锁定不了（`src/octop/infra/agents/plugins/plugin_tool_defaults.py:17-24`）。
- **技能**：普通用户能从网址或 SkillHub 给自己的 agent 装技能，脚本由 agent 的命令行执行（`src/octop/api/routers/skills.py:940,1377`）。
- **目标**：技能和插件只来自组织审核过的库；组织下发的规则和业务技能可以锁定，并带版本号。

## 5. 并发与调度（以组织大盒子为主）

**C1 网页对话加 Bridge 远程回合，全进程合计同时 4 轮**〔大〕【复核】
- **调用链**：入口 `src/octop/api/routers/chat/ws.py:174` → `src/octop/infra/gateway/gateway.py:244`（`ChannelManager(channels={})`，没传 worker 数）→ `octop_gateway/manager.py:92`（默认 4）→ `:665`（每个通道起 4 个 worker）→ `:751`（worker 持有会话锁直到整轮结束）。
- **目标**：调大 worker 数（例如 32），同时设每人、每节点的并发上限；任务入队后把排队位置推给前端。
- **验收**：20 个账号同时发消息，记录首字延迟的分布和每人看到的排队位置。

**C2 队列满时静默丢弃**〔大〕【复核】
- **上游**：`octop_gateway/manager.py:292-294`。
- **目标**：明确拒绝，并告诉调用方。

**C3 同一会话连发两条，第二条也占一个名额**〔大〕【读码】
- **上游**：第二条消息拿着 worker 名额等会话锁（`octop_gateway/manager.py:751`）。

**C4 定时任务不排队、没有上限**〔两者〕【复核】
- **上游**：`src/octop/infra/cron/delivery.py:83-87` 只拿会话锁，`:143` 直接调用 `agent_manager.stream`；APScheduler 没配执行器（`src/octop/infra/cron/manager.py:65`）【读码】。
- **目标**：定时任务单独排队，设全局上限和每人上限，整点触发的任务错峰执行。

**C5 没有模型限流，也没有显式超时**〔两者〕【读码；重试部分复核】
- **上游**：`octop_harness/llm/factory.py:260-283,643-655`；重试配置为 2 次、首次间隔 1 秒、最长 60 秒（`octop_harness/config/__init__.py:641-645`）【复核】；模型实例在 agent 之间共享缓存（`factory.py:186-199`）。
- **目标**：按供应商做全局限速，设显式超时；对齐外部 API 的 RPM、TPM，或本地推理的并发上限。

**C6 记忆整理是隐形负载，线程数没有上限**〔两者〕【部分复核】
- **上游**：
  - 会话空闲 300 秒后触发抽取（`octop_harness/config/__init__.py:800-803`）【复核】。
  - 抽取包括候选抽取、晋升、页面重写、情绪日记四步（`octop_memory/application/runtime.py:545-609`）【结构已复核】，少则 2 次、多则二十几次模型调用。
  - 每次新开一个守护线程（`octop_harness/middleware/memory.py:94-118,443`），单次超时 120 秒（`octop_harness/memory/llm_client.py:59`）【读码】。
- **目标**：放到后台低优先级处理，用有上限的线程池，可以延后执行，可以指定本地小模型。

**C7 主动关怀默认开启**〔两者〕【复核】
- **上游**：`src/octop/infra/db/repos/proactive_care_config.py:16-24`，默认开启，时段 09:00–22:00，间隔 5–24 小时，每个 agent 一个任务。
- **目标**：组织大盒子上默认关闭；个人小盒子沿用知君已定的推送规则（按内容分类和渠道区分）。

**C8 事件循环上跑着同步重活**〔两者〕【读码】
- **上游**：
  - 控制面数据库同步调用，单连接加锁（`src/octop/infra/db/pool.py:33-75`）。
  - 每轮记忆召回同步执行（`octop_harness/middleware/memory.py:362→464`）；它的 200ms 超时实际不封顶（`octop_memory/pipeline/recall/timeout.py:71-76`）。
  - 知识库预览同步解析 PDF 并做 OCR（`src/octop/api/routers/knowledge_bases.py:768` → `src/octop/infra/knowledge/service.py:190` → `src/octop/infra/knowledge/parse.py:49-55`）。
  - 在线编辑 docx 时的格式转换（`src/octop/api/routers/workspace.py:388,413`）。
  - 知识检索用纯 Python 全表计算余弦（`src/octop/infra/knowledge/index.py:98-134`）。
  - 本地 embedding 每次都重新加载模型（`src/octop/infra/agents/providers/onnx_service.py:389-401`）。
- **目标**：把这些移出事件循环；embedding 模型常驻内存；检索改用向量索引。

**C9 永远是单进程**〔两者〕【复核】
- **上游**：
  - `src/octop/launch.py:93,140` 把 workers 传给了 `uvicorn.Config`，但 `:145,152` 是用 `uvicorn.Server(...).serve()` 启动的，多 worker 不生效。
  - 会话锁、WS 连接表、待审批记录都只在进程内存里（`octop_gateway/manager.py:118`、`src/octop/infra/agents/manager.py:378`、`src/octop/infra/gateway/hitl/store.py:32-36`）【读码】。
  - ADR 001、002 声明只做纵向扩展，不承诺多实例写入。
- **目标**：组织大盒子先在单进程内修好 C1–C8；规模再大，就把执行拆出主进程。ADR 001 指出的拆分点是 `src/octop/infra/gateway/processor.py`。

**C10 常驻 agent 不回收**〔大〕【读码】
- **上游**：启动时构建全部启用的 agent（`src/octop/infra/agents/manager.py:484-488`），常驻在注册表里（`octop_harness/registry.py:51-84`）；每个 agent 带一个记忆实例、一个每小时跑的维护线程（`octop_harness/middleware/memory.py:684`）和一个主动关怀任务。
- **目标**：实测常驻内存，必要时加空闲回收。

**C11 改用 PostgreSQL 后连接数会超**〔大〕【读码】
- **上游**：控制面连接池 1–8 条且是同步调用（`src/octop/infra/db/pool.py:154`）；每个 agent 还有 1 条记忆连接和 1–4 条 checkpoint 连接（`octop_memory/storage/backends/postgres.py:119`、`octop_memory/core.py:1801-1813`）。60 个 agent 约需 120–300 条连接，超过 PostgreSQL 默认的 100 条。

## 6. 离线与对外连接（两种节点）

**O1 联网搜索默认接第三方服务**〔两者〕【复核】
- **上游**：
  - harness 默认 `web_search_tools="auto"`（`octop_harness/config/__init__.py:744`）。
  - searchfree 不需要 key，auto 模式下总会挂上（`octop_harness/builtin/tools/web_search/_registry.py:72-80`），地址是 `https://searchfree.site/api/search`（`octop_harness/builtin/tools/web_search/searchfree.py:21-24`）。
  - Octop 没有覆盖这个默认值（`src/octop/infra/agents/manager.py:3240-3267`）【读码】。
- **目标**：默认关闭；需要联网搜索时，由组织配置认可的服务。

**O2 更新检查访问 pypi.org，没有开关**〔两者〕【读码】
- **上游**：`src/octop/infra/setup/self_update.py:26,188`；页面加载、窗口切回、每小时都会触发（`dashboard/src/hooks/useUpdateStatus.ts:47-67`、`src/octop/api/routers/update.py:187-201`）。
- **目标**：加离线开关；升级改走我们自己的签名包。

**O3 语音**〔两者〕【读码】
- **上游**：服务端语音地址只允许公网 https（`src/octop/infra/voice/adapters.py:89-91` → `src/octop/infra/utils/ssrf_guard.py:53-70`）；移动端朗读强制走微软 Edge TTS（`dashboard/src/hooks/useVoiceOutput.ts:175-201`）。
- **目标**：允许组织配置内网语音服务；去掉 Edge TTS 回退。

**O4 官方镜像缺本地组件**〔两者〕【读码】
- **上游**：`docker/Dockerfile:110,127` 只装了 browser 这一组依赖，没有 Chromium、bubblewrap、pg_dump、fastembed、rapidocr；embedding 模型权重从腾讯 COS 或 HF 下载（`src/octop/infra/agents/providers/onnx_download.py:25-28`）。
- **目标**：镜像预装中文 embedding、OCR、Chromium、pg_dump，断网也能用。

**O5 Linux 启动时会自动安装 bubblewrap**〔两者〕【读码】
- **上游**：`src/octop/launch.py:16-40,76`、`src/octop/infra/utils/bwrap.py:121-155`。
- **目标**：镜像里预装好，启动时不出网。

**O6 自动证书只支持公网域名**〔两者〕【读码】
- **上游**：只支持 ACME HTTP-01，需要公网域名和 80 端口（`src/octop/infra/setup/tls/acme_issue.py:25-26,101-112`、`src/octop/infra/setup/tls/modes.py:21-32`、`src/octop/infra/setup/tls/preflight.py:25-33,252`）。内网只能自签或自带证书（`src/octop/cli/commands/run.py:152-159`、`src/octop/infra/setup/tls/store.py:19-31`）。
- **目标**：组织大盒子出厂即带本地 CA，为自己和各个小盒子签发证书，同一套证书也用作节点身份（见 N4）。浏览器语音输入需要 HTTPS。

**O7 插件启动时用 pip 装依赖**〔两者〕【读码】
- **上游**：`octop_harness/plugins/loader.py:29-41,55`。
- **目标**：离线时改用预置的 wheel 包。

## 7. 运维（两种节点）

**M1 备份**〔两者〕【读码】
- **上游**：
  - 默认不备份对话（`src/octop/config.py:95-103`、`src/octop/infra/backup/system_archive.py:187-204`）。
  - 只存在本地，不加密（`src/octop/infra/backup/store.py:1`）。
  - 自动备份默认关闭。
  - PostgreSQL 备份依赖 `pg_dump`，镜像里没有（`src/octop/infra/backup/pg_dump.py:13-31`）。
- **目标**：出厂默认每天做加密的全量备份（包括对话），存到第二介质，并演练过恢复。个人小盒子的备份不能默认放到组织读得到的地方：如果要存在组织大盒子上，应该用员工自己的密钥加密，组织只保存密文。

**M2 升级**〔两者〕【读码】
- **上游**：升级要手动触发；从写死的镜像源和 pypi 拉取（`src/octop/infra/setup/self_update.py:34-39,859`）；数据库迁移在启动时自动执行（`src/octop/infra/server.py:325`）。
- **目标**：用签名的离线升级包，升级前自动备份，可回滚；维护大小盒子之间的版本兼容表，跨节点协议要带版本号。

**M3 监控与审计**〔大〕【读码】
- **上游**：
  - 没有 Prometheus 或 OTel；`/api/admin/metrics` 只有 5 个内存计数器（`src/octop/infra/metrics.py:10-37`）。
  - 审计单次最多查 500 条，模型和密钥的变更不审计（`src/octop/api/routers/admin.py:47-76`）。
  - 用量能按人、agent、模型、日期导出（`src/octop/api/routers/usage.py`）。
  - `token_quota` 是终身累计值，不按月重置（`src/octop/infra/users/resource_policy.py:182-193`）。
- **目标**：监控队列长度、推理吞吐和每个节点的用量，并能告警；配额按月计算；密钥变更进审计。

**M4 历史累积到上限（公司盒的教训）**〔两者〕
- **事实**：2026-09-15，公司盒因为历史任务记录满了 1024 条而拒绝新操作，当时没有任务在运行（知君仓库 `docs/development/request-pressure-20260915.md`）。
- **目标**：验收时加一项“模拟使用三个月”的累积测试，检查各类历史表、日志和索引的上限与回收。

## 8. 商用

**L1 打包了 Anthropic 专有许可的技能**〔两者〕【复核】
- **上游**：`src/octop/infra/agents/experts/library/office-automation/skills/{docx,xlsx,pdf,pptx}/LICENSE.txt` 和 `octop_harness/builtin/skills/{en,zh}/powerpoint/LICENSE.txt`，版权为“© 2025 Anthropic, PBC”，条款禁止复制、改编和分发。
- **目标**：发行物里移除这些技能，自研同功能的技能替代。请法务确认。

**L2 品牌元素分散在各处**〔两者〕【读码】
- **上游**：
  - logo 用固定路径写在多处：`dashboard/src/layouts/Header.tsx:25-26`、`dashboard/src/layouts/Sidebar.tsx:432-496`、`dashboard/src/pages/Login/index.tsx:263-265`、`dashboard/src/pages/Setup/index.tsx:262-265`、`dashboard/index.html:5-32`、`dashboard/public/manifest.json:3-4`。
  - 文案里带“Octop”的，zh.json 有 79 行，en.json 有 72 行。
  - 配色集中在 `dashboard/src/styles/themePalettes.ts`。
  - 帮助和 GitHub 链接写死在 `dashboard/src/components/AvatarDropdown.tsx:59-60`。
  - 主仓和四个依赖包都是 MIT 许可，保留版权声明即可。

## 9. 需要在隔离测试机上实测的

1. 普通用户能否通过下载接口拿到 `~/.octop/octop.db`（E8）。
2. 在共享 agent 上，用别人的 session_key 能否进入别人的线程（S4）。
3. 别人的 agent 能否读到 cookie，或经 CDP 接管浏览器（E10）。
4. 20 个账号同时对话时的排队情况和首字延迟（C1）；再叠加 20 个节点同时发来的跨节点任务（N5）。
5. 同一个 agent 并发跑两轮时，记忆抓取会不会错位（S3）。
6. 模拟使用三个月后，各类上限会不会被触发（M4）。

## 10. 不在本清单里的（属于设计问题，等 johny 定）

- **对齐协议本身**：身份与职责、目标与规则、知识与事实、任务与状态、成果与反馈，这五个方面各自交换什么、谁有权执行、由谁确认。
- **个人小盒子的推理从哪里来**：如果小盒子调用大盒子的算力，这是不是 A2A 之外的第二条跨节点通道？推理服务能不能保证不留存、不进入组织记忆？
- **个人小盒子是否一定是独立硬件**：如果它是跑在组织大盒子上的虚拟节点，第 3 节的全部条目也适用于个人节点之间。
- **人员交接**：员工离开时，个人节点里的工作上下文怎么交接，哪些归组织、哪些归本人。
