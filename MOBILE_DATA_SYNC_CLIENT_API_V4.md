# V4 客户端接口文档

适用于移动端接入。通用同步协议见[移动端数据同步接口设计 V4](MOBILE_DATA_SYNC_DESIGN_V4.md)。长期记忆采用[长期记忆设计](CHAT_MEMORY_DESIGN_FINAL.md)定义的 `memories` Scope。网红人设采用[网红资料同步设计](ROLE_PROFILE_SYNC_DESIGN.md)，通过消息创建回包交付。更新于 2026-11-24。

亲密度与主动消息修订（2026-11-17）：亲密度快照统一为 `intimacy_score: {version, data: {level}}`，仅通过 `messages:batchCreate` 按独立版本回传；本次不增加亲密度读取接口，不扩展 `sync-state`，也不允许空消息批次。亲密度升级主动消息使用正式 `MessageContent.proactive_context` 交付触发信息，详见 2.2.1、5.4 和 5.5；服务端来源、监测、生成及重试规则以[聊天服务端完整设计](SERVER_DESIGN_FINAL.md)的亲密度模块为准。本修订覆盖其他同步专题文档中的旧 `score` 字段和亲密度 Epoch 特例。

post链接客户端确认写入修订（2026-11-24，用户确认，代码已接入）：最新合同为 `client-commit-v3`，替代此前仅设计的aggregate-v2。创建只提交未同步聊天的整合content和稳定reply_message_id，不要求来源消息ID、版本或客户端messages Epoch。服务端只检索并生成候选，状态generated表示等待客户端上传；客户端将原回复与proactive_context经messages:batchCreate提交，generated候选按客户端业务已发布处理，后续取消、post/Memory失效或占用变化不阻止原消息同步；正式事务成功后任务才completed，post才永久记为已链接。最终生成直接使用原content、人设、post和可选Memory，不读取服务端近期聊天；摘要仅用于检索。生成默认300秒、单次回复模型120秒，生成成功后不设上传截止时间，未上传就持续保留候选及post占用，后续检索排除该post。响应不增加客户端动作字段，仅在错误码表增加“客户端动作”列。代码、运行Schema、Prompt和联调工具已按本合同更新；未执行实际数据库迁移或部署，验证记录见SERVER_IMPLEMENTATION.md。

历史实现边界（2026-11-23）：旧单消息message-v1已实现并离线验证，MULTIMODAL_ENABLED默认false；它由服务端正式发布消息。旧请求/任务按12.6恢复，不能拿旧已发布消息当作新候选再次上传。aggregate-v2从未实现，不产生需要迁移的运行记录；client-commit-v3代码已更新，新客户端须在目标环境完成兼容迁移与部署后切换。

演示账号回包例外（2026-11-18）：认证user_id精确为`ios-demo-test`或`zyy`时，每次成功batchCreate均返回固定`intimacy_score={version:"1",data:{level:"LV2"}}`，不受请求版本或真实亲密度状态影响，详见5.4。该例外不写真实状态、不触发主动消息；其他用户仍按真实版本条件返回。

| 方法与路径 | 用途 |
|---|---|
| POST /auto-media/api/chat/threads/query | 会话列表、预览及本机阅读基准下的未读数 |
| GET /auto-media/api/chat/roles/{role_id}/sync-state | 可选轻量探测，不是同步读取前置步骤 |
| POST /auto-media/api/chat/roles/{role_id}/messages:batchCreate | 幂等创建用户/端侧消息，或校验并提交post候选回复；按独立版本回传人设和亲密度 |
| POST /auto-media/api/chat/roles/{role_id}/messages:batchGet | 按消息 ID 批量查询当前会话的有效消息正文 |
| POST /auto-media/api/chat/roles/{role_id}/scopes/messages/mutations | 幂等删除一条消息及其关联记忆 |
| GET /auto-media/api/chat/roles/{role_id}/scopes/messages/mutations/{mutation_id} | 查询消息删除状态及历史回执 |
| POST /auto-media/api/chat/roles/{role_id}/scopes/{scope}/read-batches | 创建本轮读取batch；同响应给边界、建议跨度及可选内联数据 |
| POST /auto-media/api/chat/roles/{role_id}/scopes/{scope}/records:query | 按显式参数完整查询一个区间 |
| POST /auto-media/api/chat/roles/{role_id}/multimodal-tasks | 按未同步content检索并生成post候选；不直接写消息（client-commit-v3） |
| GET /auto-media/api/chat/roles/{role_id}/multimodal-tasks/{task_id} | 查询生成状态、待上传候选或已经正式提交的消息 |
| POST /auto-media/api/chat/roles/{role_id}/multimodal-tasks/{task_id}:cancel | 幂等取消业务生成任务 |

## 1 公共类型、Header、路径与响应

字段标为必填时必须出现。可为 null 的字段会单独标明。

### 1.1 公共类型

| 类型 | JSON表示 | 含义与校验规则 |
|---|---|---|
| `ID` | 非空字符串，UTF-8最多128字节 | 业务或资源标识，例如 `"msg_01"`。`user_id`、`role_id`、`message_id`、`object_id`、`asset_id`、`mutation_id`、`task_id`、`post_id`、`moment_id`、`card_id` 都使用字符串，不按数字解析，不比较大小。具体生成方在对应字段中说明。 |
| `UInt64String` | 十进制数字字符串 | 非负 64 位无符号整数的线格式，例如 `"0"`、`"104"`。用于同步序号、游标、`thread_revision` 及两个快照的数字版本。首期聊天同步的序号和位置上限为 `9223372036854775807`。客户端按整数精确解析和比较。 |
| `OpaqueString` | 非空字符串，UTF-8最多128字节 | 不透明标识。`sync_epoch`、`messages_sync_epoch`、`receipt_sync_epoch` 等按完整字符串判断相等。 |
| `TimestampMs` | JSON整数 | Unix纪元起的毫秒时间戳，非负；不是十进制数字字符串。created_at_ms、server_created_at_ms、server_time_ms、任务expires_at_ms/generated_at_ms均使用此类型；V4读取batch不返回任务期限。 |
| `Count` | JSON整数 | 非负条数、页容量或分区数量，不是序号字符串；具体上下限由字段约束。例如`record_count=20`、`dimension=1024`。 |
| `Scope` | 字符串枚举 | `messages`、`memories`，区分大小写。完整同步范围还包含认证用户和当前`role_id`。 |
| `JSONObject` | JSON对象 | 对象结构由对应业务模块定义；不能用序列化后的JSON字符串代替对象。 |
| `JSONValue` | 合法JSON值 | 可为数字或业务模块定义的对象等合法JSON值；具体业务类型与空值含义由该业务模块约定，同步层不猜测内部字段。 |
| `Vector` | JSON数字数组 | 长期记忆向量首期固定 1024 维，元素为有限数值，按 float32、L2 归一化和余弦相似度约定使用。拒绝无效零向量。 |
| `PageToken` | 非空字符串或`null` | 服务端生成的不透明列表分页标识；首次请求用`null`，客户端原样传递。它不是Seq、Cursor或分区索引。 |

### 1.2 公共请求Header

| 字段 | 类型 | 必填条件 | 含义及来源规则 |
|---|---|---|---|
| `Authorization` | 字符串 | 所有接口必填 | 当前登录用户的访问令牌，使用`Bearer <access_token>`格式。服务端从令牌解析`user_id`并校验会话权限；请求正文不能指定另一个用户替代该身份。 |
| `X-Device-ID` | `ID` | 所有接口必填 | 客户端安装实例标识，由客户端生成并稳定保存。用于设备与请求诊断，不是用户ID、消息幂等键或多设备已读合并依据。 |
| `X-Request-ID` | `ID` | 所有接口必填 | 本次HTTP调用的追踪ID，由调用方生成；同一业务请求重试时可以使用新的追踪ID。业务去重仍依赖message_id、mutation_id或reply_message_id。服务端将此值原样返回为公共字段request_id，用于关联单次HTTP请求和响应。 |
| `Content-Type` | 字符串 | 发送JSON请求正文时必填 | 固定为`application/json`。GET不发送请求正文。 |

### 1.3 公共路径参数

本文沿用 V4 客户端接口前缀 `/auto-media/api/`。专题设计中的示例使用 `/v4/chat/`，部署联调时需确认网关前缀。路径参数由 URL 编码承载，解码后按下表校验。

| 字段 | 类型 | 必填条件 | 含义及来源规则 |
|---|---|---|---|
| `role_id` | `ID` | 路径包含该参数时必填 | 原样使用当前会话的网红角色 ID，其值来自网红资料的原 `user_pk` 字符串。服务端校验认证用户有权访问该角色。 |
| `scope` | `Scope` | 路径包含该参数时必填 | `messages` 支持 recent/history/full/incremental 和消息删除，`memories` 仅开放 full/incremental 读取。 |
| `mutation_id` | `ID` | 查询Mutation资源时必填 | 客户端首次持久化操作时生成的稳定操作ID。查询或重试使用原ID，不能为一次网络重试新建逻辑操作。 |
| `task_id` | `ID` | 查询或取消多模态任务时必填 | 服务端创建任务后返回的资源ID。它不同于回复消息ID；查询任务不会另建回复消息。 |

`user_id`不作为客户端可指定的公共请求参数，始终由认证上下文获得。会话列表通过请求中各`local_read_positions[].role_id`标识阅读位置所属会话，规则与路径`role_id`一致。

### 1.4 公共JSON响应字段

| 字段 | 类型 | 必填条件 | 含义及来源规则 |
|---|---|---|---|
| `request_id` | `ID` | 所有JSON响应必填 | 服务端返回的本次调用追踪ID，原样返回本次`X-Request-ID`；不代表业务操作已完成，也不用于幂等判断。 |
| `server_time_ms` | `TimestampMs` | 所有JSON响应必填 | 服务端生成该响应时的时间。仅用于时间展示与诊断，不能充当同步游标、对象版本或消息顺序。 |
| `error` | 对象，结构见1.5 | HTTP失败，或查询到status=failed的操作/任务时必填 | 统一错误结构；普通成功结果不返回。HTTP200仍可成功查询到一个业务失败的任务。业务字段在同一响应对象，不额外假定有data包装层。 |

V4两个同步读取入口成功均200，没有读取资源preparing/201/202/204。业务创建及任务仍按具体接口返回200/201/202；202不表示本地同步完成。所有HTTP JSON返回公共字段；嵌套RecordsPayload不重复HTTP公共字段。

### 1.5 Error

| 字段 | 类型 | 必填条件 | 含义及来源规则 |
|---|---|---|---|
| `error.code` | 非空字符串 | 失败时必填 | 供程序判断的稳定错误码，取值见 HTTP 状态与错误码。客户端根据此值执行重试、迁移、重建或冲突处理，不解析`message`决定流程。 |
| `error.message` | 字符串 | 失败时必填 | 服务端提供的错误说明，便于定位与展示；不替代错误码的协议语义。 |
| `error.retryable` | 布尔值 | 失败时必填 | 是否允许按当前业务参数自动重试。需要重建、修改参数、处理冲突或迁移的错误，不能仅因网络重试而解决。多模态的生成、待上传及正式写入分别按第12节状态和错误表处理；retryable=false不等于可以重新回复，尤其不能将上传失败当作生成失败。 |
| `error.retry_after_ms` | JSON整数 | 失败时必填 | 服务端建议的最小等待毫秒数，非负；没有等待建议时为0。它是相对时长，不是时间戳，也不表示一定允许重试。 |
| `error.details` | `JSONObject` | 可选 | 错误码专用的结构化信息，例如当前 Epoch 或冲突位置。内部字段见对应接口及第 13 节。 |

## 2 共享业务数据结构字段字典

本节定义被多个接口复用的嵌套结构。未特别说明的必填条件针对该结构已经出现的情况；整个可选结构缺失，并不等于返回一个空快照或删除业务数据。

### 2.1 SyncRecord：统一业务同步记录

适用于普通区间与 `inline_result.records[]`。认证用户、角色、Scope 和 Epoch 由已校验的读取上下文确定。

| 字段 | 类型 | 必填条件 | 含义及来源规则 |
|---|---|---|---|
| `object_id` | `ID` | 必填 | 对象的稳定业务ID；messages时等于`content.message_id`。新增时确定，跨重试及同步不变，删除后不复用。 |
| `sync_seq` | `UInt64String` | 必填 | 对象当前正式发布或删除的服务端同步序号。同一Scope、同Epoch内可以比较大小；更新/删除取得新值，不能由客户端伪造，也不要求全表连续。 |
| `object_seq` | `UInt64String` | 所有Scope的upsert必填；delete省略 | 同Scope/Epoch的永久创建号，所有创建范围归属均按这个外层字段验证；messages的content不重复。更新不改号，删除记录仍仅三个基础字段。 |
| `operation` | 字符串枚举 | 必填 | `upsert`表示以完整当前对象插入或替换；`delete`表示应用删除。messages只在新增时出现upsert，不支持已发送消息编辑。 |
| `content` | `MessageContent` 或 `MemoryContentV1` | upsert 必填，delete 省略 | `messages` 使用 2.2，`memories` 使用 2.4。以完整对象替换本地旧版本。 |
| `embedding` | `Vector` | `memories` 的 upsert 必填，其他省略 | 对 `content.summary` 编码的向量。正文、摘要向量、发散问句及各自向量与 `sync_seq` 同次发布。 |

delete仅返回`object_id`、`sync_seq`、`operation`，不返回旧正文、向量或其他对象字段。对象版本由所属`sync_epoch + sync_seq`标识；上行确认与单条记录都不能直接推进整批cursor。

记录的 `object_seq` 和 `sync_seq` 为正整数，读取游标可为 `"0"`。

### 2.2 MessageContent：已确认消息的完整正文模型

这是下行upsert及completed任务message使用的正式权威模型；generated任务的message使用第12节MessageCreateInput候选形状。消息创建请求仅包含消息批量创建接口定义的输入字段，不能因此上送服务端分配的字段。

| 字段 | 类型 | 必填条件 | 含义及来源规则 |
|---|---|---|---|
| `message_id` | `ID` | 必填 | 消息的永久业务ID，等于同步外层`object_id`。生成端首次持久化前确定；用户输入与回复使用不同ID。 |
| `sender_type` | 字符串枚举 | 必填 | `user`或`role`，表示实际业务发言方。不能用它推断消息从上行还是下行获得。 |
| `sender_id` | `ID` | 必填 | user时必须等于认证用户ID，role时必须等于当前会话`role_id`。由生成端提供、服务端按权限及发送者类型校验。 |
| `origin` | 字符串枚举 | 必填 | 服务端确认的生成来源：普通user为client_user、普通role为client_llm；亲密度主动消息为server_llm；通过任务绑定校验后经batchCreate提交的post回复为server_multimodal。客户端不能上传origin，也不能凭一个自造proactive_context选择服务端来源。 |
| `reply_to_message_id` | `ID`或`null` | 字段必填，可为null | 新客户端user/role消息、亲密度主动消息及client-commit-v3的post回复均为null；未同步content没有明确消息ID，服务端不猜测最近用户消息。旧message-v1已记录的明确关联保留，不改写历史消息；客户端创建请求不传该字段。 |
| `proactive_context` | `ProactiveContextV1`或`null` | 已确认下行必填，可为null；post候选上传时必填非null | 亲密度或post引用信息，见2.2.1。普通消息为null；客户端只可在batchCreate中原样回传已绑定任务的post分支，不能自造或上传亲密度分支。 |
| `message_type` | 字符串枚举 | 必填 | 协议枚举为`text`、`image`、`audio`、`video`或`mixed`，必须与body一致；当前`post_link`回复固定为`text`，不会复制post媒体到聊天附件。 |
| `body` | 对象，见2.3 | 必填 | 文本及媒体附件的完整内容容器。至少有非空文本或一个有效附件。 |
| `created_at_ms` | `TimestampMs` | 必填 | 生成端确定的展示时间。client-commit-v3 post回复使用服务端候选冻结的generated_at_ms，上传原样携带；server_created_at_ms是实际入库时间，两者可不同。亲密度仍按正式发布时间规则；重试时间不变，排序用object_seq。 |
| `server_created_at_ms` | `TimestampMs` | 已确认下行必填；创建请求不传 | 服务端首次接受消息的时间。幂等重试返回首次值，不用本次回包时间替代。 |

发送者、来源、回复关联、主动消息触发信息、首次时间和稳定消息顺序在创建后不可改。本版messages不提供update；删除只作用于选中消息，已存在的回复保留。

消息对象的创建位置不属于MessageContent：SyncRecord、创建回执及TaskState均在其外层返回object_seq，避免正文重复元数据。

### 2.2.1 ProactiveContextV1：主动消息触发信息

非null结构是按`trigger_type`判别的联合类型，`schema_version`均固定为JSON整数`1`。必须完整匹配下列一个分支，每个分支内的字段全部必填，不混用字段，也不增加未定义字段。客户端不得自行构造这些服务端元数据；仅允许按第5.6节将任务返回的post分支原样回传，亲密度分支仍不接受客户端上传。

亲密度等级提升分支保持原有结构：

| 字段 | 类型 | 含义与校验规则 |
|---|---|---|
| `schema_version` | JSON整数，固定`1` | 元数据结构版本，与消息`sync_seq`、亲密度版本独立。 |
| `trigger_type` | 字符串枚举 | 固定`intimacy_level_up`，表示首次达到某亲密度等级。 |
| `intimacy_level` | 字符串 | 本条消息对应的触发等级，格式为`LV`加无前导零的正整数，当前必须至少为`LV2`。按数字部分解释等级，不按整个字符串比较大小。 |
| `moment_id` | `ID` | 生成该等级卡片的任务`task_id`。该字段不是聊天生成任务ID，也不是消息ID。 |
| `card_id` | `ID` | 上述任务的`published_card_id`，对应实际采用的`current_result.cards[].card_id`。 |
| `link_type` | 字符串枚举 | 固定`intimacy_card`，表示亲密度卡片链接类型。客户端结合`moment_id`、`card_id`识别卡片跳转目标。 |

post链接分支只包含以下五个字段：

| 字段 | 类型 | 含义与校验规则 |
|---|---|---|
| `schema_version` | JSON整数，固定`1` | 元数据结构版本。 |
| `trigger_type` | 字符串枚举 | 固定`post_link`，表示本轮整合文本命中了适合链接的post。 |
| `link_type` | 字符串枚举 | 固定`post`。 |
| `post_id` | `ID` | 本条回复实际采用的post业务ID；客户端按自身产品能力展示。 |
| `task_id` | `ID` | 生成本条回复的多模态任务ID，与消息ID及post_id不同。 |

client-commit-v3正式post回复满足sender_type=role、sender_id=role_id、origin=server_multimodal、reply_to_message_id=null、message_type=text、非空body.text及body.attachments=[]。候选与上传的文本、created_at_ms及整个post上下文须相同；origin由服务端验证任务后派生。post_id不放入moment_id/card_id，不含媒体二进制或临时URL，正文最多300个Unicode码点。最终生成以客户端原content、人设、post及可选Memory为输入，不读取服务端近期聊天；Memory ID、模型配置和检索分数不通过本结构下发。

`trigger_type`表示触发原因，`link_type`表示关联内容的链接类型。以下关于等级、卡片、串行和无限请求重试的规则只适用于`intimacy_level_up`；post链接任务遵循第12节的有限重试和有效期。

亲密度主动消息满足`sender_type="role"`、`sender_id=role_id`、`origin="server_llm"`、`reply_to_message_id=null`、`message_type="text"`，并包含非null的上述结构。消息语言使用生成时冻结的网红`default_language`；生成结合角色人设、卡片文案和最近聊天，不召回长期记忆。

每个用户与网红的组合，每个等级最多发送一次；首次接入已为LV5时只补发LV5，随后从LV5升到LV8时依次补发LV6、LV7、LV8。同一组合先完成低等级消息生成与正式发布，才生成高等级消息；在同一messages Epoch内，低等级已发送消息的`object_seq`严格小于高等级，`created_at_ms`和`server_created_at_ms`也严格递增。不同`(user_id, role_id)`组合可以并行；同一用户与不同网红也属于不同组合。

低等级等待重试期间，高等级继续等待。卡片异常或模型成功返回后内容解析、结构校验失败会消耗重试次数；耗尽后该等级永久跳过，其后等级继续处理。模型请求失败不消耗次数并持续退避重试，不能把成功返回的无效内容归类为请求失败。消息提交到业务数据库即视为已发送，不设置发送有效期；删除消息不会重新触发该等级。`link_type`、`moment_id`、`card_id`和触发等级保留发布时信息，卡片后续撤回或更新不改写旧消息。具体失败分类、次数及恢复规则见服务端完整设计。

post在客户端收到generated后按业务已发布处理；batchCreate负责正式同步入库，成功后才将服务端去重状态转sent，按用户、角色、post永久去重。generated阶段持续保留占用，后续取消、post/Memory失效、占用变化或未上传时长均不撤销原候选的同步资格；占用异常可在正式上传事务内按固定task/post绑定修复。内部目标messages Epoch/G变化的会话重置边界另见12.5。删除已提交回复、清理墓碑或切换Epoch不允许再次链接已sent的post。

本结构是消息正文的一部分，随messages的普通区间、`inline_result.records[]`和`messages:batchGet`一致返回，客户端与整条消息一同保存。删除墓碑仍严格只有`object_id`、`sync_seq`、`operation`，不携带本结构。`latest_message`继续只提供预览字段，不能据其推断触发信息。客户端messages:batchCreate仅能按任务绑定原样回传post的proactive_context，不能上传origin或自造亲密度分支。proactive_context不是metadata.context_provenance，不代替生成来源声明。

### 2.3 MessageBody与Attachment

| 字段 | 类型 | 必填条件 | 含义及来源规则 |
|---|---|---|---|
| `body.text` | 字符串或`null` | 字段必填 | 消息文本。纯媒体消息可为空字符串或null；无附件时必须含有效文本。 |
| `body.attachments` | `Attachment[]` | 必填 | 媒体附件数组；没有附件时为`[]`，不使用null。不能仅凭此数组存在就认为含有效内容。 |
| `attachments[].asset_id` | `ID` | 每个附件必填 | 已准备好的稳定媒体资源ID。由媒体准备/上传链路提供，用于定位资源；不能用会过期的签名URL替代并写入冻结创建负载。 |
| `attachments[].media_type` | 字符串枚举 | 每个附件必填 | `image`、`audio`或`video`，表示这个附件的媒体种类，不使用消息级的`mixed`。 |
| `attachments[].mime_type` | 非空字符串 | 每个附件必填 | 附件的MIME类型，例如`image/jpeg`、`audio/mpeg`或`video/mp4`；由媒体元数据提供，应与实际资源及media_type相符。 |

