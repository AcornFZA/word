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

## 模型执行与历史结果判断的责任边界（2026-11-04，用户暂缓）

- 通用原则：可确定的流程、授权、状态及回执核验优先由代码控制；这不意味着把所有模型语义决策改为硬编码，或将自由文本补参自动解释为执行授权。
- 事实保留：截图普通补参续接后模型没有调用发送，追问后又把数小时前同正文记录误认本轮成功；后一条专用确认卡成功属于不同请求。此前只读证据在101 `/var/tmp/im-form-flow-20261104T0043/checks/`。
- 当前归属与范围：用户明确认定上述事件是模型问题，先不处理；不再称为本项目当前业务缺陷或普通补参改造重点。通用卡保持模型续接／授权边界，不改TokenHub，不补发旧请求。此前“本项目应优先接管该截图流程”的表述以本条为准。
- 当前工程审查仅覆盖已有明确动作、任务号或对象后的确定性步骤：持久提交、错误分类、同任务结果核验、查询截止与恢复。检查必须找到实际调用方和可复用实现；按适配层现有契约验收，不统一强加不存在的messageId或私有协议字段，不另建通用工作流框架。
- 状态：确定性工程修复与隔离回归已实施，实际发布状态见[#9](https://github.com/AcornFZA/word/issues/9)；模型暂缓范围不变。实现与验证边界见[#10](https://github.com/AcornFZA/word/issues/10)，当前重点见[#13](https://github.com/AcornFZA/word/issues/13)。准则原则由用户明确要求写入，不能把同日分支／追问当作多次独立事件。

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
