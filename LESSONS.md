# 项目教训

本文件记录已发生的返工与验证遗漏，按根因归并；通用执行规范仍以 AGENTS.md 为准。当前状态见 [交接 #13](https://github.com/AcornFZA/word/issues/13)，发布与回滚见 [#9](https://github.com/AcornFZA/word/issues/9)。以下同一事件的多个分支不重复计次。

## 持久提交先于可观察状态（2026-11-04）

- 规则：持久状态的内存发布必须发生在文件提交成功之后；外部发送前先持久认领逻辑投递及稳定发送ID，记录失败就不发送。
- 原因与事件：旧 FileTargetStore 先改内存，落盘失败后重试会被假去重；微信旧实现没有跨进程投递认领。此类数据一致性风险不能依赖成功路径测试。
- 范围与修复：imapp 的16个持久写入口共用候选状态提交；微信插件持久 journal 保留成功／处理中／未知。不额外宣称断电 fsync 或上游 exactly-once。
- 同类边界修复：`GroupMessageIntents`的get／pending与put输入均隔离可变引用，rename后才发布；实际外发后动作／原卡落盘失败保持unknown。仅改put顺序不能修复引用泄漏，一个store的专项也不能代替其它store。完整callback与10个断言负例见本轮`checks/dingtalk-fix-summary.json`，不宣称单击双发。
- 验证：101发布根 `/var/tmp/im-nine-fixes-20261103T143500Z`，`checks/storage-default-summary.json` 的34项故障回归通过，旧行为34项断言失败；微信重复／重启／写入失败见 `checks/wechat-p1-summary.json`。

## 未知结果保护必须到每次外发边界（2026-11-04）

- 规则：网络超时、响应丢失、部分成功或回执写入失败均不能推断未执行；每次真正外发前核同一业务操作的未决状态，不能仅在创建记录或进入函数时检查。
- 原因与事件：本轮交叉复核发现同一根因的三个遗漏分支：恢复已有待发记录、退避期间出现未知，以及发送后回执和unknown持久化均失败。它们可通过旧记录／新ID回退绕过只在入口的保护，按一个事件记录。
- 范围与修复：Outbox每次发送前核同Run未决状态；未决优先于最终卡回退；durable delivering也阻止再次发送。微信两入口共用iLink未知分类，插件持久去重，独立MCP错误说明不得诱导重发。
- 验证：`checks/outbox-document-summary.json`、`checks/cross-review-p1-outbox-at.json`、`checks/wechat-p1-lessons.json`。Outbox19项通过，恢复旧行为17/18项断言失败，另有逐次发送守卫竞态负例；微信响应丢失、部分成功及journal绕过均有失败负例。observer保持真实unknown，不伪装成功。
- 适用层次补充：外层Outbox冻结不代表适配器内部安全。飞书原卡PATCH或CardKit变更未知时禁止新发／普通PATCH兜底，队列也须阻断；安全unknown向imapp传播并持久冻结。明确卡体拒绝保留兼容回退。回执落盘故障和并发覆盖见本轮`checks/feishu-wechat-control-summary.json`。

- 监控微信补充（同日同事件）：外层SQLite账本不能代替helper到实际HTTP的safe分类与稳定ID。现将同任务／分段／通道／授权范围的持久deliveryId贯通正文和Idempotency-Key；明确context_expired才rejected，丢回执／部分成功／超时仍unknown。自然摘要已实证钉钉sent与微信上下文拒绝；新入站短暂恢复后也可再次失效，不证明旧unknown未发、不推断固定TTL。旧unknown不重分类或补发，24条旧unknown／written已逐条核保持；发布与证据见[#9](https://github.com/AcornFZA/word/issues/9)。

## 公共异步流程必须保留异常前状态（2026-11-03）

- 规则：抽取事件消费流程时，成功与抛异常都必须回传已发生的状态；每次调用有独立上下文，不能用外层闭包或旧快照猜测。
- 原因与事件：正常／恢复共用流程的候选重构曾漏传异常前消费状态；冻结Inbox来源与当前Run字段也不能相互替代。
- 范围与修复：imapp共享准备／消费／收尾，finally同步调用局部状态；保留入口展示、恢复与冻结来源差异。
- 验证：101前置候选 `/data/im-design-fix-20261103T141600Z` 中两项SSE负例及当前发布 `response-lifecycle.test.ts`；最终imapp镜像132项通过。该前置候选已并入本轮部署，不再是待发布状态。
- 续流复用补充（2026-11-04）：核对和消费必须共用当前execution对象，不能让回调闭包继续使用核对前的binding／run；用查询期间持久状态变化、异常后游标及晚返回保护验证。原responseStatusForId只核终态，不能假定它已验证queued／in_progress；在途识别另限续流，不放宽最终结果验收。候选中错误假设由回归发现并纠正，未部署错误候选。最终276项通过、5类保证撤回后10项断言失败，证据 `/var/tmp/im-stream-wait-fix-20261104/checks/`。

## 文本保护以真实编辑语义为准（2026-11-04）

- 规则：通用文档外部编辑保护不能删除空行或行尾空格；有损旧哈希不能兼容为新的覆盖许可。固定生成文档的渲染适配只限已证明的格式差异。
- 原因与事件：旧通用哈希忽略代码块空行等有意义修改。仅修新算法而接受旧hash仍会保留漏检风险。
- 范围与修复：lifecycle-tools使用域分离v2、仅CRLF归一化，旧hash须重新读取核对；personal-events固定摘要的既有DWS适配不变。
- 验证：63项相关回归通过；恢复旧算法7/8项按断言失败。证据 `checks/outbox-document-summary.json`。

## 验证当前契约和实际发布产物（2026-11-04）

- 规则：默认测试须进入现行鉴权及业务路径；不靠删除、跳过或削弱断言掩盖旧夹具失配。发布检查对应最终镜像源码及实际启动命令，候选测试不能替代运行产物核验。
- 原因与事件：imapp旧45项中24项失败，混有旧鉴权／文案／等待夹具，专项通过不足以证明默认兼容；动态npx和启动安装让执行器漂移。部署时DEV Compose省略原项目名触发名称冲突，运行实例身份也必须来自实际配置。
- 同次验证补充：运行镜像曾遗漏已验证的钉钉专项夹具／微信更新夹具；应以实际镜像src测试和源码SHA共同证明，不能只核生产文件。恢复夹具须先核来源与生产基线，不删除断言；根目录遗留bak测试加载失败与src测试失败分别报告。本轮对应`checks/*restored-regression-fixtures.json`及`release-image-verification.json`。
- 范围与修复：保留45项并按实际契约更新、补鉴权拒绝；固定运行时、依赖和飞书tsx，取消启动联网安装；Compose沿用原dingtalk-dev/wechat-dev项目，独立MCP按实际inspect恢复配置。
- 验证：前次实际imapp镜像132/132、两微信32/32与8/8、钉钉专项334/334及断网冷启动；源码SHA与部署后环境／挂载／健康核验见 `release/deployment-receipt.json` 和 `checks/*postdeploy-verification.json`。钉钉全量仍保留同名40项失败及4项前端类型错误，合成验证不替代真实客户端／长期稳定性。部署项目名冲突没有停止原钉钉服务，按原名完成切换后健康通过。

- 多用户候选复核补充：授权ID非空不等于已验证，`unverified:`迁移标记与当前configKey均需核验；跨语言授权代次实际为数值，不能让两侧mock共同假设字符串。已用实际加密FileTargetStore、HTTP helper及真实reader联调两用户，补拒绝与反向验证；证据 `/var/tmp/im-formal-monitor-sync-20261104/checks/imapp-personal-events-summary.json`、`imapp-reader-integration.log`。问题在候选审查中修正，不推断线上已发生越权。禁用微信的负例还须提供原本可用的完整映射，避免缺少夹具字段让移除守卫后仍不调用、掩盖测试遗漏；以实际零调用断言验证。

- DEV多用户迁移补充：发布前后两个运行阶段可能使用不同源路径；预安装的租约证明必须显式使用已验证候选reader，不能让selector已更新而脚本仍指向旧生产文件。首次保护检查在复制数据前失败，使用阶段续接后完成。构建先给本地镜像受控tag并核精确image ID，不能把`sha256:ID`误作Dockerfile仓库标签。本次证据 `/var/tmp/im-dev-monitor-multiuser-20261104/release/initial-failed-receipt.json`、`finish-deployment.py`、`checks/postdeploy-verification.json`。
- 调用方兼容补充：关闭固定配置前查全部调用方；DEV审批旁路仍用旧租约和text-only通知，监控改多用户不能顺手关闭它们。仅保留窄旧契约、新监控按当前本人；双用户、旧卡／微信、审批通知和未知拒绝分支均做实际回归。同日候选遗漏与发布返工按根因归并，不推断为线上越权或重复发送。

- 替代实现后的残留清理（同日续记）：多用户已上线后仍保留旧单账号bootstrap、旧CLI和无使用方参数，后续追问才清理。规则已由用户明确提升为通用／项目准则：同次迁移核调用方、配置和有效测试，删除无兼容用途的旧路径；审批与存储兼容保留用途、退出条件。当前两插件实际676／657项回归仅同名40旧失败，392／375项后台回归通过，恢复旧bootstrap能实际创建SQLite并被断言拦截。验证适配中的class实例字段和遗漏测试helper已纠正，未以加载／清理错误当反向验证通过。证据 `/var/tmp/im-monitor-legacy-cleanup-20261104/checks/`，现状见[#9](https://github.com/AcornFZA/word/issues/9)。

- 可视化未知状态补充（2026-11-04）：加载中、读取失败、过期与明确断开必须分别展示，不能让空值落入“未连接”或“零故障”。网站v6发布截图暴露Dot数据尚未取到时被标为未连接；v7改为按显式状态展示，并保留缺日期/空统计的未知语义。9项前端数据回归与13项MCP回归通过，恢复旧默认值会由断言检出；UI与生产数据验证分开记录，见[#14](https://github.com/AcornFZA/word/issues/14)。

- 实际运行边界补充（同日同事件）：PrivateTmp宿主status无live不代表无consumer；IPC ready／connected不等于AT实收。真实consumer的ProtectSystem及启动顺序与owner合成单元不同；缺失IPC根在strict namespace内创建失败，须由受控tmpfiles先建0700，再验证实际readonly constructor／connect。本次已证不需扩大RW。旧Docker HostConfig可能省略Init，Go模板须index读取并保留null，不能把缺省当false；v2在metadata迁移前安全拒绝后经真实只读验证修正。最终DEV范围已核但整机v3因其它容器漂移未过，不能改原baseline或伪造整体完成。见[#9](https://github.com/AcornFZA/word/issues/9)。
- 业务错误与调用方契约补充（同日同事件）：HTTP200中的PAT_ORG_POLICY_DENIED不当超时／零结果／管理员没开；控制台允许且corp匹配、原profile随后自然恢复，原因仍未知。维护恢复按各调用方实际fresh proof.authority和已测兼容映射定位，审批assistant_authorities为0行不等于无授权；无精确当前行就保留failed，不换身份／授权代次绕过。代码只精确分类failed／blocked，不自动重读。
- 脱敏边界补充（同日同事件）：结构探测也只能输出固定字段白名单／枚举计数，不能输出任意字典keys；token索引的键本身也可能是凭据。本轮一次结构探测曾误输出凭据索引键，仅在工具会话，未写代码／文件／GitHub，已向用户说明并改固定字段白名单。不得公开具体键值，也不因此无证据轮换账户或扩大生产变更。

- 摘要业务语义补充（2026-11-05，用户明确规则）：已确认的他人@本人本身就是关键动态，纯@也不能因模型items为空被省略；中性空文案只纠正措辞，不能代替必列事实。优先复用当前身份／群／时间窗内可信AT收件，再按本人权限完整补齐@我并与可见消息交集，排除自@与普通@文本；事实合并、来源和通知／文档渲染集中维护，模型只补业务摘要。原截图样本只读回放已核，隔离458项／宿主相关185项及5类撤回保证断言见 `/var/tmp/im-mention-key-updates-20261105/` 与[#9](https://github.com/AcornFZA/word/issues/9)。首轮宿主全套因缺Node失败不计通过，完整Node依赖验证留在实际镜像；不把只读回放记成新真实AT／客户端实收，也不回写旧已发送或unknown。

- 告警可理解性补充（2026-11-05）：运维字段不能直接当用户结论；`affectedUsers=0`仅表示未关联用户，异常记录数不等于漏发消息数，运行检查失败不等于已确定停机，检查恢复不承诺历史补齐。旧通知只列服务名/数量/事件号导致用户无法判断影响；网站v8将解释集中为同一纯函数，MCP返回代码生成的通知正文，两个原任务按通俗说明展示，技术标识留详情。35项回归、误报无人受影响的断言负例及本人登录后的真实详情验证见[#14](https://github.com/AcornFZA/word/issues/14)；新文案首轮定时展示仍待验，不把已保存指令当自动投递验证。

- 监控事件关联补充（2026-11-05，候选审查）：事件问答必须经原请求/发送ID精确关联，不能用相似正文、同会话或附近时间补位；旧记录缺正文/原因、后台无具体问答时显式说明。错误依据须按独立事件+异常对象保存，不能只存对象/错误码的最新一份，否则恢复后再次报错会污染旧事件解释。候选已在发布前改为观察台自己的`incident_evidence`，不改产品schema；92项回归及撤回精确关联/独立依据的断言负例通过，真实影子和本人登录后问答字段核验见[#14](https://github.com/AcornFZA/word/issues/14)。通知仍无问答正文，公开文档和截图只用虚构样本。
- 发布路径补充（同日同事件）：仓库中的`observer/OPERATIONS.md`实际部署在101观察台父目录，不能机械拼接源码相对位置。首轮预检在停timer/写线上之前拒绝；按实际路径显式映射后发布成功。预检拒绝不是线上失败或最终验证通过，收据与11项文件指纹见[#9](https://github.com/AcornFZA/word/issues/9)。

- 监控旁路漏验（2026-11-05，已发现未修）：微信直接配置流程已结束、系统回复已投递时可以没有AI Run；把无Run的`not_started`一律算等待，且只排除failed入站，会误报已completed请求。发送后缀还漏`:group-monitor`，页面无法关联该类系统直接答复。原入站/持久发送ID与来源哈希/作用域已核，不能靠相似正文或同一时刻猜关联。现有92项回归未覆盖该旁路，待补终态/流程分类、系统答复关联与回归；投递成功不等于配置业务成功，旧通用错误也不能还原底层原因。本次只读追溯，未改代码或重放，私有证据及当前缺口见[#10](https://github.com/AcornFZA/word/issues/10)、[#14](https://github.com/AcornFZA/word/issues/14)。

- 原生监控续接漏验补充（2026-11-05，用户本轮授权直接改时）：微信文字状态机和监控卡的回归不能证明钉钉原生裸时间已接管。新样本精确Response只有文字却声称投卡、原计划未改；当前以真实Stream owner／thread和按本人范围的明确群选择建有效配置会话，复用同一改时事务，不再投确认卡。无群名必须选择本人已有计划，先给时间也不能猜群；切话题／非文本／过期／授权变化清除续接，明确新群的无效时间不能回落到旧群。规则已在49项新增、实际Stream结构／两用户／持久去重、提交后回执故障和晚返回断言中固化，完整732项同名40旧失败保持；发布、原入口8项及会话守卫1项撤回保证断言、本人明确配置保存的实查见[#9](https://github.com/AcornFZA/word/issues/9)。候选的撤权错误分类漏报由回归修正后才发布；不把新合成链路或实际保存当作手机实收，不因此恢复普通补参模型问题的暂缓范围。

- 完整参数免确认补充（2026-11-05，同日同类入口漏验）：不能只改提示词或某一种入口；核原生私聊、MCP schema、补充卡回调及持久执行器。完整参数直达同一事务，缺项一次提交；同步删除已有计划编辑／保留／更新确认与旧群发送确认执行器，保留仍有兼容用途的历史快照／账本并写退出条件。新原生发送须以插件能力声明隔离旧实例，查真实唯一对象并贯通原Inbox稳定ID、当前授权和回执，未知不重发；后台摘要tools为空的边界保持。截图句式、实际HTTP入站、原生补充卡首次保存、两用户及落盘／重启／并发断言已覆盖；最终imapp309通过、DEV752项同名40旧失败无新增，恢复强制确认／撤回认领／撤回句式的8项AssertionError见 `/var/tmp/im-no-confirm-20261105/checks/negative.json` 与[#9](https://github.com/AcornFZA/word/issues/9)。同日显示返工补充：移除业务确认时要核直接执行结果、补参提交终态及失败回退的实际显示入口；业务免确认不能丢失原生卡片的分段和颜色，也不能用泛化标题掩盖结果。优先复用已发布模板和现有投递，集中维护显示规则，覆盖浅／深色、待核验及停用状态；DEV774项同名40旧失败无新增，3个显示破坏变体共9项AssertionError，证据 `/var/tmp/im-monitor-style-20261105/checks/negative.json`、[#11](https://github.com/AcornFZA/word/issues/11)。不以合成链路或目录schema证明手机实收，也不将旧同文记录当本轮成功。

## 模型执行与历史结果判断的责任边界（2026-11-04；11-05限定范围重新授权）

- 通用原则：可确定的流程、授权、状态及回执核验优先由代码控制；这不意味着把所有模型语义决策改为硬编码，或将自由文本补参自动解释为执行授权。
- 事实保留：截图普通补参续接后模型没有调用发送，追问后又把数小时前同正文记录误认本轮成功；后一条专用确认卡成功属于不同请求。此前只读证据在101 `/var/tmp/im-form-flow-20261104T0043/checks/`。
- 当前归属与范围：此前用户明确将模型问题暂缓；2026-11-05重新授权群发送／监控参数齐全直接执行、仅缺项补充。该限定原生入口和现有工具执行器已按新规则部署，其它通用日程／自由表述漏调用仍保留。通用卡模型续接不等于业务完成，不改TokenHub、不补发旧请求。
- 当前工程审查仅覆盖已有明确动作、任务号或对象后的确定性步骤：持久提交、错误分类、同任务结果核验、查询截止与恢复。检查必须找到实际调用方和可复用实现；按适配层现有契约验收，不统一强加不存在的messageId或私有协议字段，不另建通用工作流框架。
- 状态：确定性工程修复与隔离回归已实施，实际发布状态见[#9](https://github.com/AcornFZA/word/issues/9)；其余模型暂缓范围保持。实现与验证边界见[#10](https://github.com/AcornFZA/word/issues/10)，当前重点见[#13](https://github.com/AcornFZA/word/issues/13)。准则原则由用户明确要求写入，不能把同日分支／追问当作多次独立事件。

## 固定截止与持久重试必须可跨进程验证（2026-11-04）

- 规则：固定原任务ID、输入和重试次数／截止；异常、pending与重启都不能延长预算。截止检查包含授权刷新后的剩余时间，异常也记录退避，让其它任务继续。
- 原因与修复：旧摘要GET异常在截止前return，长期占队首；现为每Response持久600秒、错误15秒／pending5秒退避。微信收件按用户确定最多5次总提交、三分钟内发起新尝试，先写批次／次数再POST，重启继续累计；换授权终止旧批，所有消息终态后原子推进游标。
- 返工教训：冻结消息必须采用实际JSON传输表示，单纯structuredClone会令内存Date与磁盘字符串不一致；未知提交耗尽后的提示不能沿用“稍后再试”诱导新ID重放。等待响应取消也不能无界await。测试回调内的断言可能被生产catch吞掉，应尽量在回调外核可观察状态。
- 验证：本轮 `/var/tmp/im-code-control-fix-20261104T0115/checks/` 的 `digest-control-summary.json`（337通过／13个断言负例）、`wechat-inbound-fix-summary.json`及`wechat-inbound-extra-summary.json`（微信59通过／11个断言负例）。三分钟是新尝试窗口，不是已在途请求或平台业务的完成SLA。

## 监听恢复必须核对全部旧执行者（2026-11-04）

- 规则：区分可以安全重建的正常结束与必须冻结的未知执行；主线程退出不能代表内部线程／子进程都已停止。先确认所有写入者结束，再读取最终清理状态；同进程换群／授权切换、冷启动和热恢复使用相同边界。
- 原因与事件：正式多用户候选曾将同授权worker永久关闭，交叉复核遗漏其可恢复性；改为恢复后又发现内层listener仍可能晚写failed，过早读安全快照会绕过保护。同次候选评审已纠正，不重复计次、不称线上事故。
- 范围与修复：登记锁下检查主线程、processor、session、consume／stderr／watcher及子进程全部退出；至少10秒退避，同绑定所有授权代次listener仅new／stopped且无cleanup_required才重建。旧unknown、撤销及未决投递保持，不靠换授权或重启绕过。
- 验证：本次372项Python回归、Node14项及反向断言证据见 `/var/tmp/im-formal-monitor-sync-20261104/checks/test-summary.json` 和 `personal-events-cross-review.json`；部署状态见[#9](https://github.com/AcornFZA/word/issues/9)。AI复核仅补充实际测试与发布核验。

- DEV双bus与授权补充（同日同事件）：同身份监控／审批曾各持独立上游，审批唯一AT与截图同秒但无mid，不能扩为精确消息级归因。共享必须由独立systemd owner持有，单consumer退出不杀他方，消费角色拒绝Discover自动fork；Broker全局latest token的CAS不防旧token晚覆盖，须同canonical scope锁内force中央当前proof至ready。业务数据／查询凭据隔离，授权变化等旧执行者全退，非空attempt或unknown不能靠换根绕过，最后consumer采用有界idle。
- 实际退役与前置失败补充（同日同事件）：两个旧服务正常退出却仍有两本方SID active；原生忽略Delete错误且上层丢弃stderr，具体原因不能还原，cgroup／锁退役不等于云撤订。仅在同锁／同当前grant、持久dispatch意图后逐SID单次精确cancel，并以完整GET确认本方2缺失／无关1保留。旧Node PID1未响应默认TERM／INT，经审查的单次例外先pause，持有全部已知SQLite BEGIN IMMEDIATE锁，fresh核DB及已知JSON无在途并私有备份后，才精确KILL；真实137／非OOM／完整cgroup空按受控终止记录，不冒充应用优雅排空。新Compose仅image＋init:true。另对已证constructor前置目录失败、无scope／native提交的精确2行，本地failed→stopped保留attempts／subscription／updated及native预算，不清其它失败或unknown；新真实AT仍待验。收据及恢复见[#9](https://github.com/AcornFZA/word/issues/9)。