该结构没有内嵌资源二进制、临时URL、下载进度或模型向量。媒体读取/鉴权由媒体模块承担，不在本同步协议中扩展未定义字段。

### 2.4 MemoryContentV1：长期记忆正文

`memories` 的 upsert 采用本节结构，字段均必填，可空项单独标明。另见[长期记忆设计第 4.2 节记录示例](CHAT_MEMORY_DESIGN_FINAL.md#42-memorycontent-v1)和[附录 A JSON Schema](CHAT_MEMORY_DESIGN_FINAL.md#附录a-完整memory同步记录schema)。

| `content` 内字段 | 类型 | 含义与约束 |
|---|---|---|
| `schema_version` | JSON 整数，固定 `1` | 业务结构版本。对象更新顺序仍按 `sync_seq` 判断。 |
| `title` | 非空字符串 | 标题，最多 120 个 Unicode 字符。 |
| `summary` | 非空字符串 | 完整摘要，最多 12000 个字符，且受生成及上下文预算约束。 |
| `language` | 字符串枚举 | `zh`、`en`、`ja`。 |
| `source_start_at_ms`、`source_end_at_ms` | `TimestampMs` 或 `null` | 来源聊天的最早、最晚时间，均非空时前者不晚于后者。与记忆描述的事件时间分别解释。 |
| `root_message_ids` | `ID[]` | 完整且去重的根来源消息 ID 集合，至少一项。用于已知删除的本地反查。 |
| `retrieval_queries` | 对象数组 | 1 至 5 条发散问句。同一父记忆内 `query_id` 和规范化后的 `text` 分别唯一。 |
| `retrieval_queries[].query_id` | `ID` | 父记忆内的稳定子视图 ID。 |
| `retrieval_queries[].text` | 非空字符串 | 发散问句。`zh` 最多 80 字符，`en` 最多 180 字符且最多 40 个空白分词，`ja` 最多 100 字符。 |
| `retrieval_queries[].embedding` | `Vector` | 对该问句 `text` 编码的服务端向量。 |
| `created_at_ms`、`updated_at_ms` | `TimestampMs` | 首次与最近一次正式发布时间，前者不晚于后者。 |

父记忆更新时整体替换正文、根来源、摘要向量和全部问句视图，清除新版本已移除的子视图。摘要与问句向量都使用第 1 节的向量约定。问句只用于召回，生成时使用有效父记忆的标题和摘要。问句没有独立的 Scope、版本或消息记录。

### 2.5 SyncHint：服务端同步位置提示

在列表项、写入回包及变化检查中复用。列表项从外层`role_id`确定会话，其余接口从路径确定会话。

| 字段 | 类型 | 必填条件 | 含义及来源规则 |
|---|---|---|---|
| `scope` | `Scope` | 每个提示必填 | `messages` 与 `memories` 的位置相互独立。 |
| `sync_epoch` | `OpaqueString` | 每个提示必填 | 服务端当前同步代次。与本地不同则重新建立该Scope基线；不能比较Epoch大小判断新旧。 |
| `server_max_sync_seq` | `UInt64String` | 每个提示必填 | 服务端取得此提示时已经提交的最高同步位置，初始为`"0"`。用于发现变化，不表示客户端已下载。 |
| `min_valid_cursor` | `UInt64String` | 每个提示必填 | 当前仍允许完整增量衔接的最小游标。低于此值必须重建；等于此值仍有效。它不是剩余对象的最小Seq。 |

提示用于触发实际读取。完整 `sync-state` 返回两个 Scope，写入回包按具体接口返回变化提示。通过完整读取推进游标，删除清理目标见第 6、8 节。

### 2.6 SessionSnapshot：两个独立会话内快照

消息创建回包中的`structured_information`和`intimacy_score`为同级可选结构。各自有自己的数字Version及替换规则，不相互嵌套。

| 字段 | 类型 | 必填条件 | 含义及来源规则 |
|---|---|---|---|
| `structured_information` | `StructuredInformationSnapshot` | 按第 5.2 节的版本规则返回 | 当前网红的完整人设快照。省略时保留当前会话中仍有效的快照。 |
| `structured_information.version` | `UInt64String` | 结构出现时必填 | 按角色维护的正整数版本，首次发布为 `"1"`，完整人设实际变化时递增。`"0"` 保留给客户端表示无快照。版本独立于消息 Epoch、`sync_seq` 和亲密度版本。 |
| `structured_information.data` | `JSONObject` | 结构出现时必填 | 完整 `persona_features`，保留原始键名、未知字段、嵌套层次、数组顺序和值类型。按版本整体替换，`{}` 是有效空人设。单个完整快照上限为 512 KiB，按未压缩紧凑 UTF-8 JSON 计。 |
| `intimacy_score` | `IntimacyScoreSnapshot` | 普通用户已有已接受等级且请求版本不等时返回；演示账号每次成功回包均返回 | 普通用户为当前认证用户与该网红的完整亲密度快照；ios-demo-test/zyy固定版本1、LV2，详见5.4。与structured_information独立更新；省略时保留当前会话中仍有效的快照。 |
| `intimacy_score.version` | `UInt64String` | 结构出现时必填 | 普通用户每个用户与网红组合独立维护的正整数版本，首次接受等级时为`"1"`，只在接受更高等级时递增；演示账号固定`"1"`，不代表真实关系版本。不与人设版本横向比较，也不随消息Epoch变化。`"0"`表示客户端无快照。 |
| `intimacy_score.data` | `JSONObject` | 结构出现时必填 | 完整亲密度业务值，当前仅有必填的`level`字段；不使用旧`score`字段。 |
| `intimacy_score.data.level` | 字符串 | 结构出现时必填 | 普通用户为本服务最后接受的等级，格式为`LV`加无前导零的正整数，当前至少为`LV2`，例如`"LV2"`、`"LV10"`；普通用户不返回null、虚构默认等级或数值分数。演示账号固定`"LV2"`，详见5.4。 |

两个快照独立处理，均通过消息创建回包获取。`sync-state`不提供亲密度查询参数或快照，`read-batches`、`records:query`及`messages:batchGet`只读取其指定的同步业务对象，不另附当前亲密度快照。没有新消息时不构造假消息或空`messages`批次获取快照。`persona_features`由服务端组装，独立提取的语言、展示名和媒体属性有各自的用途。端侧生成时，按固定键顺序遍历快照的顶层字段，逐项拼接为`key: value`。字符串直接使用，其余值用JSON表示。

亲密度仅应用到当前账号、角色、会话实例和已确认的消息 Epoch，保存在会话内存。会话重进、进程重启或确认消息 Epoch 变化后清空，无快照时请求版本用 `"0"`。

### 2.7 LatestMessage：会话列表预览

列表项的`latest_message`仅用于预览。没有有效消息时整个字段为null；存在时使用以下固定五个字段，不替代完整MessageContent或完整区间读取。

| 字段 | 类型 | 必填条件 | 含义及来源规则 |
|---|---|---|---|
| `message_id` | `ID` | latest_message非null时必填 | 服务端选出的最新有效消息ID；正常删除后可转为更早仍有效的消息。 |
| `sender_type` | 字符串枚举 | latest_message非null时必填 | 最新消息的发言方，`user`或`role`，与权威消息一致。 |
| `sender_id` | `ID` | latest_message非null时必填 | 最新消息实际发送者ID，与sender_type及所属会话对应。 |
| `message_type` | 字符串枚举 | latest_message非null时必填 | 最新消息的类型：`text`、`image`、`audio`、`video`或`mixed`。 |
| `preview` | 字符串 | latest_message非null时必填 | 服务端生成的列表展示摘要；可为截断文本或媒体展示说明。不是完整消息正文，不能凭它建立本地权威消息或推进已读/cursor。 |

`LatestMessage`不额外包含`object_seq`、完整body或向量。最新消息ID不变也可能发生旧消息删除，变化检查仍须读取Scope的sync_seq提示。

### 2.8 不透明业务对象的边界

| 使用位置 | 类型 | 本同步文档负责的约束 |
|---|---|---|
| 多模态任务的`options` | `JSONObject` | 由task_type对应业务模块定义的任务参数；作为冻结生成请求的一部分参与幂等一致性判断。不能用改变同一回复ID的options来重新生成。 |
| `error.details` | `JSONObject` | 错误码专用的结构化补充信息，按具体接口说明解释。 |

“不透明”只表示同步层不解释其内部业务字段，不表示服务端可以跳过该业务模块的schema、权限与内容校验。

### 2.9 RecordsPayload 与大小口径

RecordsPayload是records:query响应剔除request_id和server_time_ms后的记录对象，字段逐项见8；inline_result复用普通模式的RecordsPayload，不重复HTTP公共字段。

服务端使用固定的UTF-8紧凑JSON、有限数字及确定字段/数值编码测量字节，包含全部嵌套正文、向量和适用配置，不含Header、压缩和公共HTTP字段。单SyncRecord<=524288字节；普通RecordsPayload<=1048576字节；inline_result<=262144字节。

普通非空区间宽度最多 200。超宽或超响应字节返回 413。

所有客户端 HTTP 请求与响应均省略模型协商字段。`memories` 读请求省略 `expected_embedding_model_version`，成功和错误响应、内联数据、`sync-state` 和 `sync_hints` 均省略 `embedding_config`、模型版本和安装包标识。向量兼容性由随应用发布的本地模型保证，见第 11 节。

## 3 会话列表查询：threads/query

`POST /auto-media/api/chat/threads/query`。第1节Header和公共JSON响应字段适用于本节及后续全部接口，不在每个表重复列出。`[]`表示数组中每一项的路径；引用一个共享结构时，该结构的每个内部字段见2。

请求字段：

| 字段路径 | 类型 | 必填/默认值 | 含义与使用规则 |
|---|---|---|---|
| `page_size` | Count | 可省略，默认20；1～100 | 本页最多返回多少个会话，不是每个会话的消息条数。分页过程中保持不变。 |
| `page_token` | PageToken | 首次省略或null；续页必填 | 服务端上一次返回的next_page_token，绑定本轮固定线程ID集合和顺序。已签发的标识或其绑定集合过期、被回收时返回410 PAGE_TOKEN_EXPIRED；格式错误、被篡改或与分页参数不匹配时返回400 INVALID_ARGUMENT。 |
| `local_read_positions` | 对象数组 | 可省略，默认[] | 本设备各会话已读位置。可以携带多于本页的会话；同一role_id最多一项。没有对应项时按未知阅读状态处理。 |
| `local_read_positions[].role_id` | ID | 每项必填 | 本条阅读位置属于哪个网红会话，不是当前消息发送者。 |
| `local_read_positions[].messages_sync_epoch` | OpaqueString或null | 每项必填；未知为null | 阅读位置所属的messages代次。与当前代次不同，服务端忽略旧阅读位置并返回reset_read_position=true。 |
| `local_read_positions[].read_through_object_seq` | UInt64String | 每项必填；无状态为"0" | 用户已确认阅读的object_seq前缀，不是最大缓存消息、不等于sync_seq。当前Epoch下不得超过current_object_seq。 |

响应字段：

| 字段路径 | 类型 | 出现条件 | 含义与使用规则 |
|---|---|---|---|
| `next_page_token` | PageToken | 必填 | 下一页标识；没有下一页时null。它不推进任何Scope的cursor。 |
| `has_more` | boolean | 必填 | 本轮固定会话集合是否还有下一页；应与next_page_token是否为null一致。 |
| `threads` | 对象数组 | 必填，可为[] | 本页会话状态；不包含该会话的全部聊天记录。 |
| `threads[].role_id` | ID | 必填 | 对聊网红ID，用于本地会话去重和路由。 |
| `threads[].thread_revision` | UInt64String | 必填 | 当前服务端会话内容/预览版本。同messages Epoch内比较，不代表用户阅读进度。 |
| `threads[].messages_sync_epoch` | OpaqueString | 必填 | 本条会话及其消息序号所属代次，供缓存和本地阅读状态匹配。 |
| `threads[].current_object_seq` | UInt64String | 必填 | 已分配过的最大消息创建序号。删除不回退，也不等于最新有效消息序号。 |
| `threads[].unread_basis_object_seq` | UInt64String | 必填 | 服务端本次实际用于计数的请求阅读基准；无状态或旧Epoch时为"0"。 |
| `threads[].unread_count` | Count | 必填 | 当前有效网红消息中object_seq大于本次计数基准的条数。用户消息、墓碑不计入。 |
| `threads[].reset_read_position` | boolean | 必填 | 是否需要按当前messages Epoch重建本机阅读位置；本轮请求Epoch缺失或不同则为true。 |
| `threads[].latest_message` | LatestMessage或null | 必填 | 最新有效消息的展示预览；全部消息已删或会话无有效消息时null。五个内部字段见2。不能据预览推进cursor或标读。 |
| `threads[].sync_hints` | SyncHint数组 | 必填，可为[] | 对应会话各Scope的变化提示；结构字段见2。提示只用于安排读取任务。 |

只有回包的messages Epoch、计数基准和本机当前阅读基准匹配时，才考虑采纳unread_count；还需过滤过时thread_revision。不能仅比较列表版本，因为相同内容版本可以因请求阅读位置不同而得到不同计数。

`POST /auto-media/api/chat/threads/query`，本次请求的 `X-Request-ID` 为 `req_threads_1`。请求正文如下。

```json
{
  "page_size": 20,
  "page_token": null,
  "local_read_positions": [
    {
      "role_id": "role_1",
      "messages_sync_epoch": "msg-e1",
      "read_through_object_seq": "98"
    }
  ]
}
```

响应（HTTP 200）：

```json
{
  "request_id": "req_threads_1",
  "server_time_ms": 1793670000000,
  "next_page_token": null,
  "has_more": false,
  "threads": [
    {
      "role_id": "role_1",
      "thread_revision": "105",
      "messages_sync_epoch": "msg-e1",
      "current_object_seq": "100",
      "unread_basis_object_seq": "98",
      "unread_count": 1,
      "reset_read_position": false,
      "latest_message": {
        "message_id": "msg_100",
        "sender_type": "role",
        "sender_id": "role_1",
        "message_type": "text",
        "preview": "明天见"
      },
      "sync_hints": [
        {
          "scope": "messages",
          "sync_epoch": "msg-e1",
          "server_max_sync_seq": "105",
          "min_valid_cursor": "0"
        }
      ]
    }
  ]
}
```

### 3.1 分页过期与重新查询

续页请求使用的page_token已过期或其固定会话集合已被回收时，返回HTTP 410及完整Error。假设本次X-Request-ID为req_threads_expired，响应如下。

```json
{
  "request_id": "req_threads_expired",
  "server_time_ms": 1793670001000,
  "error": {
    "code": "PAGE_TOKEN_EXPIRED",
    "message": "会话列表分页已过期，请从第一页重新查询",
    "retryable": false,
    "retry_after_ms": 0
  }
}
```

客户端结束该轮分页，丢弃该轮列表分页进度，以page_token=null重新请求第一页，并使用请求发出时的local_read_positions。该处理保留各会话的消息同步进度和本机已读位置。重新查询建立新的固定会话集合与顺序，客户端按role_id更新列表并隔离上一轮的迟到响应。

例如用户此时已经读到object_seq=100，使用新的X-Request-ID=req_threads_restart发送以下请求。后续响应字段与本节成功示例相同。

```json
{
  "page_size": 20,
  "page_token": null,
  "local_read_positions": [
    {
      "role_id": "role_1",
      "messages_sync_epoch": "msg-e1",
      "read_through_object_seq": "100"
    }
  ]
}
```

格式错误、被篡改或与分页参数不匹配的page_token返回HTTP 400 INVALID_ARGUMENT，客户端检查并修正分页参数。登录凭证过期按401 TOKEN_EXPIRED处理。

## 4 Scope位置查询：sync-state

`GET /auto-media/api/chat/roles/{role_id}/sync-state`。只接收公共Header和role_id路径参数，无请求正文、无业务查询参数。

| 响应字段 | 类型 | 出现条件 | 含义与使用规则 |
|---|---|---|---|
| `role_id` | ID | 必填 | 本状态所属会话，与路径一致。 |
| `scopes` | SyncHint数组 | 必填 | `messages` 和 `memories` 各一项，按 `scope` 识别。每项字段见 2.5。 |
| `current_object_seq` | UInt64String | 必填 | 当前会话的消息创建高水位，不包含额外阅读信息。 |
| `thread_revision` | UInt64String | 必填 | 会话展示状态版本，用于发现列表内容变化。 |

位置字段只用于发现变化。本接口是可选探测，客户端也可以直接调用read-batches。本次亲密度功能不扩展此接口，不接受`intimacy_score_version`查询参数，也不返回`intimacy_score`；亲密度仅按第5节在`messages:batchCreate`回包中交付。

`GET /auto-media/api/chat/roles/role_1/sync-state`，本次请求的 `X-Request-ID` 为 `req_state_1`。无请求正文。

响应（HTTP 200）：

```json
{
  "request_id": "req_state_1",
  "server_time_ms": 1793670000000,
  "role_id": "role_1",
  "scopes": [
    {
      "scope": "messages",
      "sync_epoch": "msg-e1",
      "server_max_sync_seq": "105",
      "min_valid_cursor": "0"
    },
    {
      "scope": "memories",
      "sync_epoch": "memory-e1",
      "server_max_sync_seq": "4",
      "min_valid_cursor": "0"
    }
  ],
  "current_object_seq": "100",
  "thread_revision": "105"
}
```

## 5 消息批量创建：messages:batchCreate

`POST /auto-media/api/chat/roles/{role_id}/messages:batchCreate`。role_id等公共路径字段见1。

请求字段：

| 字段路径 | 类型 | 必填/默认值 | 含义与使用规则 |
|---|---|---|---|
| `messages` | 对象数组 | 必填，至少1项 | 原子创建的消息，同一message_id不可重复；允许普通消息与已生成post候选混批。端侧来源按5.1申报，post候选按5.6校验，不因混批放宽绑定。 |
| `messages[].message_id` | ID | 必填 | 生成端首次持久化时确定的永久消息ID，也是创建幂等键。网络重试不变。 |
| `messages[].sender_type` | user或role | 必填 | 业务发言方；不等于生成设备。 |
| `messages[].sender_id` | ID | 必填 | user时等于认证用户，role时等于路径role_id。服务端校验，不接受任意发送者。 |
| `messages[].message_type` | 字符串枚举 | 必填 | text/image/audio/video/mixed；含义及与body的匹配规则见2。 |
| `messages[].body` | MessageBody | 必填 | 文本和媒体引用。body.text、body.attachments及附件字段逐项见2，至少有非空文本或有效附件。 |
| `messages[].created_at_ms` | TimestampMs | 必填 | 普通消息为生成端首次保存时间；post候选必须等于任务返回的created_at_ms（即generated_at_ms）。重试固定，不负责消息排序。 |
| `messages[].proactive_context` | ProactiveContextV1或null | 普通消息可省略/null；post候选必须非null | 仅允许完整post_link分支，并与生成任务候选逐字段一致；纳入消息幂等摘要，普通省略与null规范相同。不可上传亲密度分支或任意task/post引用。 |
| `messages[].metadata.context_provenance` | ContextProvenanceV1 | 普通端侧role按5.1；post候选可在上传时补充实际已知聊天来源 | 随消息ID冻结并参与幂等。post生成不要求消息ID；上传时可复用confirmed/pending_message_create声明实际参与content的聊天来源，服务端合并任务中的实际Memory来源，不能凭post上下文声称来源完整。 |
| `sync_positions` | 对象数组 | 可省略，默认[] | 本机已完成的 Scope 读取位置，供生成同步提示。独立于冻结消息及其业务幂等摘要。 |
| `sync_positions[].scope` | Scope | 每项必填 | 此位置所属数据类型，同Scope最多一项。 |
| `sync_positions[].sync_epoch` | OpaqueString或null | 每项必填；无基线为null | 本机该Scope的代次，用于判断提示是否要求重建。 |
| `sync_positions[].cursor` | UInt64String | 每项必填；无基线为"0" | 本机已完整应用的读取位置，不是本批待上传消息的Seq。 |
| `session_messages_sync_epoch` | OpaqueString 或 null | 可省略，默认 null | 当前会话已知的messages Epoch；不参与人设或亲密度请求版本比较，也不因不匹配将请求版本改为`"0"`。客户端仍按第5.2、5.4节校验响应所属会话与当前Epoch。 |
| `structured_information_version` | UInt64String | 可省略，默认 `"0"` | 当前会话内存中的网红人设版本。每次发送或重试时取当前值，无快照传 `"0"`。 |
| `intimacy_score_version` | UInt64String | 可省略，默认 `"0"` | 当前会话内存中的亲密度版本，与人设版本独立。每次发送或重试时从当前会话重新取值，不进入消息幂等摘要或Outbox冻结负载。 |

响应字段：

| 字段路径 | 类型 | 出现条件 | 含义与使用规则 |
|---|---|---|---|
| `results` | 对象数组 | 成功必填 | 每条消息的首次成功回执，按message_id关联，不仅依赖数组顺序。 |
| `results[].message_id` | ID | 每项必填 | 被确认的消息，关联请求中的消息。 |
| `results[].receipt_sync_epoch` | OpaqueString | 每项必填 | 该消息首次创建的messages Epoch，可能不同于服务端当前Epoch。 |
| `results[].object_seq` | UInt64String | 每项必填 | 首次分配的稳定消息创建顺序。它不直接把会话标读。 |
| `results[].sync_seq` | UInt64String | 每项必填 | 首次创建结果的同步序号，不保证仍是该对象当前状态，不直接推进本机cursor。 |
| `results[].server_created_at_ms` | TimestampMs | 每项必填 | 服务端首次接受该消息的时间，重试返回原时间。 |
| `messages_sync_epoch` | OpaqueString | 成功必填 | 当前服务端messages代次，与历史回执代次分开。 |
| `sync_hints` | SyncHint数组 | 成功必填，可为[] | 按本次请求重新计算的当前变化提示，不回放旧整个HTTP包。 |
| `structured_information` | StructuredInformationSnapshot | 已发布且版本不等时返回 | 比较请求中的人设版本与该角色当前版本，包括客户端版本过高的情况。字段见第 2.6 节，接收规则见第 5.2 节。 |
| `intimacy_score` | IntimacyScoreSnapshot | 普通用户已有已接受等级且版本不等时返回；ios-demo-test/zyy每次成功回包均返回 | 普通用户比较请求亲密度版本与当前用户、网红组合的已接受版本，包括请求版本过高的情况；演示账号跳过快照读取及版本比较，固定版本1、LV2。与消息Epoch无关；字段见2.6，应用与重试规则见5.4。 |

服务端对普通消息按sender_type确定origin：user为client_user、role为client_llm；带post上下文的候选必须先通过5.6的绑定与正文校验，才派生server_multimodal。新v3消息reply_to_message_id为null。object_seq、sync_seq、server_created_at_ms只在正式提交时分配；生成结果不提前占这些序号。

本接口不接受origin、reply_to_message_id、server_created_at_ms或顶层link_type/moment_id/card_id/post_id/task_id/触发等级。唯一的post元数据入口是messages[].proactive_context，服务端按5.6验证生成任务；不能凭其伪造其他服务端消息。results只确认本批实际上传消息（含正式提交的post候选），亲密度后台主动消息仍通过正式同步读取，不作为本接口额外生成结果混入results。

### 5.1 端侧回复的生成来源

生成时采集实际注入的消息和父记忆版本。`metadata.context_provenance` 出现时，以下字段均必填。

| 字段 | 类型 | 含义与规则 |
|---|---|---|
| `schema_version` | JSON 整数，固定 `1` | 来源声明结构版本。 |
| `capture_state` | `complete` / `partial` / `unavailable` | 分别表示完整采集、部分采集、无法采集。确实没有动态来源时可以声明 `complete` 并传 `sources: []`。 |
| `sources` | 对象数组 | 实际注入的消息和父记忆。只进入召回候选但未用于生成的记录无需申报。 |
| `sources[].source_scope` | `messages` / `memories` | 来源数据类别。 |
| `sources[].source_object_id` | `ID` | 消息 ID 或父记忆 ID。 |
| `sources[].source_epoch` | `OpaqueString` | `confirmed` 使用实际版本的 Epoch，`pending_message_create` 使用预计创建所在的消息 Epoch。 |
| `sources[].source_sync_seq` | 正 `UInt64String` 或 `null` | `confirmed` 必须是实际来源的正 `sync_seq`，`pending_message_create` 必须为 null。 |
| `sources[].ref_state` | `confirmed` / `pending_message_create` | 分别表示已确认版本、待绑定的消息创建。待绑定只允许 `messages`，服务端在原消息创建成功、幂等内容匹配且仍有效时绑定。 |

生成完成后将来源声明、回复正文和 `message_id` 一起冻结到 Outbox，重试原样发送。服务端绑定待确认来源时保留原冻结声明。格式无效的声明会被拒绝。兼容旧请求省略声明时，服务端记录为 `unavailable`。

完整且可验证的声明可参与自动记忆提取。完整声明中有合法待绑定引用或暂时未解析依赖时，等待服务端解析。部分、不可用或永久失效的来源仍可保留回复展示，回复不作为自动提取证据。普通端侧回复生成中或提交前发现依赖失效时按有效上下文重建；已generated的post消息仍按原文正常同步，来源失效仅影响其自动记忆提取资格，不得修改原回复后沿用任务ID，详见5.6/12.5。

未同步content并无可验证的消息ID，post任务不能伪造capture_state=complete。任务实际注入的Memory保留精确版本；客户端上传时可声明真实参与content的消息来源，包括同批pending创建。服务端对原声明、实际Memory和已知根进行校验与去重合并；仅在所有实际动态来源均已完整、可验证地覆盖时才可完整，否则为partial/unavailable，不自动把角色回复当作用户事实。未补声明不阻止满足其他校验的post候选写入，但影响其自动提取资格。

### 5.2 网红人设的回传、应用与重试

| 服务端资料与本次请求 | 成功响应中的 `structured_information` |
|---|---|
| 角色已授权，资料尚未入库 | 省略。 |
| 请求版本等于已发布版本 | 省略。 |
| 请求版本与已发布版本不同 | 返回 `{ "version": 当前版本, "data": 完整 persona_features }`，包括请求版本高于服务端的情形。 |
| 请求版本缺省或客户端无快照 | 按 `"0"` 比较。有已发布资料时返回完整快照。 |

服务端按数值比较请求人设版本与当前角色的人设版本，`session_messages_sync_epoch` 不参与这项比较。客户端将 `version` 和 `data` 一起保存在当前认证账号、角色和会话实例的内存中。退出会话、切换账号或角色、进程重启、确认消息 Epoch 变化或角色访问权限撤销时清空相应快照。人设和版本均不进入 SQLite、Outbox、消息幂等摘要或日志。

收到响应先处理消息回执。仅当响应仍属于当前账号、角色、会话实例且顶层 `messages_sync_epoch` 与当前已确认的消息 Epoch 一致，才处理快照。逐条 `receipt_sync_epoch` 只解释历史回执。较高版本原子替换，版本相同保留。较低版本仅在响应对应最近一次已发出的请求，且本地版本仍等于该请求所报版本时应用，作为服务端纠正。其余迟到低版本回包丢弃。字段省略时保留当前有效快照，`data: {}` 则整体替换为空。若本地已清空且回包省略字段，下次带 `"0"`。

人设查询暂时失败时，服务端返回可重试的 HTTP 503，消息可能已经提交。收到该错误或网络结果未知时，用原 `message_id` 和冻结消息内容重试，快照版本从当前会话重新取值。服务端回放首次消息回执，并按本次请求与当前资料重新比较人设。后台补发无匹配的活动会话时，只处理消息回执。

首次真实用户消息成功创建后，可以使用本次响应处理后的快照进行端侧生成。需要post候选时可走第12节服务端生成路径，但候选仍由客户端调用本接口正式提交。接口始终要求至少一条合法消息。

### 5.3 请求和响应示例

`POST /auto-media/api/chat/roles/role_1/messages:batchCreate`，本次请求的 `X-Request-ID` 为 `req_create_1`。示例同时补传一条已生成的端侧回复，它依赖同批尚未确认的用户消息。回包的人设供后续生成使用。

```json
{
  "messages": [
    {
      "message_id": "msg_101",
      "sender_type": "user",
      "sender_id": "user_1",
      "message_type": "text",
      "body": {
        "text": "帮我画一张日落",
        "attachments": []
      },
      "created_at_ms": 1793669999950
    },
    {
      "message_id": "msg_local_102",
      "sender_type": "role",
      "sender_id": "role_1",
      "message_type": "text",
      "body": {
        "text": "今天的晚霞真好看。",
        "attachments": []
      },
      "created_at_ms": 1793669999960,
      "metadata": {
        "context_provenance": {
          "schema_version": 1,
          "capture_state": "complete",
          "sources": [
            {
              "source_scope": "messages",
              "source_object_id": "msg_101",
              "source_epoch": "msg-e1",
              "source_sync_seq": null,
              "ref_state": "pending_message_create"
            }
          ]
        }
      }
    }
  ],
  "sync_positions": [
    {
      "scope": "messages",
      "sync_epoch": "msg-e1",
      "cursor": "105"
    }
  ],
  "session_messages_sync_epoch": "msg-e1",
  "structured_information_version": "0",
  "intimacy_score_version": "0"
}
```

响应：

```json
{
  "request_id": "req_create_1",
  "server_time_ms": 1793670000000,
  "results": [
    {
      "message_id": "msg_101",
      "receipt_sync_epoch": "msg-e1",
      "object_seq": "101",
      "sync_seq": "106",
      "server_created_at_ms": 1793669999990
    },
    {
      "message_id": "msg_local_102",
      "receipt_sync_epoch": "msg-e1",
      "object_seq": "102",
      "sync_seq": "107",
      "server_created_at_ms": 1793669999990
    }
  ],
  "messages_sync_epoch": "msg-e1",
  "sync_hints": [
    {
      "scope": "messages",
      "sync_epoch": "msg-e1",
      "server_max_sync_seq": "107",
      "min_valid_cursor": "0"
    }
  ],
  "structured_information": {
    "version": "1",
    "data": {
      "Biography": "喜欢记录城市街景的摄影创作者。",
      "Personality": ["好奇", "耐心"],
      "Speaking Style": {"Tone": "自然", "Sentence Length": "短句"},
      "Hobbies": ["街头摄影"]
    }
  },
  "intimacy_score": {
    "version": "1",
    "data": {
      "level": "LV2"
    }
  }
}
```

后续请求带 `"1"` 且服务端仍为 `"1"` 时，成功回包省略人设。若角色重建后发布空人设版本 `"2"`，请求带 `"1"` 时回传 `{"version":"2","data":{}}`。请求误带 `"9"` 时也回传版本 `"2"`，按第 5.2 节应用纠正。

### 5.4 亲密度快照的回传、应用与重试

演示回包例外优先：认证上下文的完整user_id大小写敏感地精确匹配`ios-demo-test`或`zyy`时，每次成功batchCreate（包括幂等重试成功）均返回`{"version":"1","data":{"level":"LV2"}}`。即使没有真实亲密度记录，或真实记录已有其他等级和版本，返回也相同；intimacy_score_version省略、"0"、"1"或任何其他合法值都不影响该字段的出现及取值。此分支跳过真实亲密度快照读取和版本比较，正常鉴权、角色/消息校验、版本字段语法校验及消息幂等规则仍适用。

此例外只作用于响应组装，不写入真实亲密度、不推进持久版本，也不登记或触发主动消息；后台仍按真实源数据监测和生成。两个演示账号之外的用户继续使用下述真实快照规则。

普通用户的回包亲密度按认证用户和路径网红维护，独立版本只反映本服务已接受的`level`。首次接受LV≥2时版本为`"1"`，以后只接受更高等级；一次监测从LV2直接变为LV5时只递增一次版本。源等级相同、降低或源记录被删除时不改变版本；后两种情况保留本服务最后接受的值并记录error。卡片调整、主动消息补发进度、失败重试及消息Epoch变化都不推进该版本。源库监测周期默认3分钟、可配置，普通用户回包读取本服务最后成功接受的状态，不保证包含刚发生而尚未监测到的源等级变化。

| 本服务亲密度状态与本次请求 | 成功响应中的`intimacy_score` |
|---|---|
| 认证user_id精确为ios-demo-test或zyy，任意真实状态及合法请求版本 | 每次成功回包均固定返回`{"version":"1","data":{"level":"LV2"}}`，不读取真实快照、不比较版本。 |
| 普通用户尚无已接受等级，包括尚未监测到LV≥2关系 | 省略，不返回占位LV2、LV1、null或空`data`。 |
| 普通用户请求版本等于当前已接受版本 | 省略，客户端保留当前有效快照。 |
| 普通用户请求版本与当前已接受版本不同 | 返回`{"version": 当前版本, "data": {"level": 当前已接受等级}}`，包括请求版本高于服务端的情况。 |
| 普通用户请求缺省版本或客户端没有快照 | 按`"0"`比较；有已接受等级时返回完整快照。 |

普通用户比较直接使用本次请求的`intimacy_score_version`，不因`session_messages_sync_epoch`缺失或不匹配改成`"0"`。消息幂等重试回放首次消息回执，但普通用户重新读取当前亲密度状态并按本次请求版本计算快照；演示账号仍固定返回版本1、LV2，不回放首次HTTP包中的旧亲密度。

客户端将亲密度`version`和完整`data`原子保存在当前账号、网红、会话实例及已确认messages Epoch绑定的内存中，不写SQLite、Outbox、消息幂等摘要或日志。退出会话、切换账号或网红、进程重启、确认messages Epoch变化或权限撤销时清空，下次真实消息上传带`"0"`。该客户端生命周期不会重置服务端亲密度版本。

收到响应先处理消息回执。只有响应仍属于当前账号、网红、会话实例，且顶层`messages_sync_epoch`等于当前已确认Epoch时，才独立应用亲密度快照；历史`receipt_sync_epoch`不决定快照归属。普通用户较高版本整体替换，版本相同保留；较低版本仅在响应对应最近一次已发出的请求，且本地版本仍等于该请求报告的版本时作为服务端纠正应用，其余迟到低版本丢弃。演示账号通过相同归属校验后，将返回的固定`version:"1"`和`data:{level:"LV2"}`整体应用；不因旧缓存版本相同或更高而保留其他等级，例如旧缓存为版本1、LV5时也替换为固定LV2。字段省略时保留当前有效值；若本地已清空而回包省略，下次真实上传继续带`"0"`。

普通用户动态亲密度状态读取暂时失败时返回可重试HTTP 503，消息可能已经提交；客户端用原`message_id`和冻结消息内容重试，并从当前会话重新取人设、亲密度请求版本。演示账号跳过该亲密度读取，其他消息/人设依赖错误仍正常处理。后台补传没有匹配活动会话时只处理消息回执，不把快照写入其他会话。源库监测故障不等于本地记录消失，已接受值继续有效。

当前亲密度快照与主动消息的历史触发信息分别处理。普通用户例如本服务已接受LV5、版本`"2"`，正在按顺序补发LV3和LV4，回包仍返回LV5；演示账号仍返回固定版本1、LV2。消息中的`proactive_context.intimacy_level`保持真实触发时的等级。客户端不能用历史触发等级覆盖当前快照或推算亲密度版本。

本期只通过`messages:batchCreate`获取该快照，不增加独立读取接口，不扩展`sync-state`、会话列表或Scope读取回包。接口仍要求至少一条真实合法消息，无上传消息时不调用空批次，也不通过同步到一条主动消息推导当前等级。

### 5.5 亲密度主动消息下行示例

下面是通过messages普通区间、内联读取或`messages:batchGet`取得的一条完整`SyncRecord`示例。`moment_id`是卡片任务ID，`card_id`是实际发布卡片ID；它们不是读取任务或本消息的身份。

```json
{
  "object_id": "c5d2172fbc654f8dba7bf8cf0b603d31",
  "object_seq": "103",
  "sync_seq": "108",
  "operation": "upsert",
  "content": {
    "message_id": "c5d2172fbc654f8dba7bf8cf0b603d31",
    "sender_type": "role",
    "sender_id": "role_1",
    "origin": "server_llm",
    "reply_to_message_id": null,
    "proactive_context": {
      "schema_version": 1,
      "trigger_type": "intimacy_level_up",
      "intimacy_level": "LV3",
      "moment_id": "theme_task_lv3_01",
      "card_id": "published_card_lv3_01",
      "link_type": "intimacy_card"
    },
    "message_type": "text",
    "body": {
      "text": "感觉我们又熟悉了一点，连今天的晚霞都想先和你分享。你那边的天空是什么颜色？",
      "attachments": []
    },
    "created_at_ms": 1793670000100,
    "server_created_at_ms": 1793670000100
  }
}
```

后续LV4消息正式发布后取得更大的`object_seq`，两个创建时间也分别大于这条LV3消息的对应时间。客户端仍以同步外层`object_seq`作为权威创建顺序，不能按等级重新编号，也不能因收到单条消息直接推进Scope游标。主动消息的历史元数据随正文保存和展示，当前亲密度快照仍只按5.4节获取。

### 5.6 服务端post候选的客户端确认写入

本节仅适用于client-commit-v3的post_link候选，不开放客户端创建亲密度或其他服务端来源。客户端收到generated后按业务已发布处理，将message对象原样持久化到Outbox用于服务端同步；首次上传前可附5.1的实际来源metadata，但不得改变候选七字段，最终上传负载首次冻结后连metadata也不再修改。首次上传可单条提交，也可与尚未同步的用户/角色消息混批，不要求先上传content中的聊天。

服务端先做认证、角色/归属和基础语法校验，再按下述顺序处理；候选当前状态等新业务前提不能挡住已有成功回执的重放：

1. post候选使用独立摘要域message-create-post-link-v1，覆盖原消息字段、精确proactive_context及原client metadata.context_provenance声明。普通消息继续原message-create-v1算法；其上下文省略/null规范成原本的无上下文形状，不改变旧摘要或使旧Outbox冲突。不能通过变化上下文规避幂等。
2. 在同一归属下优先查原message_create回执。同message_id、同摘要已提交则直接回放首次回执，即使任务已completed、post后续变更、Epoch已切换或消息已删除；不重新写入或复活消息。相同ID不同摘要仍为IDEMPOTENCY_CONFLICT。
3. 没有成功回执的首次写入，核对认证user/role、任务task_id、reply_message_id=message_id、post_id和五字段proactive_context；以持久generated_at_ms、冻结message及result_digest确认真实已生成结果，并验证当前messages Epoch/G等于任务内部冻结的目标代次。当前用户/角色下无任务用MULTIMODAL_TASK_NOT_FOUND；任务存在但绑定或冻结字段不符用MULTIMODAL_RESULT_MISMATCH。未曾产生候选用MULTIMODAL_RESULT_NOT_READY，queued/running可继续等待，生成前failed/cancelled不可重试伪造候选。后续取消标记不撤销已生成结果，不以当前reservation_token持有关系或占用是否存在作为上传前置；不校验待上传时长，不因超过原300秒生成期限拒绝原候选。
4. 核对sender_type=role、sender_id=role_id、message_type=text、attachments=[]，body.text及created_at_ms与冻结候选完全一致。不得修改、重新规范化、截断正文或自行更改时间。首次上传不再查询外部post源、Milvus或reranker，也不因post撤回/变化、Memory版本或根失效拒绝原消息；保留真实来源失效事实，只按已有规则限制后续自动记忆提取。
5. 在公共guard/fence保护的同一事务中写正式消息和message_create回执、更新任务completed、按固定user/role/post绑定确认sent，并完成既有消息身份、线程/未读、来源/语言及后台记忆处理责任。占用缺失、被改写、被另一任务reserved或已sent均不阻止原消息写入；post_link_states只承担去重，不是上传授权。事务内upsert/修复或确认永久sent事实，已有sent保留第一正式映射，各任务另保留自己的正式消息映射，不用占用变化触发重新生成。任何一步失败整批回滚，不先写消息再异步补sent。generated本身不分配正式同步序号。


客户端收到generated即按业务已发布处理。后续下列变化不会产生MULTIMODAL_RESULT_INVALID，也不改变原消息及上传ID：

| 候选已生成后的变化 | batchCreate行为 |
|---|---|
| 请求取消，或任务留有取消标记 | 以持久生成结果为准，原message正常写入；不因取消清候选或释放占用。 |
| post删除、撤回、归属或概述变化 | 按原固定post_id及冻结候选写入，不重新检索，不查询外部post源。 |
| 实际Memory版本或已知根失效 | 原消息正常写入，保留真实来源失效事实；依既有规则限制自动记忆提取，不重新生成。 |
| post占用丢失、持有关系变化或已sent | 按原task/reply/post绑定确认消息真实性，在正式事务内修复或确认sent，不因占用状态拒绝。 |

上述规则不放松身份/归属、冻结正文/时间/context、幂等摘要和内部目标messages Epoch/G校验；目标会话重置仍按12.5处理。

普通消息不得使用已预留的reply ID绕过上述分支；有绑定却不带原post上下文、无绑定却自造上下文、修改task/post/text/time均拒绝。旧message-v1由服务端发布的消息不是v3候选，不能通过本入口再次创建。

每批仍全部原子成功或失败：一条候选非法不会静默丢弃该条或部分写入其他消息。客户端保留所有Outbox原项，按错误定位处理无效候选后再提交合法消息；不得用新ID重复上传已经确认的消息。候选和用户消息在同批时，已有pending来源绑定继续按5.1规则工作。

上传超时、503、响应丢失或回包快照阶段失败均可能发生在事务提交后。此时只重试原batchCreate消息及冻结负载，查询任务/批量读回辅助确认；不重建生成任务、不更换reply ID、不因上传异常再生成普通回复。内部目标Epoch/G重置与上传使用同一guard/fence协调，提交结果未知时须先查证。取消只可阻止尚未持久产生候选的生成；已generated后不因取消、post/Memory失效或占用变化阻断原消息同步，也不先释放其占用。

post的created_at_ms来自服务端冻结的generated_at_ms，是可核验的生成时间；server_created_at_ms为本次首次正式提交时间，可能晚于生成时间，不要求两者相等。服务器不能仅因客户端传server_multimodal标签而信任任意时间。任务完成后通过batchGet/正式同步读取权威消息，不把Outbox候选当作已同步记录。

上传示例，replymsg_post_1及全部候选字段来自12.2的generated响应：

```json
{
  "messages": [
    {
      "message_id": "replymsg_post_1",
      "sender_type": "role",
      "sender_id": "role_1",
      "message_type": "text",
      "body": {
        "text": "想放松一下的话，可以看看我之前分享的海边散步。你更喜欢静静看海，还是沿着海岸走走？",
        "attachments": []
      },
      "created_at_ms": 1795410010000,
      "proactive_context": {
        "schema_version": 1,
        "trigger_type": "post_link",
        "link_type": "post",
        "post_id": "post_123",
        "task_id": "task_post_1"
      }
    }
  ],
  "session_messages_sync_epoch": "msg-e1",
  "structured_information_version": "0",
  "intimacy_score_version": "0"
}
```

MULTIMODAL_ENABLED停止新生成时，已经generated且仍合法有效的候选仍可首次batchCreate提交；已有成功回执始终可回放。生成开关不是撤销已接受候选上传责任的依据。

其中session_messages_sync_epoch仍按本接口原规则使用，不回填到生成请求。post候选的内部目标代次校验独立执行，即使客户端省略该可选字段，也不能将旧候选首次写入新Epoch。成功响应复用5.3的逐条results结构，没有额外的“已生成但未写入”回执。

## 6 单次变更与状态查询：mutations

`POST /auto-media/api/chat/roles/{role_id}/scopes/messages/mutations`。一次请求删除一条消息，对应一个稳定的 `mutation_id`。首期长期记忆由服务端提取和发布，客户端对 `memories` 请求创建、编辑或删除时返回 422 `OPERATION_NOT_SUPPORTED`。

选中消息及所有以该消息为根来源的记忆在同一事务写入各自 Scope 的墓碑，其余消息及回复继续展示。实际来源包括经过其他回复或记忆传递的依赖。首期事务保护基线为最多 100 条关联记忆、预计写入不超过 4 MiB、应用事务预算 10 秒。超过预算时整笔拒绝并返回 409 `DELETE_REQUIRES_MAINTENANCE`，客户端停止自动重试。成功删除以 HTTP 200 的 `published` 回执确认。

| 请求字段 | 类型 | 必填条件 | 含义与使用规则 |
|---|---|---|---|
| `mutation_id` | ID | 必填 | 客户端生成的唯一操作 ID；同一次操作重试不变。用于识别处理过的请求，不代替版本前提。 |
| `origin_sync_epoch` | OpaqueString | 必填 | 操作依据的数据代次，冻结后不自动换成新Epoch重发。 |
| `object_id` | ID | 必填 | 待删除的 `message_id`。 |
| `operation` | `delete` | 必填 | 消息删除。 |
| `base_sync_seq` | UInt64String | 必填 | 客户端依据的消息 `sync_seq`。与当前状态不符时冲突，已是墓碑时可返回 `already_deleted=true`。 |

`GET /auto-media/api/chat/roles/{role_id}/scopes/messages/mutations/{mutation_id}` 查询删除结果，仅需路径参数和公共请求头。POST 和 GET 共用以下响应结构。

| 响应字段路径 | 类型 | 出现条件 | 含义与使用规则 |
|---|---|---|---|
| `mutation_id` | ID | 必填 | 当前操作资源ID，与原请求相同。 |
| `status` | `published` / `failed` | 必填 | `published` 表示删除已持久提交。查询到 `failed` 表示操作已终止。 |
| `receipt_sync_epoch` | OpaqueString | published必填，其他省略 | 首次成功处理时的Scope代次。历史回执不作为新Epoch的业务数据写入。 |
| `result` | 对象 | published必填，其他省略 | 首次成功结果，不包含所有历史正文。 |
| `result.object_id` | ID | result出现时必填 | 实际被操作的对象。 |
| `result.sync_seq` | UInt64String | result出现时必填 | 本操作发布的对象或墓碑版本；already_deleted时为已存在墓碑的位置。 |
| `result.operation` | `delete` | result 出现时必填 | 已确认执行的操作。 |
| `result.already_deleted` | boolean | delete可选，省略为false | true表示对象已是墓碑，本次确认删除目标已达成，未分配第二个删除Seq。 |
| `sync_hints` | SyncHint数组 | 必填，可为[] | 当前Scope位置提示，不是该操作首次结果的版本替代品。 |
| `cascade_sync_barrier` | 对象数组 | `published` 删除返回 | 此次关联记忆清理的目标，见第 11.3 节。没有受影响的记忆时可为 `[]`。 |
| `cascade_sync_barrier[].scope` | `memories` | 每项必填 | 需追赶的长期记忆 Scope。 |
| `cascade_sync_barrier[].sync_epoch` | OpaqueString | 每项必填 | required_sync_seq所属代次，不能跨Epoch直接比较数值。 |
| `cascade_sync_barrier[].required_sync_seq` | UInt64String | 每项必填 | 此次清理在该Scope提交后的目标水位。必须靠实际同步达到，不能直接赋给cursor。 |
| `deleted_counts` | 对象 | published消息删除返回 | 各Scope本次实际新删除的记录数，不是未读数。 |
| `deleted_counts.messages` | Count | deleted_counts出现时必填 | 选中消息实际新删除数量，普通单消息操作为0或1，不包括保留回复。 |
| `deleted_counts.memories` | Count | 同上 | 本次新清理的关联记忆数，可为 0。 |
| `error` | Error | status=failed时必填 | 操作失败原因，内部字段见1.5。GET可以HTTP200成功返回一个status=failed的资源。 |

网络结果未知时查询或原样重试原 `mutation_id`。已提交回执重放首次结果及其清理目标，当前 `sync_hints` 单独处理。客户端收到有效删除后，同事务保存消息墓碑、已知删除标记和待清理屏障，并按第 11.3 节处理受影响的检索与生成。

`POST /auto-media/api/chat/roles/role_1/scopes/messages/mutations`，本次请求的 `X-Request-ID` 为 `req_mutation_1`。请求正文如下。

```json
{
  "mutation_id": "mut_1",
  "origin_sync_epoch": "msg-e1",
  "object_id": "msg_20",
  "operation": "delete",
  "base_sync_seq": "20"
}
```

响应：

```json
{
  "request_id": "req_mutation_1",
  "server_time_ms": 1793670000000,
  "mutation_id": "mut_1",
  "status": "published",
  "receipt_sync_epoch": "msg-e1",
  "result": {
    "object_id": "msg_20",
    "sync_seq": "108",
    "operation": "delete",
    "already_deleted": false
  },
  "sync_hints": [
    {
      "scope": "messages",
      "sync_epoch": "msg-e1",
      "server_max_sync_seq": "108",
      "min_valid_cursor": "0"
    },
    {
      "scope": "memories",
      "sync_epoch": "memory-e1",
      "server_max_sync_seq": "5",
      "min_valid_cursor": "0"
    }
  ],
  "cascade_sync_barrier": [
    {
      "scope": "memories",
      "sync_epoch": "memory-e1",
      "required_sync_seq": "5"
    }
  ],
  "deleted_counts": {
    "messages": 1,
    "memories": 1
  }
}
```

`GET /auto-media/api/chat/roles/role_1/scopes/messages/mutations/mut_1`，本次请求的 `X-Request-ID` 为 `req_mutation_2`。无请求正文。

响应（HTTP 200）：

```json
{
  "request_id": "req_mutation_2",
  "server_time_ms": 1793670000000,
  "mutation_id": "mut_1",
  "status": "published",
  "receipt_sync_epoch": "msg-e1",
  "result": {
    "object_id": "msg_20",
    "sync_seq": "108",
    "operation": "delete",
    "already_deleted": false
  },
  "sync_hints": [
    {
      "scope": "messages",
      "sync_epoch": "msg-e1",
      "server_max_sync_seq": "108",
      "min_valid_cursor": "0"
    },
    {
      "scope": "memories",
      "sync_epoch": "memory-e1",
      "server_max_sync_seq": "5",
      "min_valid_cursor": "0"
    }
  ],
  "cascade_sync_barrier": [
    {
      "scope": "memories",
      "sync_epoch": "memory-e1",
      "required_sync_seq": "5"
    }
  ],
  "deleted_counts": {
    "messages": 1,
    "memories": 1
  }
}
```

## 7 创建逻辑 batch 并返回本轮参数：read-batches

`POST /auto-media/api/chat/roles/{role_id}/scopes/{scope}/read-batches?include_data=true|false`，成功HTTP 200。这一步已经确定范围并建议跨度，不再另发“查询边界”请求。

| 请求字段/查询参数 | 类型与出现条件 | 含义、来源及约束 |
|---|---|---|
| `include_data` | URL true/false；省略默认false | 只影响本次响应；不放JSON正文，只出现一次，非法/重复参数400。满足小范围条件时附带inline_result。 |
| `mode` | recent/history/full/incremental，正文必填 | 确定扫描轴及位置规则。 |
| `sync_epoch` | OpaqueString或null，字段必填 | recent/full首次无基线可null；incremental/history必须匹配当前Scope；指定旧值也不静默换代。 |
| `cursor` | UInt64String；full/inc必填，其他省略 | full固定"0"；inc为已完整覆盖的C，不取回执或对象MAX替代。 |
| `base_cursor` | UInt64String；history必填，其他省略 | 本机当前实时C，保护删除接续，不参与历史选择。 |
| `before_object_seq` | UInt64String；history必填，其他省略 | 历史独占上界，取之前服务端根锚点；边界对应消息被删不影响数值条件。 |
| `limit` | JSON整数，1～500。recent/history 默认50，incremental 可省，full 省略 | recent/history 限制有效消息条数。incremental 省略时覆盖全部增量，传入时按本节规则计算 H，包含墓碑。 |
| `preferred_partition_span` | JSON整数；可省，1～200 | 客户端偏好的序号跨度；服务端可调整后返回建议值，不构成读取结果或完成数量承诺。 |
| `require_fresh_provenance_anchor` | boolean，恢复用，可省略 | `memories` 的缓存损坏且无法可信读取本机删除约束时，恢复 full 请求传 `true`。服务端在与来源删除和模型切换串行的事务中取得新锚点，随后按第 11.3 节追赶并核对屏障。 |

`recent` 和 `history` 只允许 `messages`。`memories` 使用 `full` 或 `incremental`，默认建议跨度为 5，可因记录大小调小。所有成功 JSON 都有公共 `request_id` 和 `server_time_ms`，业务字段直接位于顶层。

| 响应字段 | 类型/出现条件 | 含义及使用规则 |
|---|---|---|
| `role_id` | ID，必填 | 当前会话，与路径一致。 |
| `scope` | Scope，必填 | 当前数据类别；所有序号仅在此Scope/Epoch内比较。 |
| `mode` | 四种模式枚举，必填 | 回显读取语义，后续区间保持不变。 |
| `sync_epoch` | OpaqueString，必填 | 本次一致性读的正式数据世代。 |
| `base_cursor` | UInt64String或null，必填 | recent=null；full="0"；inc/history=C。 |
| `source_max_cursor` | UInt64String，必填 | 本轮同步上界H。增量传 limit 时可能低于当前已提交水位，计算规则见本节。本轮接受后固定，不是记录MAX或H时刻副本。 |
| `creation_boundary_seq` | UInt64String，必填 | 本轮创建高水位 B，表示已分配的最大 object_seq。 |
| `commit_cursor` | UInt64String或null，必填 | recent/full/inc目标H；history=null。仅客户端完整覆盖、合并及适用索引完成后提交。 |
| `lower_exclusive` | UInt64String，必填 | 待查询范围的排除下界。若返回inline_result，此值等于顶层upper_inclusive，表示待查询范围为空。 |
| `upper_inclusive` | UInt64String，必填 | 待查询范围的包含上界，等于原根范围的包含上界。未返回inline_result时，两项顶层边界就是原根范围。 |
| `suggested_partition_span` | JSON整数，必填，1～200 | 划分叶区间的建议序号宽度；可采用更小合法值，改变建议不改变根C/H/B或完成条件。 |
| `next_before_object_seq` | UInt64String或null，仅recent/history必填 | 整个根窗口发布后使用的下一历史独占锚点；无更早消息为null，不能用子区间MIN重算。 |
| `has_more_history` | boolean，仅recent/history必填 | 同次读视图判断根窗口之前是否还有有效消息；后来删除可使后续历史页为空。 |
| `inline_result` | RecordsPayload，可选 | `include_data=true`、根宽大于 0 且不超过建议跨度、完整 `RecordsPayload` 不超过 262144 字节时可返回。覆盖整个根范围，`complete=true`。内层上下界为原根范围，外层上下界相等。内含第 8 节的记录及元数据。 |

重试创建可能得到不同的 H 或范围。客户端通过request_id关联本次HTTP请求，并仅为本地任务接受一份仍适用的创建响应。无内联时按顶层范围细分；有内联时校验并完整保存内层范围及记录，保留外层的提交游标和历史锚点。

增量创建请求省略 limit 时，本轮 H 取同一读视图的当前已提交水位，范围覆盖全部 (C,H]，包含删除墓碑。客户端通过内联或按区间查询取得整轮数据，单次区间仍适用容量上限。

增量传 limit=N 时，按 sync_seq 升序选择至多 N 条当前记录，删除记录也计入。如果还有更多记录，本轮 H 为第 N 条的 sync_seq。否则 H 为当前水位。下一轮从本轮 commit_cursor 继续。带 limit 的增量不会扩大消息已读确认边界。

为写全消息体，以下用recent(limit=2)演示；最近20条采用limit=20，满足内联条件时同样一次请求拿完。此前创建1～100、删除1～5，所以H=105、B=100。

`POST /auto-media/api/chat/roles/role_01/scopes/messages/read-batches?include_data=true`：

```json
{
  "mode": "recent",
  "sync_epoch": null,
  "limit": 2,
  "preferred_partition_span": 200
}
```
```json
{
  "request_id": "req_batch",
  "server_time_ms": 1793577600000,
  "role_id": "role_01",
  "scope": "messages",
  "mode": "recent",
  "sync_epoch": "messages-epoch-A",
  "base_cursor": null,
  "source_max_cursor": "105",
  "creation_boundary_seq": "100",
  "commit_cursor": "105",
  "lower_exclusive": "100",
  "upper_inclusive": "100",
  "suggested_partition_span": 200,
  "next_before_object_seq": "99",
  "has_more_history": true,
  "inline_result": {
    "role_id": "role_01",
    "scope": "messages",
    "mode": "recent",
    "sync_epoch": "messages-epoch-A",
    "base_cursor": null,
    "source_max_cursor": "105",
    "lower_exclusive": "98",
    "upper_inclusive": "100",
    "record_count": 2,
    "related_memory_barrier": {
      "scope": "memories",
      "sync_epoch": "memory-e1",
      "required_sync_seq": "4"
    },
    "records": [
      {
        "object_id": "msg_99",
        "object_seq": "99",
        "sync_seq": "99",
        "operation": "upsert",
        "content": {
          "message_id": "msg_99",
          "sender_type": "user",
          "sender_id": "user_01",
          "origin": "client_user",
          "reply_to_message_id": null,
          "proactive_context": null,
          "message_type": "text",
          "body": {
            "text": "周末去哪里玩？",
            "attachments": []
          },
          "created_at_ms": 1793577598000,
          "server_created_at_ms": 1793577598020
        }
      },
      {
        "object_id": "msg_100",
        "object_seq": "100",
        "sync_seq": "100",
        "operation": "upsert",
        "content": {
          "message_id": "msg_100",
          "sender_type": "role",
          "sender_id": "role_01",
          "origin": "client_llm",
          "reply_to_message_id": null,
          "proactive_context": null,
          "message_type": "text",
          "body": {
            "text": "去公园走走吧。",
            "attachments": []
          },
          "created_at_ms": 1793577599000,
          "server_created_at_ms": 1793577599020
        }
      }
    ],
    "complete": true
  }
}
```

顶层待查询范围为(100,100]，内联已完整覆盖(98,100]。仅完整保存这两条并发布后才提交105；历史下一锚点99，后台预取不标读。原根宽为0时省略inline_result，按空范围完成规则处理。若本例使用默认include_data=false，则省略inline_result，顶层返回(98,100]，再按第8节查询。

## 8 完整区间查询：records:query（普通模式）

`POST /auto-media/api/chat/roles/{role_id}/scopes/{scope}/records:query`，成功HTTP 200。用于完整查询一个区间。

| 请求字段 | 类型/出现条件 | 含义与约束 |
|---|---|---|
| `mode` | 四种普通模式枚举，必填 | recent/history/full使用object_seq，incremental使用sync_seq；不接收客户端另传扫描轴。 |
| `sync_epoch` | OpaqueString，必填 | 客户端该轮已接受的Epoch。 |
| `base_cursor` | UInt64String或null，字段必填 | recent=null、full="0"、inc/history=C，必须与本轮上下文一致。 |
| `source_max_cursor` | UInt64String，必填 | 固定根H；不是请求发出时自行读取的新高水位。 |
| `lower_exclusive` | UInt64String，必填 | 此次叶区间排除下界。 |
| `upper_inclusive` | UInt64String，必填 | 此次叶区间包含上界；普通请求宽度须1～200，空根不发区间请求。 |

区间必须非空且在本轮范围内。`incremental` 须满足 `C <= lower_exclusive < upper_inclusive <= H`。沿用创建响应的 Epoch、C、H。服务端在同一读视图校验位置及查询正文。`incremental` 检查原 C，`full` 检查固定 H，低于当前 `min_valid_cursor` 时返回 409 `SYNC_CURSOR_EXPIRED`。读源尚未追到 H 时等待、切换读源或返回可重试的 503。

服务端以请求中的显式范围及上下文执行查询。客户端通过 `request_id` 关联本地任务，保存前核对角色、Scope、模式、Epoch、C、H 和上下界，并确认任务、缓存代次及当前叶区间仍有效。`memories` 的摘要和问句向量按本地已验收的模型约定校验。

RecordsPayload是普通响应剔除request_id和server_time_ms后的记录对象。inline_result完整复用它。


| 响应业务字段（也解释inline_result内同名字段） | 类型/出现条件 | 含义及约束 |
|---|---|---|
| `role_id` | ID，必填 | 当前会话，必须与请求相同。 |
| `scope` | Scope，必填 | 当前数据类别，必须与请求相同。 |
| `mode` | 四种模式枚举，必填 | 回显请求；决定范围轴。 |
| `sync_epoch` | OpaqueString，必填 | 与已接受上下文相同。 |
| `base_cursor` | UInt64String或null，必填 | 按模式回显C、0或null。 |
| `source_max_cursor` | UInt64String，必填 | 回显固定H，不用此次可能更大的实际读源水位替换。 |
| `lower_exclusive` | UInt64String，必填 | 本次请求叶区间的精确下界。 |
| `upper_inclusive` | UInt64String，必填 | 本次请求叶区间的精确上界。 |
| `record_count` | 非负JSON整数，必填 | 本次records长度，<=200；可0，不是全轮总数。 |
| `records` | SyncRecord数组，必填 | 完整当前对象/墓碑；嵌套object_id/object_seq/sync_seq/operation/content/embedding及消息正文、附件见2。upsert都有外层object_seq，delete只有三个基础字段。 |
| `complete` | boolean，必填且恒true | 本次数值区间已完整查询，不是根任务已经完成；无内部page_token或部分成功。 |
| `related_memory_barrier` | 对象，`messages` 的区间及内联结果返回 | 同一读视图采样的关联记忆清理目标，供其他设备收到消息删除后追赶。处理见第 11.3 节。 |
| `related_memory_barrier.scope` | `memories`，对象出现时必填 | 目标 Scope。 |
| `related_memory_barrier.sync_epoch` | OpaqueString，对象出现时必填 | 目标所属记忆 Epoch。 |
| `related_memory_barrier.required_sync_seq` | UInt64String，对象出现时必填 | 需要实际同步并清理索引达到的位置。 |

普通 `RecordsPayload` 的未压缩紧凑 UTF-8 JSON 上限为 1048576 字节，计入记录和响应元数据。宽度或条数超过 200，或载荷超限时，返回 413 `PARTITION_TOO_LARGE`。客户端按建议在同一事务中用更小叶区间替换原叶，并拒绝原叶迟到回包。单条 `SyncRecord` 上限为 524288 字节，宽度为 1 仍超限时按 422 `OBJECT_TOO_LARGE` 处理。

独立示例中，已有缓存 C=100，创建响应返回 H=105，客户端先读 (100,102]。

```json
{
  "mode": "incremental",
  "sync_epoch": "messages-epoch-A",
  "base_cursor": "100",
  "source_max_cursor": "105",
  "lower_exclusive": "100",
  "upper_inclusive": "102"
}
```
```json
{
  "request_id": "req_range",
  "server_time_ms": 1793577600000,
  "role_id": "role_01",
  "scope": "messages",
  "mode": "incremental",
  "sync_epoch": "messages-epoch-A",
  "base_cursor": "100",
  "source_max_cursor": "105",
  "lower_exclusive": "100",
  "upper_inclusive": "102",
  "record_count": 2,
  "related_memory_barrier": {
    "scope": "memories",
    "sync_epoch": "memory-e1",
    "required_sync_seq": "5"
  },
  "records": [
    {
      "object_id": "msg_1",
      "sync_seq": "101",
      "operation": "delete"
    },
    {
      "object_id": "msg_2",
      "sync_seq": "102",
      "operation": "delete"
    }
  ],
  "complete": true
}
```

本次只完成 `(100,102]`。其余区间也完成后才可提交消息游标 `"105"`。同时保存这两条消息的删除标记和记忆清理目标，实际追赶 `memories` 的 `memory-e1` 至 `"5"` 并核对索引。

以下413是Memory区间(0,5]因完整对象较大超限的示例；建议只是提示，实际每次仍按上限校验：

```json
{
  "request_id": "req_range_large",
  "server_time_ms": 1793577600000,
  "error": {
    "code": "PARTITION_TOO_LARGE",
    "message": "完整区间结果超限，请缩小区间重新查询",
    "retryable": false,
    "retry_after_ms": 0,
    "details": {
      "scope": "memories",
      "lower_exclusive": "0",
      "upper_inclusive": "5",
      "suggested_partition_span": 2,
      "suggested_ranges": [
        {
          "lower_exclusive": "0",
          "upper_inclusive": "2"
        },
        {
          "lower_exclusive": "2",
          "upper_inclusive": "4"
        },
        {
          "lower_exclusive": "4",
          "upper_inclusive": "5"
        }
      ]
    }
  }
}
```

suggested_ranges 最多 200 项。建议区间应按序无重叠、无缺口地覆盖原范围。只收到建议跨度时，可自行细分。

## 9 增量创建示例

请求 `POST /auto-media/api/chat/roles/role_01/scopes/messages/read-batches`。客户端从 100 继续同步，省略 `limit`。

```json
{
  "mode": "incremental",
  "sync_epoch": "messages-epoch-A",
  "cursor": "100"
}
```

假设当前已提交水位为 160，创建高水位为 155。本轮覆盖整个 `(100,160]`。

```json
{
  "request_id": "req_inc_batch",
  "server_time_ms": 1793577600000,
  "role_id": "role_01",
  "scope": "messages",
  "mode": "incremental",
  "sync_epoch": "messages-epoch-A",
  "base_cursor": "100",
  "source_max_cursor": "160",
  "creation_boundary_seq": "155",
  "commit_cursor": "160",
  "lower_exclusive": "100",
  "upper_inclusive": "160",
  "suggested_partition_span": 20
}
```

客户端按建议跨度读取三个区间，完整发布后推进到 160。若同样条件下传 `limit=2`，且可查询的记录序号为 110、130、160，本轮的 `source_max_cursor`、`upper_inclusive` 和 `commit_cursor` 均为 130，下一轮从 130 继续。

### 历史创建示例

调用 `POST /auto-media/api/chat/roles/role_01/scopes/messages/read-batches`，使用上一页返回的历史锚点。请求和响应如下。

```json
{
  "mode": "history",
  "sync_epoch": "messages-epoch-A",
  "base_cursor": "105",
  "before_object_seq": "99",
  "limit": 2,
  "preferred_partition_span": 200
}
```

```json
{
  "request_id": "req_history",
  "server_time_ms": 1793577602000,
  "role_id": "role_01",
  "scope": "messages",
  "mode": "history",
  "sync_epoch": "messages-epoch-A",
  "base_cursor": "105",
  "source_max_cursor": "105",
  "creation_boundary_seq": "100",
  "commit_cursor": null,
  "lower_exclusive": "96",
  "upper_inclusive": "98",
  "suggested_partition_span": 200,
  "next_before_object_seq": "97",
  "has_more_history": true
}
```

### Memory 全量创建示例

调用 `POST /auto-media/api/chat/roles/role_01/scopes/memories/read-batches`。首次读取可以传 null Epoch。请求和响应如下。

```json
{
  "mode": "full",
  "sync_epoch": null,
  "cursor": "0",
  "preferred_partition_span": 2
}
```

```json
{
  "request_id": "req_memory_batch",
  "server_time_ms": 1793577610000,
  "role_id": "role_01",
  "scope": "memories",
  "mode": "full",
  "sync_epoch": "memory-e2",
  "base_cursor": "0",
  "source_max_cursor": "5",
  "creation_boundary_seq": "3",
  "commit_cursor": "5",
  "lower_exclusive": "0",
  "upper_inclusive": "3",
  "suggested_partition_span": 2
}
```

按 `(0,2]`、`(2,3]` 查询完整范围。第一个叶区间请求 `POST /auto-media/api/chat/roles/role_01/scopes/memories/records:query` 的正文如下。

```json
{
  "mode": "full",
  "sync_epoch": "memory-e2",
  "base_cursor": "0",
  "source_max_cursor": "5",
  "lower_exclusive": "0",
  "upper_inclusive": "2"
}
```

两叶完整覆盖并核对索引后，候选缓存从 `"5"` 发起至少一轮省略 `limit` 的增量读取，再按第 11.1 节激活。

## 10 四种普通模式对照

| 模式 | batch请求及根范围 | 普通records请求的base_cursor | 完成后的动作 |
|---|---|---|---|
| recent，仅messages | limit=N；有效最近窗口(L-1,U]，按object_seq | null，H使用已接受根H | 替换新尾窗、提交H并保存根历史锚点，不声称旧历史已下载。 |
| history，仅messages | base_cursor=C、before_object_seq；有效旧窗口(L-1,U] | 原实时C | 仅合并旧历史和推进根锚点；不改实时cursor、确认尾部或已读。 |
| full | cursor="0"；所有Scope按object_seq，范围(0,B] | "0" | 完整替换/构建候选，根覆盖后从H追赶。 |
| incremental | cursor=C；按sync_seq，范围(C,H]，含墓碑 | 原C | 全部根范围覆盖后才处理H；移到>H的当前版本后续追赶。 |

recent/history未选中消息时根范围为(0,0]且历史锚点null/false；full只在B=0时自然为空，B>0的已删对象范围仍可能形成空叶；inc在C=H时为(C,C]。原根为空时无需发records请求，仍执行适用的替换或位置提交。有inline_result时，顶层零宽仅表示无需补发区间请求，须完整应用内联原根后再执行对应提交。普通空叶records=[]且complete=true只证明该叶覆盖。

仅当所有区间完整覆盖本轮范围后，才使用 commit_cursor。full 的记录可能带有大于 H 的 sync_seq，对象版本仍按 sync_seq 比较。

## 11 长期记忆同步与本地检索

### 11.1 同步、持久化与就绪

1. 打开角色聊天、回到前台或收到该角色 `memories` 更新提示时异步同步，活跃角色每 30 秒补查。首次 `full`，随后从本地已提交游标 C 做 `incremental`。消息发送或补传回执只触发相关读取，游标按实际覆盖推进。
2. 按认证账号、角色、Epoch 和缓存代次隔离。SQLite 同事务保存当前父记录、完整摘要及问句向量、根来源关联、最高删除版本和叶区间完成状态。较新 upsert 整体替换父记录及全部派生项，delete 按父 ID 移除各视图并保留最高墓碑。若使用外部索引，同事务登记精确新增和淘汰视图 ID，重启后可幂等重放。
3. 普通增量首次修改时标记 `updating` 并暂停该 Scope 检索。全部叶区间完整覆盖且索引的父版本、完整视图身份核对通过后提交本轮 H。检索就绪还要求本地模型有效、同 Epoch 删除目标均已达到、旧 Epoch 删除目标已通过重建解决。带 `limit` 的增量即使已提交本轮 H，若仍未达到目标则保持 `updating` 并继续追赶。
4. 首次同步和重建在候选缓存完成 `full`、至少一轮省略 `limit` 的 `incremental`、本地模型探针、索引与删除屏障核对后原子激活。确认旧 Epoch 失效时停用旧缓存并取消旧读取任务，按代次清理旧记录及索引，保留 Outbox 和仍需核验的删除约束。跨 Epoch 不比较 Seq。向量空间不兼容时由发布流程先提供对应应用包，并通过版本门禁或独立入口保证兼容。

### 11.2 本地向量生成与使用

应用预装与服务端同源且完成跨端对照的 BGE-M3 dense 编码器，使用安装包内的模型和预处理标识。端侧只对当前用户文本编码，摘要和发散问句直接使用下行的 1024 维向量。离线可使用已就绪的缓存。模型缺失、向量校验失败或索引不可用时，停用长期记忆检索，保留待处理区间和当前游标，回复可使用近期对话继续。

用同一查询向量分别查摘要 Top 20 和问句 Top 100。回查 SQLite 验证账号、角色、缓存代次、Epoch、模型和当前父版本，过滤墓碑和已知失效根。按父 ID 去重取有效视图的最高余弦分，并列按 `memory_id` 排序。生产环境使用已验收的检索配置，按其阈值选最多 20 个父候选，再按预算注入最多 4 条完整摘要及其标题、可用时间。缺少已验收配置时使用近期对话生成。当前用户输入只注入一次，冻结实际注入的来源版本，见第 5.1 节。

### 11.3 删除屏障与损坏恢复

- 发起设备使用 `cascade_sync_barrier`，其他设备在消息区间或内联结果收到删除时使用同响应的 `related_memory_barrier`。同一个 SQLite 事务应用消息删除、记录 `known_deleted` 和待清理屏障。立即按根消息反查屏蔽受影响的记忆，并取消依赖这些来源的未完成生成。来源不足以安全判断时暂停该角色的记忆查询。仅未加载消息或仅提示水位增加时，继续按正常同步处理。
- 同 Epoch 的清理目标取最大值。正文和全部摘要及问句视图实际追赶并完成索引清理后解除屏障。目标 Epoch 已失效时，在观察到删除后新建当前 Epoch 的 full 和增量，完成核对再恢复。若缺少可用屏障，后续记忆读取视图至少要在因果上覆盖已观察到的删除，才可恢复检索。
- 缓存损坏时隔离旧缓存并使旧任务失效。能够可信读取的删除约束经完整性校验后迁移到新缓存。无法可信读取约束时停用记忆检索，在线发起 `memories` full，在第 9 节的全量请求正文中加 `"require_fresh_provenance_anchor": true`，再完成至少一轮省略 `limit` 的增量以及模型、索引和新屏障核对。断网期间继续使用近期对话。Outbox 独立保留，迟到回包和旧备份不能直接激活。

## 12 服务端多模态任务

本节为2026-11-24用户确认并已接入代码的client-commit-v3合同；目标部署须先升级Schema及索引。服务端仅检索和生成候选；客户端通过batchCreate正式提交消息，详见5.6。创建不要求聊天先同步，不接受来源消息ID列表。旧message-v1恢复边界见12.6；此前仅设计的aggregate-v2已被本节替代，不能当作已上线接口。

### 12.1 创建请求与检索决策

`POST /auto-media/api/chat/roles/{role_id}/multimodal-tasks`。

| 请求字段 | 类型 | 必填/默认 | 含义与规则 |
|---|---|---|---|
| content | string | 必填 | 客户端整合的最近聊天原文，可包含尚未同步的多条user/role发言；至少一个非空白字符，最多12000个Unicode码点且UTF-8不超过49152字节，另受实际模型token预算约束。保留精确内容与换行，不trim、Unicode归一化或静默截断。 |
| reply_message_id | ID | 必填 | 客户端首次持久保存本轮时生成的稳定回复ID，后续候选和batchCreate的message_id均为此值。相同轮次重试保持ID和content不变，不使用已有消息或其他任务占用的ID。 |
| task_type | string | 必填 | 固定post_link。 |
| options | JSONObject | 可省略，默认{} | 只接受空对象；客户端不另传post、摘要开关、模型、阈值或回复语言。 |

不要求source_messages、input_message_id、input_sync_seq或messages_sync_epoch，提交这些旧字段/混用合同返回400 INVALID_ARGUMENT。content不是新聊天消息，服务端不会将它写入messages，不据此创建本轮输入消息ID。任务登记可以初始化必要的会话元数据并内部冻结目标messages Epoch，但不创建会话活动、不写消息、不分配object_seq/sync_seq。上传时按内部目标代次校验，客户端不必先读取或提供Epoch才能生成。

业务幂等键为(user_id,role_id,reply_message_id)，摘要覆盖合同版本、归属、精确content、task_type、reply ID和规范options，省略与空options等价。相同键不同内容返回IDEMPOTENCY_CONFLICT；X-Request-ID仅追踪单次HTTP。不同真实轮次允许相同文本，不按文本哈希永久去重；同轮客户端不得换reply ID重建请求。无匹配/失败请求的reply ID同样保留为原请求身份，普通回复使用新消息ID。

#### 12.1.1 摘要、检索和生成输入

POST_QUERY_SUMMARY_ENABLED由服务端配置，默认false，首次登记冻结。关闭时使用content检索；开启时可先提炼当前诉求、主题和必要条件，不新增事实。完整摘要输入默认8192 token、输出预留512 token，并受实际context window约束；输出query非空白、最多500个Unicode码点且须满足embedding预算。

每次HTTP成功取得准备租约后，摘要阶段默认最多30秒、最多1次模型发送，不做SDK内部重试；后续post检索默认15秒，各外部阶段最多2次总尝试。两段默认最多45秒（摘要关闭为15秒），全部受首次登记起300秒生成期限约束。retrieval_text尚未冻结的暂时失败可重试原POST；一旦确定摘要或原文降级并持久冻结，后续重试直接复用，不再次摘要或改变query。

摘要超时、请求失败或输出非法时，可用原文则降级并冻结；原文也不可直接检索则返回503 POST_QUERY_SUMMARY_FAILED，按剩余期限重试原请求，不伪装no_match。硬长度、已知固定输入token预算均不可用等确定参数错误返回400 INVALID_ARGUMENT；不把永久输入错误当暂时故障反复重试。已登记的确定失败在查清无未决决定后持久终止。embedding本地token检查继续按现有可配置策略执行，不强制增加本地BGE tokenizer就绪条件。

同一retrieval_text用于post的embedding/rerank及可选top1 Memory召回。只有成功完成检索筛选才形成no_match；检索故障不能当无匹配。已确认no_match、matched或终止失败不再检索，服务端不等待新post、不后台重新匹配。

最终回复Prompt直接包含原始完整content一次、冻结人设、选中的post概述、可选Memory和reply_language，不读取/追加服务端近期历史；检索摘要不替代原content进入生成Prompt。无记忆正常继续，记忆检索故障有限重试后降级。语言只从原始content检测，按角色supported_languages匹配zh/en/ja语言族；范围外、混合或不确定时使用完整default_language，不用摘要/post/Memory语言决定回复。角色资料、模型和预算首次登记冻结，重试不暗换。

#### 12.1.2 创建分支与示例

| 分支 | HTTP / 响应 | 客户端动作 |
|---|---|---|
| 成功检索无匹配 | 200，matched=false、reply_message_id；无task_id/status | 本轮改用普通回复，新消息ID；重放仍为原no_match，不等待新post。 |
| 命中并预留post | 首次202，matched=true及TaskState | 查询原任务；generated后上传候选，不再并行生成普通回复。 |
| 重放已命中请求 | 200，matched=true及原TaskState | 使用原task、原reply ID和当前结果，不重新生成。 |
| 决定形成中/查证中 | 409 MULTIMODAL_REQUEST_IN_PROGRESS | 按retry_after_ms恢复原POST。 |
| 摘要或post检索暂时故障 | 503及具体Error | 重试原POST，不能据此判断无匹配。 |
| 确认未建任务且生成期限耗尽 | 410 MULTIMODAL_REQUEST_EXPIRED | 服务端持久终止后可普通回复，新ID；不能重建本轮post任务。 |

创建请求，无需先同步content中的聊天：

```json
{
  "content": "用户：最近有点累。角色：忙完想去哪里放松？用户：想找个安静的地方，最好能看看海和晚霞。",
  "reply_message_id": "replymsg_post_1",
  "task_type": "post_link",
  "options": {}
}
```

首次命中HTTP 202：

```json
{
  "request_id": "req_post_create_1",
  "server_time_ms": 1795410001200,
  "matched": true,
  "task_id": "task_post_1",
  "status": "queued",
  "reply_message_id": "replymsg_post_1",
  "post_id": "post_123",
  "expires_at_ms": 1795410300000,
  "message": null
}
```

另一真实轮次无匹配HTTP 200：

```json
{
  "request_id": "req_post_no_match_1",
  "server_time_ms": 1795410001300,
  "matched": false,
  "reply_message_id": "replymsg_post_2"
}
```

响应仅有公共字段和本节业务字段，不增加动作枚举字段。“客户端动作”只存在于文档表格，客户端结合HTTP、稳定错误码和原任务阶段执行恢复。

### 12.2 查询任务、候选和正式结果

`GET /auto-media/api/chat/roles/{role_id}/multimodal-tasks/{task_id}`。无正文，只查当前认证用户和角色下的任务；成功HTTP 200。

| 字段 | 类型 | 出现条件 | 含义与规则 |
|---|---|---|---|
| task_id | ID | 必填 | 稳定生成任务ID。 |
| status | queued/running/generated/completed/cancelled/failed | 必填 | generated只表示候选已生成且允许等待上传；completed表示batchCreate已正式提交。 |
| reply_message_id | ID | 必填 | 稳定候选/正式回复消息ID。 |
| post_id | ID | 必填 | 此任务选定post，必须与候选及正式消息上下文一致。 |
| expires_at_ms | TimestampMs | 必填 | 从首次登记开始的生成期限，默认300秒；generated之后不再以此值限制上传，也不自动清理候选或释放post。 |
| generated_at_ms | TimestampMs | 已成功产出候选后出现，后续保留 | 服务端冻结生成时间，等于候选created_at_ms。未产候选时省略。 |
| message | MessageCreateInput / MessageContent / null | 所有TaskState必填 | generated时为七字段待上传候选；completed时为当前正式MessageContent，已删除或历史Epoch时为null；queued/running/failed/cancelled为null。按status解释，字段名始终相同。 |
| reply_sync_epoch | OpaqueString | 仅completed必填 | 首次正式写入messages的代次。 |
| object_seq / sync_seq | 正UInt64String或null | 仅completed必填 | 当前正式消息的创建号/版本；message为null则均null。不能据此直接推进cursor。 |
| sync_hints | SyncHint数组 | 仅completed必填，可为[] | 正式Scope变化提示，不证明本机已下载结果。 |
| error | Error | failed必填，其他省略 | 区分生成前失败与生成后内部目标Epoch/G变化；取消、post/Memory失效和占用变化不使已生成任务失败，见12.5。 |

MessageCreateInput恰好包括message_id、sender_type、sender_id、message_type、body、created_at_ms和proactive_context。message_id等于reply_message_id，role与post/task绑定，text非空且最多300个Unicode码点、attachments=[]；created_at_ms等于generated_at_ms，proactive_context恰好是2.2.1的五字段post分支。它不带origin、reply_to_message_id、server_created_at_ms或正式序号。

所有TaskState统一返回message，不另设reply字段。generated的message是可上传候选，尚无正式origin、server_created_at_ms或同步序号；completed的message是正式对象或null，不能将completed的完整对象再次作为候选上传。queued/running/failed/cancelled的message=null。reply_sync_epoch、object_seq、sync_seq和sync_hints仍仅completed返回，generated不伪造正式序号；只有内部目标Epoch/G变化等会话重置导致候选不可上传时才按12.5返回failed/null；后续取消、post/Memory失效或占用变化不清候选、不改变generated返回message的规则。任务查询、候选接收和本地展示不更新消息Scope、会话预览、未读、标读或后台记忆责任。完成后的正式结果补查也不直接推进cursor；仍通过普通messages同步建立权威视图。

generated示例：

```json
{
  "request_id": "req_post_generated_1",
  "server_time_ms": 1795410011000,
  "task_id": "task_post_1",
  "status": "generated",
  "reply_message_id": "replymsg_post_1",
  "post_id": "post_123",
  "expires_at_ms": 1795410300000,
  "generated_at_ms": 1795410010000,
  "message": {
    "message_id": "replymsg_post_1",
    "sender_type": "role",
    "sender_id": "role_1",
    "message_type": "text",
    "body": {
      "text": "想放松一下的话，可以看看我之前分享的海边散步。你更喜欢静静看海，还是沿着海岸走走？",
      "attachments": []
    },
    "created_at_ms": 1795410010000,
    "proactive_context": {
      "schema_version": 1,
      "trigger_type": "post_link",
      "link_type": "post",
      "post_id": "post_123",
      "task_id": "task_post_1"
    }
  }
}
```

客户端按5.6上传该message后，首次成功batchCreate回执仍采用第5节results。再查任务HTTP 200示例：

```json
{
  "request_id": "req_post_completed_1",
  "server_time_ms": 1795410031000,
  "task_id": "task_post_1",
  "status": "completed",
  "reply_message_id": "replymsg_post_1",
  "post_id": "post_123",
  "expires_at_ms": 1795410300000,
  "generated_at_ms": 1795410010000,
  "reply_sync_epoch": "msg-e1",
  "message": {
    "message_id": "replymsg_post_1",
    "sender_type": "role",
    "sender_id": "role_1",
    "origin": "server_multimodal",
    "reply_to_message_id": null,
    "proactive_context": {
      "schema_version": 1,
      "trigger_type": "post_link",
      "link_type": "post",
      "post_id": "post_123",
      "task_id": "task_post_1"
    },
    "message_type": "text",
    "body": {
      "text": "想放松一下的话，可以看看我之前分享的海边散步。你更喜欢静静看海，还是沿着海岸走走？",
      "attachments": []
    },
    "created_at_ms": 1795410010000,
    "server_created_at_ms": 1795410030000
  },
  "object_seq": "103",
  "sync_seq": "109",
  "sync_hints": [
    {
      "scope": "messages",
      "sync_epoch": "msg-e1",
      "server_max_sync_seq": "109",
      "min_valid_cursor": "0"
    }
  ]
}
```

completed且message=null只表示正式消息当前不可见，不回放旧候选、补发或生成第二条消息。若原batchCreate成功响应丢失，重放原消息仍回首次回执，不因消息已删除而重建。

### 12.3 期限、取消与post去重

生成从首次持久登记起默认300秒，包含可选摘要、post/Memory检索、排队、语言判断、回复生成和退避。回复模型最多3次，单次默认120秒，实际不超过剩余生成期限；摘要另计每次准备阶段最多1次/30秒，不占这3次。配置变更仅影响新请求。

合法生成结果须在期限内持久保存为generated；客户端收到后按业务已发布处理，后续batchCreate只是正式同步入库。模型仅返回、但服务端尚未确认持久保存的文本不构成可上传结果。生成事务结果未决先恢复原操作，不重新调用模型、换reply ID或伪造失败。未产候选的请求/任务确认到期后用REQUEST_EXPIRED/TASK_EXPIRED等生成错误终止，允许按12.5选择普通回复。

generated后不设置上传超时或上传截止时间。expires_at_ms只保留为原300秒生成期限，不能用于拒绝已生成候选的首次上传。客户端一直未上传时，任务维持generated，持续保留冻结message及message/task/post绑定，post维持reserved并被后续检索排除；不因等待时长转failed、清候选或释放占用。客户端后续仍可按5.6上传原候选；取消、post/Memory失效和占用异常都不阻断原消息同步，成功回执优先回放。内部冻结的目标messages Epoch/G变化仍按会话重置边界处理，不能将旧候选迁移到新会话。

`POST /auto-media/api/chat/roles/{role_id}/multimodal-tasks/{task_id}:cancel`无业务正文或空对象{}，HTTP 200返回当前TaskState。queued/running且确认尚未持久产生候选时可取消，原子阻止旧worker保存结果并释放本人占用。取消与生成落盘竞争时，以权威提交结果为准；未决生成先查证，不能擦除已经成立的候选。已generated的取消不撤销客户端已发布消息，仍返回generated及原message，不清正文、不释放占用、不阻止batchCreate；已有取消标记也不影响真实已生成结果的同步资格。

已completed取消不删除消息、不撤回post sent，返回既有完成结果；尚未产候选的failed/cancelled重复取消保持原结论。已generated后的post/Memory失效或占用变化与取消一样，不改变原文、原ID及同步责任，不使任务转failed/cancelled，也不触发第二条普通回复。具体本地展示由客户端决定。

post命中后为reserved，生成成功仍保持reserved；只有batchCreate原子成功才确认sent并永久去重。候选未产生前的取消/生成失败可释放占用；已generated后取消、post/Memory失效或未上传本身均不释放占用，异常占用也不能使已发布消息同步失败。内部目标Epoch/G重置按独立会话边界处理，不重置已sent事实。永久去重确认只是服务端同步状态，不否认客户端已经发布。删除正式回复、墓碑GC或Epoch切换不重置已sent事实。

### 12.4 客户端调用顺序与恢复

1. 本地整合最近聊天为content，持久保存稳定reply ID及原请求，调用创建；聊天可以尚未同步。
2. 无匹配时生成普通回复，使用新的普通消息ID；暂时检索/摘要错误只重试原POST；命中后查询同task，停止并行普通回复。
3. 收到generated，将message及task/reply ID持久化，按客户端业务已发布处理；它尚未取得正式同步序号，不能当作已同步记录。后续把message原样传入batchCreate，可以与原本未同步聊天一起提交。
4. 上传结果未知只重试原batchCreate，不能新建任务、换消息ID或再生成一次。先处理正式逐条回执，再按原有人设/亲密度会话规则处理快照，随后读取正式消息。
5. 只有未产候选、已确认无匹配或生成失败时，才按错误表考虑普通回复；已generated后遇到取消、post/Memory失效或占用变化继续原上传，绑定错误或暂时上传异常按原责任恢复，均不生成第二条回复。确认completed后只读取正式消息，message=null也不补发。

服务端不再增加客户端动作字段。错误表的“客户端动作”是文档说明，由HTTP、稳定code、已知任务阶段与幂等回执共同决定，不能解析error.message猜测流程。网络中断、无合法JSON、请求ID或业务ID不匹配的响应没有权威结论，保留原请求或原上传恢复。

content未提供精确消息版本，服务端无法据其建立全部聊天删除来源；不伪造来源完整性或reply_to关联。已注入Memory仍保存真实版本和已知根，明确失效时保留失效事实并按既有规则停止自动提取，原post消息仍正常上传保存。客户端以后可以在batchCreate附已有provenance声明，按5.1解析合并；这不是生成请求的前置条件，也不能仅因同批消息存在就猜测它们全部参与生成。post的content必含动态聊天，仅有Memory引用或complete+[]不证明来源完整；须有真实完整的聊天声明并成功解析才可按既有规则自动提取，否则partial/unavailable，可保存展示但不自动提取，不改写原客户端声明及上传摘要。

### 12.5 创建、生成与候选上传错误

| 场景 / code | HTTP或TaskState | 客户端动作 | 规则 |
|---|---|---|---|
| [POST_QUERY_SUMMARY_FAILED](#e01-post_query_summary_failed) / [POST_RETRIEVAL_FAILED](#e02-post_retrieval_failed) | 503，retryable=true | 重试原创建POST | 检索尚无可靠结论，不视为无匹配，不换reply ID或原content。 |
| [MULTIMODAL_REQUEST_IN_PROGRESS](#e03-multimodal_request_in_progress) | 409，retryable=true | 按retry_after_ms恢复原POST | 请求形成决定或查证中，不能伪造无匹配/生成失败。 |
| [MULTIMODAL_REQUEST_EXPIRED](#e04-multimodal_request_expired) | 410，retryable=false | 普通回复可接管，使用新消息ID | 已持久确认未建任务且生成期限到期，无未决决定；同请求保持原失败。 |
| [MULTIMODAL_TASK_EXPIRED](#e05-multimodal_task_expired) / [MULTIMODAL_GENERATION_FAILED](#e06-multimodal_generation_failed) | 200、failed，retryable=false | 未产候选时可普通回复，新ID | 生成期限或次数耗尽，已阻止迟到候选保存。不是上传失败。 |
| [MULTIMODAL_POST_INVALID](#e07-multimodal_post_invalid) | 200、failed，生成前 | 未产候选时可普通回复，新ID | 选定post在候选生成前明确失效，本轮不重新检索；已generated后的post变化不返回此码或RESULT_INVALID，原消息正常同步。 |
| [MULTIMODAL_TASK_NOT_FOUND](#e08-multimodal_task_not_found) | 查询或batchCreate 404，retryable=false | 核对身份并以原POST恢复 | 当前用户/角色下无任务，不用404推断原请求从未提交，不重建本轮；已有任务的绑定不符另用RESULT_MISMATCH。 |
| [MESSAGE_ID_RESERVED](#e34-message_id_reserved) | batchCreate 409，retryable=false | 恢复真实候选及绑定，不冒用预留ID | 将任务预留reply ID当普通消息上传、缺少合法post上下文等；不能换ID绕过该轮责任。 |
| [MULTIMODAL_RESULT_NOT_READY](#e09-multimodal_result_not_ready) | batchCreate 409 | 未产候选时不上传，按原task状态处理 | queued/running的retryable=true，查询到generated后上传原message；生成前failed/cancelled的retryable=false，不能伪造候选；本次放宽仅适用于真实已生成结果。 |
| [MULTIMODAL_RESULT_MISMATCH](#e10-multimodal_result_mismatch) | batchCreate 409，retryable=false | 停止候选上传，恢复原任务返回字段 | message/task/post/context/text/created_at或发送者不匹配；不修改内容并沿用原ID，不自动再回复。 |
| [MULTIMODAL_RESULT_INVALID](#e11-multimodal_result_invalid) | batchCreate 409或任务failed，retryable=false | 按会话重置处理旧责任，不自动重新回复 | 仅用于任务内部冻结的目标messages Epoch/G已变化；取消、post/Memory失效和占用不再有效不返回此码，允许原消息正常写入。 |
| batchCreate网络超时/响应丢失/[TEMPORARY_SERVER_ERROR](#e33-temporary_server_error) | 网络未知或503 | 重试原batchCreate负载 | 可能已经写入；消息ID、正文、时间及上下文保持冻结，不能重新生成。 |
| [IDEMPOTENCY_CONFLICT](#e22-idempotency_conflict) | 409，retryable=false | 停止冲突负载，恢复正确原请求/上传 | 同reply ID改生成内容，或同message ID改候选/metadata等冻结字段；不能用换ID掩盖本轮冲突。 |
| [INVALID_ARGUMENT](#e14-invalid_argument) / [OPERATION_NOT_SUPPORTED](#e31-operation_not_supported) | 400/422，retryable=false | 修正参数或部署，不凭此自动接管 | content硬限、确定token预算超限、旧新合同混用、非法post上下文/任务类型等；需核对是否已经有候选或正式回执。 |
| 生成前内部Epoch失效 / 确认取消 | cancelled或相应409 | 结束旧轮，不自动回复 | 不将旧任务迁移到新会话。生成后取消不撤销原候选，继续正常同步；生成提交未决先查证。 |
| 身份/权限/角色错误 | 相应401/403/404 | 修正身份或目标，保留原操作 | 不证明之前已登记任务或上传已经取消。 |
| [RATE_LIMITED](#e32-rate_limited) / [TEMPORARY_SERVER_ERROR](#e33-temporary_server_error) | 429/503 | 按出错阶段恢复原POST/GET/batchCreate | 遵守retry_after_ms，有候选后不因暂时异常重新生成。 |
| post向量Schema/配置诊断直接作为HTTP错误 | 具体HTTP错误 | 修复服务后恢复原责任 | 不因配置异常猜测已生成或已上传状态。 |
| [MULTIMODAL_INPUT_INVALID](#e12-multimodal_input_invalid) / [MULTIMODAL_INPUT_ALREADY_DECIDED](#e13-multimodal_input_already_decided)（仅message-v1） | 原v1的409 | 按旧请求/规范ID恢复 | 不用于v3 content的来源校验或文本全局去重，不转成新v3轮次。 |

任务failed的error.retryable=false仅表示该任务不会复活。生成前失败与生成后候选/上传失败必须按code和generated_at_ms区分：后者可能已在客户端使用，禁止一概映射成“客户端接管回复”。同批候选错误导致整批未提交时，保留各原消息，处理问题后重新提交合法Outbox项；已成功回执始终先回放。

候选生成后，内部冻结的目标messages Epoch/G因会话重置而变化的任务错误示例；取消、post/Memory失效、占用变化或等待时长不会触发此错误，客户端不因此自动再回复：

```json
{
  "request_id": "req_post_invalid_1",
  "server_time_ms": 1795496411000,
  "task_id": "task_post_1",
  "status": "failed",
  "reply_message_id": "replymsg_post_1",
  "post_id": "post_123",
  "expires_at_ms": 1795410300000,
  "generated_at_ms": 1795410010000,
  "error": {
    "code": "MULTIMODAL_RESULT_INVALID",
    "message": "候选所属的目标messages Epoch已重置",
    "retryable": false,
    "retry_after_ms": 0,
    "details": {
      "reason": "messages_epoch_changed"
    }
  },
  "message": null
}
```

### 12.6 契约版本与升级边界

client-commit-v3已接入API/worker、batchCreate、兼容validator、生成Prompt及联调工具；实际数据库迁移和部署仍由目标环境执行。内部input_contract_version=client-commit-v3区分新责任，客户端不增加版本字段，HTTP X-Request-ID不充当业务版本。

旧message-v1历史上由服务端直接正式发布。升级切换前停止旧新建并完成排空或取消可确认未提交的责任；未决发布仅按原事务身份查证，确认提交恢复completed，明确未提交则取消。新版本不再为旧任务发起新生成或直接发布，既有消息保留旧reply_to和正式回执。旧POST仅恢复已登记v1责任，不再登记新v1；不能将旧请求换成新content并复用同reply ID。旧v1 completed不可能成为v3 generated，不允许借新batchCreate分支重建旧已发布/已删消息。

此前aggregate-v2仅更新过文档，未实现，其来源列表和响应动作字段要求均由本次合同撤销，不构造不存在的v2运行迁移。新创建只接受v3形状，旧字段混用返回400；存量任务以实际冻结合同为准，不能仅凭缺版本标记猜成v3。

代码已接入message/task绑定、候选持续保留、batchCreate原子提交/回执优先、来源/时间规则及错误表；部署前须在隔离环境执行兼容Schema和索引迁移，再同步客户端与可视化工具。验收包括旧v1恢复、未同步content生成、generated到completed、长期未上传仍占用且可上传、上传未知/重放、生成前取消竞态、已generated后取消/post/Memory失效/占用异常仍可上传、目标Epoch/G重置、混批原子性和永久post去重。此文档更新不等于这些能力已经上线。

## 13 错误详情字段

error.code/message/retryable/retry_after_ms及可选details沿用1。下表只规定details已经出现时的子字段，不要求每个错误返回全部内容。

| details字段 | 类型/出现条件 | 含义与使用规则 |
|---|---|---|
| `role_id` | ID，可选 | 出错会话，不能扩大当前用户权限。 |
| `scope` | Scope，可选；413建议返回 | 出错数据类别。 |
| `requested_sync_epoch` | OpaqueString或null，可选 | 原请求Epoch；不能静默替换冻结业务写操作的来源Epoch。 |
| `current_sync_epoch` | OpaqueString，可选 | 当前服务端Epoch，供重新建基线。 |
| `cursor` | UInt64String，可选 | 被拒绝的原C或本轮接续H；结合模式解释，不直接推进本机位置。 |
| `min_valid_cursor` | UInt64String，可选 | 仍能保证墓碑接续的最低位置，低于它需重建。 |
| `server_max_sync_seq` | UInt64String，可选 | 服务端观察的当前已提交位置，仅提示。 |
| `object_id` | ID，可选；OBJECT_TOO_LARGE应返回 | 超大/冲突/被删除的目标对象。 |
| `base_sync_seq` | UInt64String，可选 | 业务Mutation依据的对象版本，不是创建号。 |
| `current_sync_seq` | UInt64String，可选 | 当前对象版本；不得据此改负载并复用原操作ID。 |
| `deleted` | boolean，可选 | 对象当前是否为墓碑。 |
| `canonical_reply_message_id` | ID，仅旧v1的MULTIMODAL_INPUT_ALREADY_DECIDED时必填 | 恢复原单消息规范业务回复ID；v3不按单消息或文本唯一去重，不是新回复许可。 |
| `task_id` | ID，多模态冲突且已有任务时返回 | 当前认证用户、角色下已经存在的任务ID；无匹配或未建任务失败不提供。 |
| `expires_at_ms` | TimestampMs，多模态错误时可返回 | 首次登记起的生成期限，不因重试刷新；generated之后不限制上传，任务不设上传截止时间。 |
| `reason` | string，目标会话重置导致候选不可上传时可返回 | 例如messages_epoch_changed；取消、post_changed、memory_source_invalid或占用变化不作为已生成消息的上传错误原因。不含原始聊天、模型凭据或内部Prompt。 |
| `mongo_code` / `code_name` | JSON整数 / string，Mongo拒绝写入时可选 | 原数值错误码及服务端固定名称映射；例如121 / DocumentValidationFailure。仅供定位，不替代外层code或原操作恢复规则。 |
| `logical_collection` / `stage` | string，已标记写入阶段时可选 | 本地白名单逻辑集合与操作阶段，例如reply_provenance_state / update_post_provenance；不包含物理连接信息、消息正文或原始Mongo异常。 |
| `operation_outcome` | string，提交结果未知时可选 | unknown表示原事务提交未能确认；继续用原message_id和原冻结负载恢复，不重新生成。未出现本字段也不证明消息一定未提交。 |
| `action` | string，可选 | 既有同步恢复提示，如retry_smaller_range、rebuild_recent、rebuild_full；只用于既有同步规则，不增加多模态响应动作枚举。 |
| `read_through_object_seq` | UInt64String，可选 | 非法阅读位置请求值。 |
| `current_object_seq` | UInt64String，可选 | 阅读位置校验时messages的创建高水位。 |
| `lower_exclusive` | UInt64String，413必填，其他可选 | 被拒绝区间的原下界。 |
| `upper_inclusive` | UInt64String，413必填，其他可选 | 被拒绝区间的原上界，不能借建议改变原覆盖任务。 |
| `suggested_partition_span` | JSON正整数，413必填 | 1～200且严格小于原区间宽度的建议值；建议不保证下一次一定不过字节限制。 |
| `suggested_ranges` | 对象数组，413可选 | 最多200项建议的替代叶范围；如出现，按顺序精确覆盖原失败范围，无空洞/重叠，不能含原范围自身。 |
| `suggested_ranges[].lower_exclusive` | UInt64String，每项必填 | 子范围排除下界；首项等于原下界，后项等于前项上界。 |
| `suggested_ranges[].upper_inclusive` | UInt64String，每项必填 | 子范围包含上界，须大于下界；最后一项等于原上界，每项宽度严格更小且<=建议跨度。 |

错误处理见下表。

## 14 按 ID 批量查询消息：messages:batchGet

`POST /auto-media/api/chat/roles/{role_id}/messages:batchGet`。用于按已知消息 ID 补查正文，例如展示回复引用。服务端从认证上下文获取用户身份，并校验会话访问权限。

| 请求字段 | 类型 | 必填条件 | 含义与校验规则 |
|---|---|---|---|
| `message_ids` | ID 数组 | 必填，1～100 项 | 要查询的消息 ID。每项必须是有效 ID，且同一 ID 不得重复。 |

| 响应字段 | 类型 | 出现条件 | 含义与使用规则 |
|---|---|---|---|
| `request_id` | ID | 必填 | 本次 HTTP 调用的追踪 ID。 |
| `server_time_ms` | TimestampMs | 必填 | 本次响应生成时间。 |
| `sync_epoch` | OpaqueString | 必填 | 本次读取的 messages 代次，适用于所有返回的序号。 |
| `messages` | SyncRecord 数组 | 必填 | 仅返回当前会话中有权读取的有效消息，按请求 ID 的相对顺序排列。每项包含 object_id、object_seq、sync_seq、operation="upsert" 和完整 MessageContent，结构见 2。object_id 等于 content.message_id。已删除、未找到或不属于本会话的 ID 跳过。无匹配项时返回空数组。 |

参数校验失败返回 HTTP 400，成功返回 HTTP 200。客户端可用 object_id 关联结果。同步游标和已读位置保持原值。

`POST /auto-media/api/chat/roles/role_1/messages:batchGet`，本次请求的 `X-Request-ID` 为 `req_batch_get_1`。请求正文如下。

```json
{
  "message_ids": ["msg_101", "msg_20"]
}
```

本例 `msg_101` 仍有效，`msg_20` 已删除。HTTP 200 响应如下。

```json
{
  "request_id": "req_batch_get_1",
  "server_time_ms": 1793670001000,
  "messages": [
    {
      "object_id": "msg_101",
      "object_seq": "101",
      "sync_seq": "101",
      "operation": "upsert",
      "content": {
        "message_id": "msg_101",
        "sender_type": "user",
        "sender_id": "user_1",
        "origin": "client_user",
        "reply_to_message_id": null,
        "proactive_context": null,
        "message_type": "text",
        "body": {
          "text": "帮我画一张日落",
          "attachments": []
        },
        "created_at_ms": 1793669999950,
        "server_created_at_ms": 1793669999990
      }
    }
  ],
  "sync_epoch": "messages-epoch-A"
}
```

## HTTP 状态与错误码

“客户端动作”列为文档说明，不是新增响应字段。多模态错误按创建、生成、候选上传的阶段解释；上传异常不等于需要重新生成。任务详细错误见12.5，普通同步继续执行原有恢复规则。[第15节](#15-错误码逐项示例)逐码提供请求、响应与客户端动作示例。

| HTTP / code | 客户端动作 | 说明 |
|---|---|---|
| 400 [INVALID_ARGUMENT](#e14-invalid_argument) / [INVALID_READ_POSITION](#e15-invalid_read_position) | 修正原调用 | 多模态包括content硬限、旧新合同混用和非法上下文；普通同步包括page_token、未来游标及非法区间。 |
| 401 [TOKEN_EXPIRED](#e16-token_expired) / [UNAUTHENTICATED](#e17-unauthenticated) | 修正身份后恢复原操作 | 保留冻结生成请求、候选Outbox与本地同步进度，不推断此前请求或上传未提交。 |
| 403 [FORBIDDEN](#e18-forbidden) / [UNTRUSTED_UPSTREAM](#e19-untrusted_upstream) | 停止当前调用，恢复权限/入口后查证 | 清除已撤销权限的会话快照，保留尚需恢复的原操作。 |
| 404 [ROLE_NOT_FOUND](#e20-role_not_found) / [NOT_FOUND](#e21-not_found) | 检查目标与归属 | 不将普通404当作生成或上传未提交的证明。 |
| 404 [MULTIMODAL_TASK_NOT_FOUND](#e08-multimodal_task_not_found) | 核对身份并通过原创建POST恢复 | 查询或batchCreate中当前user/role下无此task；保持原reply ID和content，不重建本轮。 |
| 409 [IDEMPOTENCY_CONFLICT](#e22-idempotency_conflict) | 停止冲突负载，恢复原值 | 同业务ID改变冻结参数；候选上下文、时间及正文也参与幂等。 |
| 409 [BASE_SYNC_SEQ_CONFLICT](#e23-base_sync_seq_conflict) / [OBJECT_DELETED](#e24-object_deleted) | 同步并由业务处理 | 保留草稿/原操作；已成功消息创建回执优先回放，不复活已删消息。 |
| 409 [SYNC_EPOCH_MISMATCH](#e25-sync_epoch_mismatch) | 停止旧同步责任，重新建立基线 | v3生成请求不传Epoch；候选首次上传另校验任务内部目标Epoch，失效用RESULT_INVALID。 |
| 409 [SYNC_CURSOR_EXPIRED](#e26-sync_cursor_expired) | 重开recent或full读取 | 不直接修改游标绕过墓碑边界。 |
| 409 [DELETE_REQUIRES_MAINTENANCE](#e27-delete_requires_maintenance) | 停止自动重试，交维护 | 删除整笔未执行，不能当部分成功。 |
| 410 [PAGE_TOKEN_EXPIRED](#e28-page_token_expired) | page_token置null重开首页 | 保留有效消息cursor/阅读位置。 |
| 413 [PARTITION_TOO_LARGE](#e29-partition_too_large) | 按建议拆小读取区间 | 不应用不存在的部分records。 |
| 422 [OBJECT_TOO_LARGE](#e30-object_too_large) | 修复单对象 | 停止无限拆分。 |
| 422 [OPERATION_NOT_SUPPORTED](#e31-operation_not_supported) | 检查任务类型、部署或Scope | 不表示任务已接受，也不说明之前候选/上传已取消。 |
| 409 [MULTIMODAL_REQUEST_IN_PROGRESS](#e03-multimodal_request_in_progress) | 重试原创建POST | 遵守retry_after_ms，不换reply ID，不推断no_match。 |
| 410 [MULTIMODAL_REQUEST_EXPIRED](#e04-multimodal_request_expired) | 未产候选时普通回复可接管，新ID | 已持久确认无任务、无未决决定且生成期限到期。 |
| 503 [POST_QUERY_SUMMARY_FAILED](#e01-post_query_summary_failed) / [POST_RETRIEVAL_FAILED](#e02-post_retrieval_failed) | 重试原创建POST | 原检索尚无可靠决定，不作为无匹配。 |
| 200 + TaskState queued/running | 查询同task | 不并行生成本轮普通回复。 |
| 200 + TaskState generated | 冻结message并调用batchCreate | 候选未写messages，不推进同步cursor。 |
| 200 + failed，[MULTIMODAL_TASK_EXPIRED](#e05-multimodal_task_expired) / [MULTIMODAL_GENERATION_FAILED](#e06-multimodal_generation_failed) / 生成前[MULTIMODAL_POST_INVALID](#e07-multimodal_post_invalid) | 未产候选时普通回复可接管，新ID | 先按稳定错误码确认生成已终止，不复活本轮。 |
| 409 [MESSAGE_ID_RESERVED](#e34-message_id_reserved) | 恢复原候选/上下文，保留原责任 | 预留reply ID不能当普通消息先行写入；与候选尚未就绪和正文不符分别处理。 |
| 409 [MULTIMODAL_RESULT_NOT_READY](#e09-multimodal_result_not_ready) | queued/running继续查询；生成前failed/cancelled停止候选上传 | 未产候选，queued/running可重试，生成前failed/cancelled不可重试；不能因放宽取消规则伪造从未生成的结果。 |
| 409 [MULTIMODAL_RESULT_MISMATCH](#e10-multimodal_result_mismatch) | 停止上传，恢复原候选字段 | 检查message/task/post/text/time/context，不自动再回复。 |
| 409 [MULTIMODAL_RESULT_INVALID](#e11-multimodal_result_invalid) | 按目标会话重置处理，不自动再回复 | 仅内部目标messages Epoch/G变化；取消、post/Memory失效和占用变化仍允许原消息正常写入。 |
| 200 + failed，[MULTIMODAL_RESULT_INVALID](#e11-multimodal_result_invalid) | 按目标会话重置处理，不自动接管回复 | 目标messages Epoch/G已变化，保留generated_at_ms以区别生成前失败；不用于取消、post/Memory或占用变化。 |
| 200 + TaskState cancelled | 结束尚未产候选的旧轮 | 已generated的取消仍返回generated及原message并允许上传，不会因取消变为此状态。 |
| 200 + TaskState completed | 接收正式回执/消息并同步 | message=null也不补发，不把旧候选重新上传成新消息。 |
| batchCreate网络未知或503 | 重试原batchCreate | 可能已经正式提交；保持消息ID、上下文、正文和时间，不重新生成。 |
| 429 [RATE_LIMITED](#e32-rate_limited) / 503 [TEMPORARY_SERVER_ERROR](#e33-temporary_server_error) | 按出错阶段重试原POST/GET/batchCreate | 遵守retry_after_ms；候选已返回后不因暂时异常生成第二条回复。 |
| 配置 / [POST_VECTOR_SCHEMA_INVALID](#e35-post_vector_schema_invalid)等直接HTTP诊断 | 修复服务后恢复原责任 | 配置异常本身不证明候选或上传结果。 |
| [MULTIMODAL_INPUT_INVALID](#e12-multimodal_input_invalid) / [MULTIMODAL_INPUT_ALREADY_DECIDED](#e13-multimodal_input_already_decided)（旧message-v1） | 按原单消息合同恢复 | 只保留旧责任恢复，不要求v3先同步聊天或传来源列表。 |

## 15 错误码逐项示例

本节为前述客户端公开错误表中的每个code提供独立示例；一个表格行中的多个code分别展示，不使用斜杠拼接error.code。每例给出触发条件、接口、请求、HTTP状态、完整错误响应和客户端动作。details只展示该例需要的可选字段，message文字仅供说明，客户端不能按文字判断流程。

涉及client-commit-v3的新分支均为当前代码合同示例，不代表目标部署已升级或真实依赖已验收；E12/E13仅解释旧message-v1历史责任。登录Token、网关权限和受信入口的兼容拒绝见相应示例，不因此新增业务服务的鉴权实现。

除身份/入口错误例外，请求均使用第1.2节合法公共头：Authorization为当前用户的Bearer令牌、X-Device-ID为device_1，X-Request-ID取各例标注值，JSON请求的Content-Type为application/json。认证上下文为user_1；客户端不提交user_id或自行构造受信转发身份。

各例独立，不表示依次发生在同一真实任务上。E01–E11及E33的post候选例使用role_1、replymsg_post_1、task_post_1和post_123；原始请求登记时间1795410000000，生成期限1795410300000（300秒），产生候选时generated_at_ms=1795410010000。生成期限只约束产出候选，上传没有截止时间；原候选已存在时不得把HTTP错误直接理解为需要第二次生成。

| 错误码 | 独立示例 |
|---|---|
| `POST_QUERY_SUMMARY_FAILED` | [E01](#e01-post_query_summary_failed) |
| `POST_RETRIEVAL_FAILED` | [E02](#e02-post_retrieval_failed) |
| `MULTIMODAL_REQUEST_IN_PROGRESS` | [E03](#e03-multimodal_request_in_progress) |
| `MULTIMODAL_REQUEST_EXPIRED` | [E04](#e04-multimodal_request_expired) |
| `MULTIMODAL_TASK_EXPIRED` | [E05](#e05-multimodal_task_expired) |
| `MULTIMODAL_GENERATION_FAILED` | [E06](#e06-multimodal_generation_failed) |
| `MULTIMODAL_POST_INVALID` | [E07](#e07-multimodal_post_invalid) |
| `MULTIMODAL_TASK_NOT_FOUND` | [E08](#e08-multimodal_task_not_found) |
| `MULTIMODAL_RESULT_NOT_READY` | [E09](#e09-multimodal_result_not_ready) |
| `MULTIMODAL_RESULT_MISMATCH` | [E10](#e10-multimodal_result_mismatch) |
| `MULTIMODAL_RESULT_INVALID` | [E11](#e11-multimodal_result_invalid) |
| `MULTIMODAL_INPUT_INVALID` | [E12](#e12-multimodal_input_invalid) |
| `MULTIMODAL_INPUT_ALREADY_DECIDED` | [E13](#e13-multimodal_input_already_decided) |
| `INVALID_ARGUMENT` | [E14](#e14-invalid_argument) |
| `INVALID_READ_POSITION` | [E15](#e15-invalid_read_position) |
| `TOKEN_EXPIRED` | [E16](#e16-token_expired) |
| `UNAUTHENTICATED` | [E17](#e17-unauthenticated) |
| `FORBIDDEN` | [E18](#e18-forbidden) |
| `UNTRUSTED_UPSTREAM` | [E19](#e19-untrusted_upstream) |
| `ROLE_NOT_FOUND` | [E20](#e20-role_not_found) |
| `NOT_FOUND` | [E21](#e21-not_found) |
| `IDEMPOTENCY_CONFLICT` | [E22](#e22-idempotency_conflict) |
| `BASE_SYNC_SEQ_CONFLICT` | [E23](#e23-base_sync_seq_conflict) |
| `OBJECT_DELETED` | [E24](#e24-object_deleted) |
| `SYNC_EPOCH_MISMATCH` | [E25](#e25-sync_epoch_mismatch) |
| `SYNC_CURSOR_EXPIRED` | [E26](#e26-sync_cursor_expired) |
| `DELETE_REQUIRES_MAINTENANCE` | [E27](#e27-delete_requires_maintenance) |
| `PAGE_TOKEN_EXPIRED` | [E28](#e28-page_token_expired) |
| `PARTITION_TOO_LARGE` | [E29](#e29-partition_too_large) |
| `OBJECT_TOO_LARGE` | [E30](#e30-object_too_large) |
| `OPERATION_NOT_SUPPORTED` | [E31](#e31-operation_not_supported) |
| `RATE_LIMITED` | [E32](#e32-rate_limited) |
| `TEMPORARY_SERVER_ERROR` | [E33](#e33-temporary_server_error) |
| `MESSAGE_ID_RESERVED` | [E34](#e34-message_id_reserved) |
| `POST_VECTOR_SCHEMA_INVALID` | [E35](#e35-post_vector_schema_invalid) |

### E01 POST_QUERY_SUMMARY_FAILED

本例配置了较小的直接embedding输入预算，下面原文超出该预算但仍符合摘要模型输入预算。开启摘要后模型超时，无法安全降级，且原请求仍在300秒生成期限内。若原文可用，应直接降级检索，不返回此错误。

`POST /auto-media/api/chat/roles/role_1/multimodal-tasks`，`X-Request-ID: req_error_01`。请求正文：

```json
{
  "content": "用户：最近有点累。角色：忙完想去哪里放松？用户：想找个安静的地方，最好能看看海和晚霞。",
  "reply_message_id": "replymsg_post_1",
  "task_type": "post_link",
  "options": {}
}
```

HTTP 503：

```json
{
  "request_id": "req_error_01",
  "server_time_ms": 1795410020000,
  "error": {
    "code": "POST_QUERY_SUMMARY_FAILED",
    "message": "检索摘要暂时不可用，原文无法直接检索",
    "retryable": true,
    "retry_after_ms": 1000
  }
}
```

客户端动作：等待至少1000毫秒，使用完全相同的content、reply_message_id和其他业务参数重试原创建POST；本轮尚未得出无匹配结论。

### E02 POST_RETRIEVAL_FAILED

Milvus检索或post重排服务在本次有限重试后仍不可用，尚未形成持久检索决定。

`POST /auto-media/api/chat/roles/role_1/multimodal-tasks`，`X-Request-ID: req_error_02`。请求正文：

```json
{
  "content": "用户：最近有点累。角色：忙完想去哪里放松？用户：想找个安静的地方，最好能看看海和晚霞。",
  "reply_message_id": "replymsg_post_1",
  "task_type": "post_link",
  "options": {}
}
```

HTTP 503：

```json
{
  "request_id": "req_error_02",
  "server_time_ms": 1795410020000,
  "error": {
    "code": "POST_RETRIEVAL_FAILED",
    "message": "post检索暂时不可用",
    "retryable": true,
    "retry_after_ms": 1000
  }
}
```

客户端动作：按建议等待后重试原创建POST；不换reply ID，不把服务故障当作no_match，也不并行生成普通回复。

### E03 MULTIMODAL_REQUEST_IN_PROGRESS

同一reply ID的原请求仍由有效准备租约处理，或其检索决定的提交结果还在查证；重复POST不能抢占后重新检索。

`POST /auto-media/api/chat/roles/role_1/multimodal-tasks`，`X-Request-ID: req_error_03`。请求正文：

```json
{
  "content": "用户：最近有点累。角色：忙完想去哪里放松？用户：想找个安静的地方，最好能看看海和晚霞。",
  "reply_message_id": "replymsg_post_1",
  "task_type": "post_link",
  "options": {}
}
```

HTTP 409：

```json
{
  "request_id": "req_error_03",
  "server_time_ms": 1795410020000,
  "error": {
    "code": "MULTIMODAL_REQUEST_IN_PROGRESS",
    "message": "原请求仍在处理或查证中",
    "retryable": true,
    "retry_after_ms": 1000,
    "details": {
      "expires_at_ms": 1795410300000
    }
  }
}
```

客户端动作：等待至少1000毫秒后重试原POST；保持原业务参数，不推断失败、取消或无匹配。

### E04 MULTIMODAL_REQUEST_EXPIRED

首次登记后300秒已过；服务端已持久确认本轮没有创建任务、没有未决检索决定，并终止原请求。

`POST /auto-media/api/chat/roles/role_1/multimodal-tasks`，`X-Request-ID: req_error_04`。请求正文：

```json
{
  "content": "用户：最近有点累。角色：忙完想去哪里放松？用户：想找个安静的地方，最好能看看海和晚霞。",
  "reply_message_id": "replymsg_post_1",
  "task_type": "post_link",
  "options": {}
}
```

HTTP 410：

```json
{
  "request_id": "req_error_04",
  "server_time_ms": 1795410301000,
  "error": {
    "code": "MULTIMODAL_REQUEST_EXPIRED",
    "message": "原请求的生成期限已结束且未创建任务",
    "retryable": false,
    "retry_after_ms": 0,
    "details": {
      "expires_at_ms": 1795410300000
    }
  }
}
```

客户端动作：本轮可由客户端生成普通回复，使用新的普通消息ID；原post请求不复活，也不换reply ID重建同一轮post任务。

### E05 MULTIMODAL_TASK_EXPIRED

已命中post并创建任务，但排队、准备或模型调用耗尽生成期限，未接受任何候选；迟到模型结果已被阻止。

`GET /auto-media/api/chat/roles/role_1/multimodal-tasks/task_post_1`，`X-Request-ID: req_error_05`。请求无正文。

HTTP 200；查询成功，任务业务失败：

```json
{
  "request_id": "req_error_05",
  "server_time_ms": 1795410301000,
  "task_id": "task_post_1",
  "status": "failed",
  "reply_message_id": "replymsg_post_1",
  "post_id": "post_123",
  "expires_at_ms": 1795410300000,
  "message": null,
  "error": {
    "code": "MULTIMODAL_TASK_EXPIRED",
    "message": "生成期限已结束，未产生候选",
    "retryable": false,
    "retry_after_ms": 0
  }
}
```

客户端动作：确认这是未产候选的终止状态后，可用新的普通消息ID接管回复；不继续重试该任务的生成。

### E06 MULTIMODAL_GENERATION_FAILED

回复模型在最多3次发送预算内持续返回不合法输出，尚未生成候选；该任务已终止，不再尝试。

`GET /auto-media/api/chat/roles/role_1/multimodal-tasks/task_post_1`，`X-Request-ID: req_error_06`。请求无正文。

HTTP 200；查询成功，任务业务失败：

```json
{
  "request_id": "req_error_06",
  "server_time_ms": 1795410020000,
  "task_id": "task_post_1",
  "status": "failed",
  "reply_message_id": "replymsg_post_1",
  "post_id": "post_123",
  "expires_at_ms": 1795410300000,
  "message": null,
  "error": {
    "code": "MULTIMODAL_GENERATION_FAILED",
    "message": "回复生成尝试已耗尽，未产生候选",
    "retryable": false,
    "retry_after_ms": 0
  }
}
```

客户端动作：本轮可由客户端以新的普通消息ID生成普通回复；不通过改content或reply ID让原post任务重跑。

### E07 MULTIMODAL_POST_INVALID

任务已选定post，但在生成候选前发现Mongo中该post的posted_status已从success改变；本轮不重新检索其他post。

`GET /auto-media/api/chat/roles/role_1/multimodal-tasks/task_post_1`，`X-Request-ID: req_error_07`。请求无正文。

HTTP 200；查询成功，任务业务失败：

```json
{
  "request_id": "req_error_07",
  "server_time_ms": 1795410020000,
  "task_id": "task_post_1",
  "status": "failed",
  "reply_message_id": "replymsg_post_1",
  "post_id": "post_123",
  "expires_at_ms": 1795410300000,
  "message": null,
  "error": {
    "code": "MULTIMODAL_POST_INVALID",
    "message": "选定post在候选生成前已失效",
    "retryable": false,
    "retry_after_ms": 0
  }
}
```

客户端动作：未产候选时可用新的普通消息ID接管；不能将此码用于已生成候选的上传失效。

### E08 MULTIMODAL_TASK_NOT_FOUND

当前认证用户与role_1下查不到请求的task ID；该结果不能证明先前创建请求一定没有提交。

`GET /auto-media/api/chat/roles/role_1/multimodal-tasks/task_missing`，`X-Request-ID: req_error_08`。请求无正文。

HTTP 404：

```json
{
  "request_id": "req_error_08",
  "server_time_ms": 1795410020000,
  "error": {
    "code": "MULTIMODAL_TASK_NOT_FOUND",
    "message": "当前用户和角色下未找到该任务",
    "retryable": false,
    "retry_after_ms": 0
  }
}
```

客户端动作：核对账号、角色和task ID，通过原content与原reply ID重放创建POST恢复责任；不另起一轮。

### E09 MULTIMODAL_RESULT_NOT_READY

任务仍queued/running，客户端却提前拼接正文和上下文并尝试写入预留reply ID。下面是应被拒绝的提前上传，不能作为正常接入方式。

`POST /auto-media/api/chat/roles/role_1/messages:batchCreate`，`X-Request-ID: req_error_09`。请求正文：

```json
{
  "messages": [
    {
      "message_id": "replymsg_post_1",
      "sender_type": "role",
      "sender_id": "role_1",
      "message_type": "text",
      "body": {
        "text": "想放松一下的话，可以看看我之前分享的海边散步。你更喜欢静静看海，还是沿着海岸走走？",
        "attachments": []
      },
      "created_at_ms": 1795410010000,
      "proactive_context": {
        "schema_version": 1,
        "trigger_type": "post_link",
        "link_type": "post",
        "post_id": "post_123",
        "task_id": "task_post_1"
      }
    }
  ]
}
```

HTTP 409：

```json
{
  "request_id": "req_error_09",
  "server_time_ms": 1795410020000,
  "error": {
    "code": "MULTIMODAL_RESULT_NOT_READY",
    "message": "任务尚未生成可上传候选",
    "retryable": true,
    "retry_after_ms": 1000,
    "details": {
      "task_id": "task_post_1"
    }
  }
}
```

客户端动作：停止提前上传，查询同task；仅在收到generated及服务端message后，冻结并上传实际候选，不能沿用猜测正文。

### E10 MULTIMODAL_RESULT_MISMATCH

任务已generated且尚无成功上传回执，客户端将候选原文改写后仍沿用原message/task/post ID。即使语义接近也不接受。

`POST /auto-media/api/chat/roles/role_1/messages:batchCreate`，`X-Request-ID: req_error_10`。请求正文：

```json
{
  "messages": [
    {
      "message_id": "replymsg_post_1",
      "sender_type": "role",
      "sender_id": "role_1",
      "message_type": "text",
      "body": {
        "text": "我替服务端改写了一下这条回复。",
        "attachments": []
      },
      "created_at_ms": 1795410010000,
      "proactive_context": {
        "schema_version": 1,
        "trigger_type": "post_link",
        "link_type": "post",
        "post_id": "post_123",
        "task_id": "task_post_1"
      }
    }
  ]
}
```

HTTP 409：

```json
{
  "request_id": "req_error_10",
  "server_time_ms": 1795410020000,
  "error": {
    "code": "MULTIMODAL_RESULT_MISMATCH",
    "message": "上传文本与任务冻结候选不一致",
    "retryable": false,
    "retry_after_ms": 0,
    "details": {
      "task_id": "task_post_1"
    }
  }
}
```

客户端动作：停止冲突上传，从原任务恢复未经修改的七字段message，保留该轮原始身份；不换消息ID上传改写文本，不自动再生成回复。若已经有正式成功回执，优先按幂等回执规则处理。

### E11 MULTIMODAL_RESULT_INVALID

任务已在generated返回过候选，内部冻结的目标messages Epoch为messages-epoch-A；首次上传前会话重置为messages-epoch-B，旧候选不得迁移到新代次。没有已经成功的原上传回执。任务取消、post/Memory失效、占用变化或等待多久本身均不会触发此错误，原目标代次仍活动时这些情况允许正常写入。

`POST /auto-media/api/chat/roles/role_1/messages:batchCreate`，`X-Request-ID: req_error_11`。请求正文：

```json
{
  "messages": [
    {
      "message_id": "replymsg_post_1",
      "sender_type": "role",
      "sender_id": "role_1",
      "message_type": "text",
      "body": {
        "text": "想放松一下的话，可以看看我之前分享的海边散步。你更喜欢静静看海，还是沿着海岸走走？",
        "attachments": []
      },
      "created_at_ms": 1795410010000,
      "proactive_context": {
        "schema_version": 1,
        "trigger_type": "post_link",
        "link_type": "post",
        "post_id": "post_123",
        "task_id": "task_post_1"
      }
    }
  ]
}
```

HTTP 409：

```json
{
  "request_id": "req_error_11",
  "server_time_ms": 1795410020000,
  "error": {
    "code": "MULTIMODAL_RESULT_INVALID",
    "message": "候选所属的目标messages Epoch已重置",
    "retryable": false,
    "retry_after_ms": 0,
    "details": {
      "task_id": "task_post_1",
      "reason": "messages_epoch_changed"
    }
  }
}
```

客户端动作：按messages会话重置处理旧责任，不把旧候选或冻结任务切换到新Epoch/G，不自动补一条普通回复。若上传结果仍未知，先恢复原batchCreate责任，已有成功回执仍优先回放，不能凭此跳过查证。

同一目标Epoch/G变化若已确认并持久终止，再调用 `GET /auto-media/api/chat/roles/role_1/multimodal-tasks/task_post_1`（`X-Request-ID: req_error_11_query`，无正文），HTTP 200查询结果如下。保留generated_at_ms以明确本轮曾生成候选：

```json
{
  "request_id": "req_error_11_query",
  "server_time_ms": 1795410021000,
  "task_id": "task_post_1",
  "status": "failed",
  "reply_message_id": "replymsg_post_1",
  "post_id": "post_123",
  "expires_at_ms": 1795410300000,
  "generated_at_ms": 1795410010000,
  "message": null,
  "error": {
    "code": "MULTIMODAL_RESULT_INVALID",
    "message": "候选所属的目标messages Epoch已重置",
    "retryable": false,
    "retry_after_ms": 0,
    "details": {
      "reason": "messages_epoch_changed"
    }
  }
}
```

客户端动作同上；这是查询成功返回的业务失败，不是生成前失败。

### E12 MULTIMODAL_INPUT_INVALID

仅旧message-v1责任恢复示例：已登记旧请求引用的输入消息msg_input_1已删除，或不再满足冻结版本/有效用户文本条件。v3新创建不接受这组字段，混用时返回INVALID_ARGUMENT；不能通过此示例新建旧任务。

`POST /auto-media/api/chat/roles/role_1/multimodal-tasks`，`X-Request-ID: req_error_12`。请求正文：

```json
{
  "messages_sync_epoch": "messages-epoch-A",
  "input_message_id": "msg_input_1",
  "input_sync_seq": "42",
  "reply_message_id": "replymsg_legacy_1",
  "task_type": "post_link",
  "options": {}
}
```

HTTP 409：

```json
{
  "request_id": "req_error_12",
  "server_time_ms": 1795410020000,
  "error": {
    "code": "MULTIMODAL_INPUT_INVALID",
    "message": "旧请求的输入消息已失效",
    "retryable": false,
    "retry_after_ms": 0
  }
}
```

客户端动作：按旧责任与删除规则结束或查证该轮，保留原业务ID；不把旧source字段加入v3请求，不换成content并复用旧reply ID。

### E13 MULTIMODAL_INPUT_ALREADY_DECIDED

仅旧message-v1历史错误回放示例：同一冻结输入版本已经登记规范reply ID，但另一个reply ID重复请求。它解释存量错误及旧客户端恢复行为；新部署不登记新v1责任，v3也不按相同文本或单条输入全局去重。

`POST /auto-media/api/chat/roles/role_1/multimodal-tasks`，`X-Request-ID: req_error_13`。请求正文：

```json
{
  "messages_sync_epoch": "messages-epoch-A",
  "input_message_id": "msg_input_1",
  "input_sync_seq": "42",
  "reply_message_id": "replymsg_legacy_duplicate",
  "task_type": "post_link",
  "options": {}
}
```

HTTP 409：

```json
{
  "request_id": "req_error_13",
  "server_time_ms": 1795410020000,
  "error": {
    "code": "MULTIMODAL_INPUT_ALREADY_DECIDED",
    "message": "旧输入已有规范回复责任",
    "retryable": false,
    "retry_after_ms": 0,
    "details": {
      "canonical_reply_message_id": "replymsg_legacy_1",
      "task_id": "task_legacy_1"
    }
  }
}
```

客户端动作：丢弃本轮重复身份，按canonical_reply_message_id恢复原旧请求；有task_id则查询原任务。不能另开v3同轮任务或再次上传旧已发布消息。

### E14 INVALID_ARGUMENT

会话列表的 page_size 必须为 1–100；本例传入 0。

请求：

```http
POST /auto-media/api/chat/threads/query
Authorization: Bearer <access_token>
X-Device-ID: device_1
X-Request-ID: req_e14
Content-Type: application/json

{
  "page_size": 0,
  "page_token": null
}
```

响应：`HTTP 400`。

```json
{
  "request_id": "req_e14",
  "server_time_ms": 1795410000000,
  "error": {
    "code": "INVALID_ARGUMENT",
    "message": "page_size 必须在 1–100 范围内",
    "retryable": false,
    "retry_after_ms": 0
  }
}
```

客户端动作：修正 page_size 后重新请求；参数校验失败不用于判断其他未决任务或上传是否成功。

### E15 INVALID_READ_POSITION

服务端当前消息最大 sync_seq 为 500，客户端却使用未来游标 501。

请求：

```http
POST /auto-media/api/chat/roles/role_1/scopes/messages/read-batches
Authorization: Bearer <access_token>
X-Device-ID: device_1
X-Request-ID: req_e15
Content-Type: application/json

{
  "mode": "incremental",
  "sync_epoch": "msg-e1",
  "cursor": "501"
}
```

响应：`HTTP 400`。

```json
{
  "request_id": "req_e15",
  "server_time_ms": 1795410000000,
  "error": {
    "code": "INVALID_READ_POSITION",
    "message": "读取位置超过服务端当前位置",
    "retryable": false,
    "retry_after_ms": 0,
    "details": {
      "scope": "messages",
      "server_max_sync_seq": "500"
    }
  }
}
```

客户端动作：检查本地 Scope、Epoch 和游标来源；恢复正确的已提交位置，不把服务端提示位置直接当作本地已完成游标。

### E16 TOKEN_EXPIRED

既有网关鉴权透传的可选兼容示例：登录令牌已过期。该错误由网关签发，不表示聊天服务自身解析或刷新 Token。

请求：

```http
GET /auto-media/api/chat/role-profiles/role_1
Authorization: Bearer expired_access_token
X-Device-ID: device_1
X-Request-ID: req_e16
```

响应：`HTTP 401`。

```json
{
  "request_id": "req_e16",
  "server_time_ms": 1795410000000,
  "error": {
    "code": "TOKEN_EXPIRED",
    "message": "访问令牌已过期",
    "retryable": false,
    "retry_after_ms": 0
  }
}
```

客户端动作：刷新登录令牌后恢复原调用；保留原业务 ID、冻结请求和待上传候选，不重新生成回复。

### E17 UNAUTHENTICATED

客户端经公共入口发起请求，但身份链路未能提供有效身份；例如网关未正确转发已认证身份。聊天服务会拒绝缺失或非法的身份信息。

请求：

```http
GET /auto-media/api/chat/role-profiles/role_1
Authorization: Bearer <access_token>
X-Device-ID: device_1
X-Request-ID: req_e17
```

响应：`HTTP 401`。

```json
{
  "request_id": "req_e17",
  "server_time_ms": 1795410000000,
  "error": {
    "code": "UNAUTHENTICATED",
    "message": "无法确认请求身份",
    "retryable": false,
    "retry_after_ms": 0
  }
}
```

客户端动作：恢复登录会话；若登录正常仍失败，检查网关身份转发。恢复后重试原操作，不据此推断此前生成或上传未提交。

### E18 FORBIDDEN

既有网关鉴权透传的可选兼容示例：令牌有效，但网关按已有角色访问权限策略拒绝访问 role_1。

请求：

```http
GET /auto-media/api/chat/role-profiles/role_1
Authorization: Bearer <access_token>
X-Device-ID: device_1
X-Request-ID: req_e18
```

响应：`HTTP 403`。

```json
{
  "request_id": "req_e18",
  "server_time_ms": 1795410000000,
  "error": {
    "code": "FORBIDDEN",
    "message": "无权访问该角色",
    "retryable": false,
    "retry_after_ms": 0
  }
}
```

客户端动作：停止当前调用并恢复权限；清理已撤销权限的会话快照，保留需要查证的原操作，不自动接管生成回复。

### E19 UNTRUSTED_UPSTREAM

受信入口部署的兼容拒绝示例：请求没有经过部署要求的受信网关链路，被入口层拒绝。是否直出本码取决于入口配置，当前业务服务不负责实现网关网络准入；客户端不得自行构造内部身份转发头。

请求：

```http
GET /auto-media/api/chat/role-profiles/role_1
Authorization: Bearer <access_token>
X-Device-ID: device_1
X-Request-ID: req_e19
```

响应：`HTTP 403`。

```json
{
  "request_id": "req_e19",
  "server_time_ms": 1795410000000,
  "error": {
    "code": "UNTRUSTED_UPSTREAM",
    "message": "请求入口不受信任",
    "retryable": false,
    "retry_after_ms": 0
  }
}
```

客户端动作：恢复正确网关地址与入口配置后，查证并恢复原操作；不要用客户端增加内部转发头的方式重试。

### E20 ROLE_NOT_FOUND

role_missing 在角色资料中不存在；本例身份和请求头均有效。

请求：

```http
GET /auto-media/api/chat/role-profiles/role_missing
Authorization: Bearer <access_token>
X-Device-ID: device_1
X-Request-ID: req_e20
```

响应：`HTTP 404`。

```json
{
  "request_id": "req_e20",
  "server_time_ms": 1795410000000,
  "error": {
    "code": "ROLE_NOT_FOUND",
    "message": "角色不存在",
    "retryable": false,
    "retry_after_ms": 0
  }
}
```

客户端动作：检查角色 ID 和可访问角色列表，停止对不存在角色的自动调用；该错误不证明其他目标上的旧任务或上传已经取消。

### E21 NOT_FOUND

角色存在，但当前认证用户与该角色范围内不存在 mutation_id=mut_missing 的删除回执。

请求：

```http
GET /auto-media/api/chat/roles/role_1/scopes/messages/mutations/mut_missing
Authorization: Bearer <access_token>
X-Device-ID: device_1
X-Request-ID: req_e21
```

响应：`HTTP 404`。

```json
{
  "request_id": "req_e21",
  "server_time_ms": 1795410000000,
  "error": {
    "code": "NOT_FOUND",
    "message": "变更回执不存在",
    "retryable": false,
    "retry_after_ms": 0
  }
}
```

客户端动作：核对用户、角色和 mutation_id；恢复删除操作时使用原请求及原 mutation_id，不因查询 404 就换 ID 发起不同删除。

### E22 IDEMPOTENCY_CONFLICT

m_user_1 已用正文“原始正文”成功创建；本例复用同一 message_id 却改成“修改后的正文”。

请求：

```http
POST /auto-media/api/chat/roles/role_1/messages:batchCreate
Authorization: Bearer <access_token>
X-Device-ID: device_1
X-Request-ID: req_e22
Content-Type: application/json

{
  "messages": [
    {
      "message_id": "m_user_1",
      "sender_type": "user",
      "sender_id": "user_1",
      "message_type": "text",
      "body": {
        "text": "修改后的正文",
        "attachments": []
      },
      "created_at_ms": 1795410000000
    }
  ]
}
```

响应：`HTTP 409`。

```json
{
  "request_id": "req_e22",
  "server_time_ms": 1795410000000,
  "error": {
    "code": "IDEMPOTENCY_CONFLICT",
    "message": "同一消息 ID 的请求内容不一致",
    "retryable": false,
    "retry_after_ms": 0,
    "details": {
      "object_id": "m_user_1"
    }
  }
}
```

客户端动作：停止冲突请求，恢复原始冻结负载并查询或重放原回执；不要用新 ID 掩盖同一轮请求的冲突。

### E23 BASE_SYNC_SEQ_CONFLICT

要删除的 m_user_1 仍存在，其正式 sync_seq 为108；客户端错误地把106作为该对象的版本前提。消息不支持编辑，本例不表示同一有效消息经过正文更新。

请求：

```http
POST /auto-media/api/chat/roles/role_1/scopes/messages/mutations
Authorization: Bearer <access_token>
X-Device-ID: device_1
X-Request-ID: req_e23
Content-Type: application/json

{
  "mutation_id": "mut_delete_1",
  "origin_sync_epoch": "msg-e1",
  "object_id": "m_user_1",
  "operation": "delete",
  "base_sync_seq": "106"
}
```

响应：`HTTP 409`。

```json
{
  "request_id": "req_e23",
  "server_time_ms": 1795410000000,
  "error": {
    "code": "BASE_SYNC_SEQ_CONFLICT",
    "message": "对象版本已变化",
    "retryable": false,
    "retry_after_ms": 0,
    "details": {
      "current_sync_seq": "108"
    }
  }
}
```

客户端动作：同步最新对象并由业务决定是否重新删除；原操作不改参数，若重新决定删除，应使用新的 mutation_id 和最新对象版本。

### E24 OBJECT_DELETED

历史记录的永久消息身份表明m_deleted_1已使用且被删除，但没有对应的可重放客户端message_create成功回执；例如它原本由服务端发布而不是由客户端batchCreate创建。若存在原成功回执，应优先回放成功，不返回本码或复活消息。

请求：

```http
POST /auto-media/api/chat/roles/role_1/messages:batchCreate
Authorization: Bearer <access_token>
X-Device-ID: device_1
X-Request-ID: req_e24
Content-Type: application/json

{
  "messages": [
    {
      "message_id": "m_deleted_1",
      "sender_type": "user",
      "sender_id": "user_1",
      "message_type": "text",
      "body": {
        "text": "尝试重新插入旧消息",
        "attachments": []
      },
      "created_at_ms": 1795410000000
    }
  ]
}
```

响应：`HTTP 409`。

```json
{
  "request_id": "req_e24",
  "server_time_ms": 1795410000000,
  "error": {
    "code": "OBJECT_DELETED",
    "message": "消息 ID 已使用且对象已删除",
    "retryable": false,
    "retry_after_ms": 0,
    "details": {
      "object_id": "m_deleted_1"
    }
  }
}
```

客户端动作：同步并保留删除结果，停止恢复该旧 ID。只有新的业务消息才使用新 message_id，不能重建已删除消息来掩盖本次冲突。

### E25 SYNC_EPOCH_MISMATCH

客户端读取旧消息 Epoch msg-e0，服务端当前已为 msg-e1。该例是普通 Scope 同步；v3 多模态创建请求不传 Epoch。

请求：

```http
POST /auto-media/api/chat/roles/role_1/scopes/messages/read-batches
Authorization: Bearer <access_token>
X-Device-ID: device_1
X-Request-ID: req_e25
Content-Type: application/json

{
  "mode": "incremental",
  "sync_epoch": "msg-e0",
  "cursor": "100"
}
```

响应：`HTTP 409`。

```json
{
  "request_id": "req_e25",
  "server_time_ms": 1795410000000,
  "error": {
    "code": "SYNC_EPOCH_MISMATCH",
    "message": "同步 Epoch 已变化",
    "retryable": false,
    "retry_after_ms": 0,
    "details": {
      "scope": "messages",
      "requested_sync_epoch": "msg-e0",
      "current_sync_epoch": "msg-e1"
    }
  }
}
```

客户端动作：停止并废弃旧 Epoch 的读取责任与迟到响应，清理对应缓存后重新建立基线；不要只替换 Epoch 并继续旧游标。

### E26 SYNC_CURSOR_EXPIRED

Epoch 仍是 msg-e1，但服务端的 min_valid_cursor 已为 200，旧游标 100 已不能保证墓碑连续性。

请求：

```http
POST /auto-media/api/chat/roles/role_1/scopes/messages/read-batches
Authorization: Bearer <access_token>
X-Device-ID: device_1
X-Request-ID: req_e26
Content-Type: application/json

{
  "mode": "incremental",
  "sync_epoch": "msg-e1",
  "cursor": "100"
}
```

响应：`HTTP 409`。

```json
{
  "request_id": "req_e26",
  "server_time_ms": 1795410000000,
  "error": {
    "code": "SYNC_CURSOR_EXPIRED",
    "message": "同步游标已过期",
    "retryable": false,
    "retry_after_ms": 0,
    "details": {
      "scope": "messages",
      "cursor": "100",
      "min_valid_cursor": "200",
      "current_sync_epoch": "msg-e1"
    }
  }
}
```

客户端动作：按业务重开 recent 或 full 读取并重建缓存；不要直接将本地游标改成 200 跳过清理。

### E27 DELETE_REQUIRES_MAINTENANCE

m_user_1 版本确为 106，但其根来源关联了 101 条记忆，超过原子删除最多 100 条关联记忆的保护预算。

请求：

```http
POST /auto-media/api/chat/roles/role_1/scopes/messages/mutations
Authorization: Bearer <access_token>
X-Device-ID: device_1
X-Request-ID: req_e27
Content-Type: application/json

{
  "mutation_id": "mut_delete_large",
  "origin_sync_epoch": "msg-e1",
  "object_id": "m_user_1",
  "operation": "delete",
  "base_sync_seq": "106"
}
```

响应：`HTTP 409`。

```json
{
  "request_id": "req_e27",
  "server_time_ms": 1795410000000,
  "error": {
    "code": "DELETE_REQUIRES_MAINTENANCE",
    "message": "删除超过原子操作预算，需要维护处理",
    "retryable": false,
    "retry_after_ms": 0
  }
}
```

客户端动作：停止自动重试并交由维护处理。本次删除整笔未执行，不能把消息或部分记忆标为已成功删除。

### E28 PAGE_TOKEN_EXPIRED

page_token 是此前合法签发并原样保存的令牌，但该设备随后重开了首页，令牌所绑定的会话排序视图已被替换。请求中的占位字符串必须替换成真实旧令牌；随意编造令牌可能得到 INVALID_ARGUMENT。

请求：

```http
POST /auto-media/api/chat/threads/query
Authorization: Bearer <access_token>
X-Device-ID: device_1
X-Request-ID: req_e28
Content-Type: application/json

{
  "page_size": 20,
  "page_token": "<previously_issued_page_token>"
}
```

响应：`HTTP 410`。

```json
{
  "request_id": "req_e28",
  "server_time_ms": 1795410000000,
  "error": {
    "code": "PAGE_TOKEN_EXPIRED",
    "message": "分页视图已失效",
    "retryable": false,
    "retry_after_ms": 0
  }
}
```

客户端动作：将 page_token 置 null，重新请求首页；保留仍有效的消息 Scope 游标和本地阅读位置。

### E29 PARTITION_TOO_LARGE

当前 Epoch 与游标均有效，服务端最大 sync_seq 至少为 500；请求区间 (100,500] 宽度为 400，超过单次上限 200。

请求：

```http
POST /auto-media/api/chat/roles/role_1/scopes/messages/records:query
Authorization: Bearer <access_token>
X-Device-ID: device_1
X-Request-ID: req_e29
Content-Type: application/json

{
  "mode": "incremental",
  "sync_epoch": "msg-e1",
  "base_cursor": "100",
  "source_max_cursor": "500",
  "lower_exclusive": "100",
  "upper_inclusive": "500"
}
```

响应：`HTTP 413`。

```json
{
  "request_id": "req_e29",
  "server_time_ms": 1795410000000,
  "error": {
    "code": "PARTITION_TOO_LARGE",
    "message": "区间过大，请拆分后重新查询",
    "retryable": false,
    "retry_after_ms": 0,
    "details": {
      "scope": "messages",
      "lower_exclusive": "100",
      "upper_inclusive": "500",
      "suggested_partition_span": 200
    }
  }
}
```

客户端动作：在本地同一事务中将原叶替换为 (100,300] 与 (300,500] 等更小区间，查询时保持原 base_cursor 和 source_max_cursor；拒绝原叶迟到响应，不应用部分结果。

### E30 OBJECT_TOO_LARGE

当前源位置至少为 500，区间 (100,101] 中仅有一条历史消息 m_oversized_1；其完整 SyncRecord 已超过 524288 字节。此例用于既有超大记录读取，已将区间缩到宽度 1。

请求：

```http
POST /auto-media/api/chat/roles/role_1/scopes/messages/records:query
Authorization: Bearer <access_token>
X-Device-ID: device_1
X-Request-ID: req_e30
Content-Type: application/json

{
  "mode": "incremental",
  "sync_epoch": "msg-e1",
  "base_cursor": "100",
  "source_max_cursor": "500",
  "lower_exclusive": "100",
  "upper_inclusive": "101"
}
```

响应：`HTTP 422`。

```json
{
  "request_id": "req_e30",
  "server_time_ms": 1795410000000,
  "error": {
    "code": "OBJECT_TOO_LARGE",
    "message": "单条同步记录超过大小限制",
    "retryable": false,
    "retry_after_ms": 0,
    "details": {
      "object_id": "m_oversized_1"
    }
  }
}
```

客户端动作：停止继续拆分或循环重试，修复该对象或交由维护处理；不要将失败区间当作空结果推进游标。

### E31 OPERATION_NOT_SUPPORTED

首期长期记忆只由服务端提取和发布，客户端不能通过 mutations 删除 memories。

请求：

```http
POST /auto-media/api/chat/roles/role_1/scopes/memories/mutations
Authorization: Bearer <access_token>
X-Device-ID: device_1
X-Request-ID: req_e31
Content-Type: application/json

{
  "mutation_id": "mut_memory_delete",
  "origin_sync_epoch": "memory-e1",
  "object_id": "memory_1",
  "operation": "delete",
  "base_sync_seq": "10"
}
```

响应：`HTTP 422`。

```json
{
  "request_id": "req_e31",
  "server_time_ms": 1795410000000,
  "error": {
    "code": "OPERATION_NOT_SUPPORTED",
    "message": "不支持客户端修改长期记忆",
    "retryable": false,
    "retry_after_ms": 0
  }
}
```

客户端动作：停止该 Scope 的客户端修改，检查调用类型和部署能力；本码不能用于判断其他已接受任务或候选是否取消。

### E32 RATE_LIMITED

同一用户并发数或服务总并发数已达到配置上限，本次被限流。

请求：

```http
POST /auto-media/api/chat/roles/role_1/messages:batchCreate
Authorization: Bearer <access_token>
X-Device-ID: device_1
X-Request-ID: req_e32
Content-Type: application/json

{
  "messages": [
    {
      "message_id": "m_user_rate",
      "sender_type": "user",
      "sender_id": "user_1",
      "message_type": "text",
      "body": {
        "text": "你好",
        "attachments": []
      },
      "created_at_ms": 1795410000000
    }
  ]
}
```

响应：`HTTP 429`。

```json
{
  "request_id": "req_e32",
  "server_time_ms": 1795410000000,
  "error": {
    "code": "RATE_LIMITED",
    "message": "请求过于频繁，请稍后重试",
    "retryable": true,
    "retry_after_ms": 1000
  }
}
```

客户端动作：至少等待 retry_after_ms 后重试原 batchCreate，保持消息 ID 与完整负载不变；不要因限流重新生成或创建第二条回复。

### E33 TEMPORARY_SERVER_ERROR

已收到generated候选，原样上传时Mongo写事务发生暂时故障或提交结果尚不能确认，本次API无法返回可靠成功回执。

请求：

```http
POST /auto-media/api/chat/roles/role_1/messages:batchCreate
Authorization: Bearer <access_token>
X-Device-ID: device_1
X-Request-ID: req_e33
Content-Type: application/json

{
  "messages": [
    {
      "message_id": "replymsg_post_1",
      "sender_type": "role",
      "sender_id": "role_1",
      "message_type": "text",
      "body": {
        "text": "想放松一下的话，可以看看我之前分享的海边散步。你更喜欢静静看海，还是沿着海岸走走？",
        "attachments": []
      },
      "created_at_ms": 1795410010000,
      "proactive_context": {
        "schema_version": 1,
        "trigger_type": "post_link",
        "link_type": "post",
        "post_id": "post_123",
        "task_id": "task_post_1"
      }
    }
  ]
}
```

响应：`HTTP 503`。

```json
{
  "request_id": "req_e33",
  "server_time_ms": 1795410020000,
  "error": {
    "code": "TEMPORARY_SERVER_ERROR",
    "message": "服务暂时不可用，请重试原请求",
    "retryable": true,
    "retry_after_ms": 0
  }
}
```

客户端动作：按退避策略重试同一 batchCreate，保持 message_id、正文、时间与 metadata 不变。原请求可能已经提交；不要换 ID 或重新生成消息。

同一码也可能出现在创建阶段，例如任务登记数据库暂时不可用。此时恢复原创建POST，不能误改为上传或普通回复。`POST /auto-media/api/chat/roles/role_1/multimodal-tasks`，`X-Request-ID: req_e33_create`，请求正文：

```json
{
  "content": "用户：最近有点累。角色：忙完想去哪里放松？用户：想找个安静的地方，最好能看看海和晚霞。",
  "reply_message_id": "replymsg_post_1",
  "task_type": "post_link",
  "options": {}
}
```

HTTP 503：

```json
{
  "request_id": "req_e33_create",
  "server_time_ms": 1795410001000,
  "error": {
    "code": "TEMPORARY_SERVER_ERROR",
    "message": "任务登记服务暂时不可用",
    "retryable": true,
    "retry_after_ms": 1000
  }
}
```

客户端动作：等待至少1000毫秒，保持原content、reply_message_id和其他业务参数，重试原创建POST；不并行生成另一条回复。

### E34 MESSAGE_ID_RESERVED

m_reply_reserved已被多模态任务预留，任务已generated；客户端却遗漏返回候选中的proactive_context，将该reply ID当作普通role消息上传。

请求：

```http
POST /auto-media/api/chat/roles/role_1/messages:batchCreate
Authorization: Bearer <access_token>
X-Device-ID: device_1
X-Request-ID: req_e34
Content-Type: application/json

{
  "messages": [
    {
      "message_id": "m_reply_reserved",
      "sender_type": "role",
      "sender_id": "role_1",
      "message_type": "text",
      "body": {
        "text": "客户端普通回复",
        "attachments": []
      },
      "created_at_ms": 1795410000000
    }
  ]
}
```

响应：`HTTP 409`。

```json
{
  "request_id": "req_e34",
  "server_time_ms": 1795410000000,
  "error": {
    "code": "MESSAGE_ID_RESERVED",
    "message": "消息 ID 已被服务端任务保留",
    "retryable": false,
    "retry_after_ms": 0
  }
}
```

客户端动作：查询原任务，恢复带完整proactive_context的原候选message并冻结上传，不能将预留ID当普通消息使用；已经generated不能自动另生成回复。只有确认未产候选且已按错误表允许普通回复接管时，才使用新的普通消息ID。

### E35 POST_VECTOR_SCHEMA_INVALID

仅在部署允许向客户端直出底层诊断时适用：配置期望 post_vectors.embedding 使用 COSINE，但现有索引实际为 IP，Schema 校验拒绝检索。这不是未命中 post；正常创建流程通常将底层失败包装为 POST_RETRIEVAL_FAILED，客户端优先按实际收到的外层 code 处理。

请求：

```http
POST /auto-media/api/chat/roles/role_1/multimodal-tasks
Authorization: Bearer <access_token>
X-Device-ID: device_1
X-Request-ID: req_e35
Content-Type: application/json

{
  "content": "user: 最近想去海边散步。",
  "reply_message_id": "m_reply_schema",
  "task_type": "post_link"
}
```

响应：`HTTP 503`。

```json
{
  "request_id": "req_e35",
  "server_time_ms": 1795410000000,
  "error": {
    "code": "POST_VECTOR_SCHEMA_INVALID",
    "message": "post 向量索引与服务配置不一致",
    "retryable": false,
    "retry_after_ms": 0,
    "details": {
      "field": "embedding",
      "expected_metric": "COSINE",
      "actual_metrics": [
        "IP"
      ]
    }
  }
}
```

客户端动作：停止盲目重复调用，由服务端修复依赖与原冻结配置的一致性，再按原业务ID和原生成期限恢复责任，不改写旧请求的冻结metric。仅将环境POST_VECTOR_METRIC从COSINE改成IP，只影响后续新登记请求，不能让本例原请求切换metric重新检索；原责任仍须查证并按原期限终止，确认未产候选的终态后才按错误表使用新普通消息ID接管，不能重建同轮post任务。若实际收到的是POST_RETRIEVAL_FAILED，先按外层错误重试原创建请求，不把配置故障当作no_match。

### 上传结果未知（没有权威错误码）

客户端已将generated返回的原message通过batchCreate上传，但连接超时、响应丢失或收到无法校验的非JSON响应。此时没有可用error.code，不能在本地伪造MULTIMODAL_GENERATION_FAILED或认定整批未提交。

客户端动作：保留原message_id、正文、created_at_ms、proactive_context及首次冻结metadata，重试原batchCreate消息负载；公共追踪ID和既有动态快照版本按原规则更新。查询同task仅辅助观察，generated也不能单独证明刚才上传未提交。服务端若已提交则回放原成功逐条回执，客户端不换ID、不重建任务、不生成第二条回复。MULTIMODAL_TASK_EXPIRED和MULTIMODAL_REQUEST_EXPIRED都不代表候选上传超时。
