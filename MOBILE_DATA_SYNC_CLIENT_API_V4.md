# V4 客户端接口文档

适用于移动端接入。通用同步协议见[移动端数据同步接口设计 V4](MOBILE_DATA_SYNC_DESIGN_V4.md)。长期记忆采用[长期记忆设计](CHAT_MEMORY_DESIGN_FINAL.md)定义的 `memories` Scope。网红人设采用[网红资料同步设计](ROLE_PROFILE_SYNC_DESIGN.md)，通过消息创建回包交付。更新于 2026-11-05。

| 方法与路径 | 用途 |
|---|---|
| POST /auto-media/api/chat/threads/query | 会话列表、预览及本机阅读基准下的未读数 |
| GET /auto-media/api/chat/roles/{role_id}/sync-state | 可选轻量探测，不是同步读取前置步骤 |
| POST /auto-media/api/chat/roles/{role_id}/messages:batchCreate | 幂等创建用户消息或端侧回复，接收回复来源，按版本返回完整网红人设 |
| POST /auto-media/api/chat/roles/{role_id}/messages:batchGet | 按消息 ID 批量查询当前会话的有效消息正文 |
| POST /auto-media/api/chat/roles/{role_id}/scopes/messages/mutations | 幂等删除一条消息及其关联记忆 |
| GET /auto-media/api/chat/roles/{role_id}/scopes/messages/mutations/{mutation_id} | 查询消息删除状态及历史回执 |
| POST /auto-media/api/chat/roles/{role_id}/scopes/{scope}/read-batches | 创建本轮读取batch；同响应给边界、建议跨度及可选内联数据 |
| POST /auto-media/api/chat/roles/{role_id}/scopes/{scope}/records:query | 按显式参数完整查询一个区间 |
| POST /auto-media/api/chat/roles/{role_id}/multimodal-tasks | 提交服务端生成任务 |
| GET /auto-media/api/chat/roles/{role_id}/multimodal-tasks/{task_id} | 查询生成状态及当前仍有效的回复 |
| POST /auto-media/api/chat/roles/{role_id}/multimodal-tasks/{task_id}:cancel | 幂等取消业务生成任务 |

## 1 公共类型、Header、路径与响应

字段标为必填时必须出现。可为 null 的字段会单独标明。

### 1.1 公共类型

| 类型 | JSON表示 | 含义与校验规则 |
|---|---|---|
| `ID` | 非空字符串，UTF-8最多128字节 | 业务或资源标识，例如 `"msg_01"`。`user_id`、`role_id`、`message_id`、`object_id`、`asset_id`、`mutation_id`、`task_id` 都使用字符串，不按数字解析，不比较大小。具体生成方在对应字段中说明。 |
| `UInt64String` | 十进制数字字符串 | 非负 64 位无符号整数的线格式，例如 `"0"`、`"104"`。用于同步序号、游标、`thread_revision` 及两个快照的数字版本。首期聊天同步的序号和位置上限为 `9223372036854775807`。客户端按整数精确解析和比较。 |
| `OpaqueString` | 非空字符串，UTF-8最多128字节 | 不透明标识。`sync_epoch`、`messages_sync_epoch`、`receipt_sync_epoch` 等按完整字符串判断相等。 |
| `TimestampMs` | JSON整数 | Unix纪元起的毫秒时间戳，非负；不是十进制数字字符串。`created_at_ms`、`server_created_at_ms`、`server_time_ms`等均使用此类型；V4读取batch不返回expires_at_ms。 |
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
| `error.retryable` | 布尔值 | 失败时必填 | 是否允许按当前业务参数自动重试。需要重建、修改参数、处理冲突或迁移的错误，不能仅因网络重试而解决。 |
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

这是下行upsert、完整当前任务结果中使用的权威模型。消息创建请求仅包含消息批量创建接口定义的输入字段，不能因此上送服务端分配的字段。

| 字段 | 类型 | 必填条件 | 含义及来源规则 |
|---|---|---|---|
| `message_id` | `ID` | 必填 | 消息的永久业务ID，等于同步外层`object_id`。生成端首次持久化前确定；用户输入与回复使用不同ID。 |
| `sender_type` | 字符串枚举 | 必填 | `user`或`role`，表示实际业务发言方。不能用它推断消息从上行还是下行获得。 |
| `sender_id` | `ID` | 必填 | user时必须等于认证用户ID，role时必须等于当前会话`role_id`。由生成端提供、服务端按权限及发送者类型校验。 |
| `origin` | 字符串枚举 | 必填 | 服务端确认的生成来源。客户端创建入口按sender_type推导，user为`client_user`，role为`client_llm`；服务端模型和多模态任务分别为`server_llm`和`server_multimodal`。 |
| `reply_to_message_id` | `ID`或`null` | 字段必填，可为null | 明确关联的同会话用户输入ID。新创建的客户端user和role消息均为null；服务端任务有明确输入时填写该输入ID。历史消息已记录的关联继续保留。原输入删除后仍保留这个关联；本地未加载不等于原消息已删除。 |
| `message_type` | 字符串枚举 | 必填 | `text`、`image`、`audio`、`video`或`mixed`；必须与body中的正文及附件类型一致。 |
| `body` | 对象，见2.3 | 必填 | 文本及媒体附件的完整内容容器。至少有非空文本或一个有效附件。 |
| `created_at_ms` | `TimestampMs` | 必填 | 首次生成消息的时间，用于展示。重试时沿用原值，排序使用 object_seq。 |
| `server_created_at_ms` | `TimestampMs` | 已确认下行必填；创建请求不传 | 服务端首次接受消息的时间。幂等重试返回首次值，不用本次回包时间替代。 |

发送者、来源、回复关联、首次时间和稳定消息顺序在创建后不可改。本版messages不提供update；删除只作用于选中消息，已存在的回复保留。

消息对象的创建位置不属于MessageContent：SyncRecord、创建回执及TaskState均在其外层返回object_seq，避免正文重复元数据。

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
| `intimacy_score` | 对象 | 服务端需返回该结构新完整版本时出现 | 当前会话亲密度的完整快照；与structured_information独立更新。 |
| `intimacy_score.version` | `UInt64String` | 结构出现时必填 | 亲密度结构自己的整体数字版本，不与structured_information.version横向比较。 |
| `intimacy_score.score` | `JSONValue` | 结构出现时必填 | 亲密度模块定义的完整业务值，可为数值或模块定义的结构；合法类型及空值含义由业务模块确定，同步层不发明分数字段、量表或内部schema。 |

两个快照独立处理。人设通过消息创建回包获取，`read-batches` 和 `records:query` 只读取其路径指定的同步 Scope。`persona_features` 由服务端组装，独立提取的语言、展示名和媒体属性有各自的用途。端侧生成时，按固定键顺序遍历快照的顶层字段，逐项拼接为 `key: value`。字符串直接使用，其余值用 JSON 表示。

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
| `intimacy_score.score` | `JSONValue` | 由亲密度模块定义，完整业务值按其独立Version替换。 |
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

位置字段只用于发现变化。本接口是可选探测，客户端也可以直接调用 read-batches。

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
| `messages` | 对象数组 | 必填，至少 1 项 | 本次原子创建的消息，同一 `message_id` 不可重复。端侧 role 消息可单独创建，生成来源按第 5.1 节显式申报。 |
| `messages[].message_id` | ID | 必填 | 生成端首次持久化时确定的永久消息ID，也是创建幂等键。网络重试不变。 |
| `messages[].sender_type` | user或role | 必填 | 业务发言方；不等于生成设备。 |
| `messages[].sender_id` | ID | 必填 | user时等于认证用户，role时等于路径role_id。服务端校验，不接受任意发送者。 |
| `messages[].message_type` | 字符串枚举 | 必填 | text/image/audio/video/mixed；含义及与body的匹配规则见2。 |
| `messages[].body` | MessageBody | 必填 | 文本和媒体引用。body.text、body.attachments及附件字段逐项见2，至少有非空文本或有效附件。 |
| `messages[].created_at_ms` | TimestampMs | 必填 | 生成端首次保存时间，重试固定；不负责服务器确认后的消息排序。 |
| `messages[].metadata.context_provenance` | ContextProvenanceV1 | 新客户端创建端侧 role 回复时发送，兼容旧请求省略 | 本次生成实际注入的近期消息和长期记忆来源。随回复 ID 冻结并参与消息幂等比较。字段见第 5.1 节。 |
| `sync_positions` | 对象数组 | 可省略，默认[] | 本机已完成的 Scope 读取位置，供生成同步提示。独立于冻结消息及其业务幂等摘要。 |
| `sync_positions[].scope` | Scope | 每项必填 | 此位置所属数据类型，同Scope最多一项。 |
| `sync_positions[].sync_epoch` | OpaqueString或null | 每项必填；无基线为null | 本机该Scope的代次，用于判断提示是否要求重建。 |
| `sync_positions[].cursor` | UInt64String | 每项必填；无基线为"0" | 本机已完整应用的读取位置，不是本批待上传消息的Seq。 |
| `session_messages_sync_epoch` | OpaqueString 或 null | 可省略，默认 null | 当前会话已知的 messages Epoch。Epoch 不匹配时服务端将亲密度请求版本按 `"0"` 处理。 |
| `structured_information_version` | UInt64String | 可省略，默认 `"0"` | 当前会话内存中的网红人设版本。每次发送或重试时取当前值，无快照传 `"0"`。 |
| `intimacy_score_version` | UInt64String | 可省略，默认 `"0"` | 当前会话内存中的亲密度版本，与人设版本独立。 |

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
| `intimacy_score` | IntimacyScoreSnapshot | 条件返回 | 按亲密度请求版本和会话 Epoch 独立判断，返回完整业务值。字段见第 2.6 节。 |

服务端按 `sender_type` 确定 `origin`，user 对应 `client_user`，role 对应 `client_llm`。新建客户端消息的 `reply_to_message_id` 为 null，实际生成依赖由 `context_provenance` 单独表达。`object_seq`、`sync_seq`、`server_created_at_ms` 由服务端在回执中给出。

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

完整且可验证的声明可参与自动记忆提取。完整声明中有合法待绑定引用或暂时未解析依赖时，等待服务端解析。部分、不可用或永久失效的来源仍可保留回复展示，回复不作为自动提取证据。生成中或提交前发现依赖已失效时，停止生成或提交，按有效上下文重建。

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

首次真实用户消息成功创建后，可以使用本次响应处理后的快照进行端侧生成。缺少人设时可走服务端生成路径。接口始终要求至少一条合法消息。

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
    "score": 0
  }
}
```

后续请求带 `"1"` 且服务端仍为 `"1"` 时，成功回包省略人设。若角色重建后发布空人设版本 `"2"`，请求带 `"1"` 时回传 `{"version":"2","data":{}}`。请求误带 `"9"` 时也回传版本 `"2"`，按第 5.2 节应用纠正。

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
          "origin": "server_llm",
          "reply_to_message_id": "msg_99",
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

### 12.1 创建任务

`POST /auto-media/api/chat/roles/{role_id}/multimodal-tasks`。

| 请求字段 | 类型 | 必填条件 | 含义与使用规则 |
|---|---|---|---|
| `messages_sync_epoch` | OpaqueString | 必填 | 生成所依据的输入消息代次，任务完成写入前还要再次匹配。 |
| `input_message_id` | ID | 必填 | 已由服务端确认、仍有效的同会话用户消息；不得引用其他用户/角色消息。 |
| `input_sync_seq` | UInt64String | 必填 | 创建任务时所依据输入的版本，不是object_seq；输入删除/变化后不能发布过期结果。 |
| `reply_message_id` | ID | 必填 | 开始生成前确定的新回复ID，也是本次生成业务幂等键，与input_message_id不同。 |
| `task_type` | string | 必填 | 服务端支持的多模态任务类型。具体类型由任务模块能力定义，未支持的值拒绝，不能作为任意命令执行。 |
| `options` | JSONObject | 可省略，默认{} | 影响本次生成的固定业务参数，参与幂等比较。内部字段由该task_type的业务Schema规定；不能用同输入ID另外传一份不同正文绕过已确认输入。 |

创建结果包含task_id/status/reply_message_id及公共响应字段，其他TaskState字段在实际可用时提供。reply_message_id相同但参数不同返回幂等冲突。

假设任务模块支持 `image_generation`，并允许空 `options`。

`POST /auto-media/api/chat/roles/role_1/multimodal-tasks`，本次请求的 `X-Request-ID` 为 `req_task_1`。请求正文如下。

```json
{
  "messages_sync_epoch": "msg-e1",
  "input_message_id": "msg_101",
  "input_sync_seq": "106",
  "reply_message_id": "msg_img_1",
  "task_type": "image_generation",
  "options": {}
}
```

响应：

```json
{
  "request_id": "req_task_1",
  "server_time_ms": 1793670000000,
  "task_id": "task_1",
  "status": "queued",
  "reply_message_id": "msg_img_1"
}
```

### 12.2 查询任务与结果

`GET /auto-media/api/chat/roles/{role_id}/multimodal-tasks/{task_id}`。无请求正文。创建和查询使用以下TaskState字段：

| 响应字段 | 类型 | 出现条件 | 含义与使用规则 |
|---|---|---|---|
| `task_id` | ID | 必填 | 服务端分配的异步资源ID，用于查询与取消；不等于消息ID。 |
| `status` | queued/running/completed/cancelled/failed | 必填 | 分别表示已排队、正在处理、结果已提交、已取消、生成失败。completed只表示任务曾完成，不表示回复现在一定仍存在。 |
| `reply_message_id` | ID | 必填 | 本任务绑定的稳定回复消息ID，不因查询或计算重试改变。 |
| `reply_sync_epoch` | OpaqueString | completed必填，其他省略 | 回复首次落库的messages代次；不能用当前Epoch包装历史结果。 |
| `message` | MessageContent或null | 查询/取消时必填；创建可省略 | 仅在回复属于当前代次且当前权威对象仍有效时返回完整正文，其他情况null；不回放已删除的首次制品。消息字段见2。 |
| `object_seq` | UInt64String或null | 查询/取消时必填；创建可省略，但message非空时必填且非null | 回复的不可变创建号，位于TaskState同层；message为空时为null。与正式SyncRecord.object_seq及创建回执相同，不自动标读。 |
| `sync_seq` | UInt64String或null | 查询/取消时必填；message非空必须非null | 返回正文当前对象版本；message为空时为null。它不是可以直接写入本机cursor的位置。 |
| `sync_hints` | SyncHint数组 | 查询/取消时必填，可为[]；创建可省略 | 当前服务端各Scope的变化提示，与任务历史回执分开；不足以证明本机已下载结果。 |
| `error` | Error | status=failed必填；其他省略 | 已终止任务的原因；HTTP200可以成功查询到失败任务，内部字段见1。 |

临时任务正文即使存在，也不代替区间读取的权威合并；不推进cursor、历史锚点或标读边界。message=null不应被通用代码随意解释成“删除整个会话”。

任何创建/查询/取消响应只要提供非null message，就必须同时提供非null object_seq和sync_seq；若message=null，两者均为null（查询/取消必返）。临时任务数据仍不能推进权威扫描位置。

`GET /auto-media/api/chat/roles/role_1/multimodal-tasks/task_1`，本次请求的 `X-Request-ID` 为 `req_task_2`。无请求正文。

响应（HTTP 200）：

```json
{
  "request_id": "req_task_2",
  "server_time_ms": 1793670000000,
  "task_id": "task_1",
  "status": "completed",
  "reply_message_id": "msg_img_1",
  "reply_sync_epoch": "msg-e1",
  "message": {
    "message_id": "msg_img_1",
    "sender_type": "role",
    "sender_id": "role_1",
    "origin": "server_multimodal",
    "reply_to_message_id": "msg_101",
    "message_type": "image",
    "body": {
      "text": null,
      "attachments": [
        {
          "asset_id": "asset_1",
          "media_type": "image",
          "mime_type": "image/jpeg"
        }
      ]
    },
    "created_at_ms": 1793669999995,
    "server_created_at_ms": 1793669999999
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

### 12.3 取消任务

`POST /auto-media/api/chat/roles/{role_id}/multimodal-tasks/{task_id}:cancel`。不需要新业务幂等ID，请求无业务正文或使用空对象{}。返回HTTP200及TaskState；queued/running任务取消成功后为cancelled，已completed则返回既有完成状态，不删除用户要求保留的回复。重复取消不创建新任务。

这里取消另一个仍在排队的任务 `task_2`。

`POST /auto-media/api/chat/roles/role_1/multimodal-tasks/task_2:cancel`，本次请求的 `X-Request-ID` 为 `req_task_3`。请求正文如下。

```json
{}
```

响应（HTTP 200）：

```json
{
  "request_id": "req_task_3",
  "server_time_ms": 1793670000000,
  "task_id": "task_2",
  "status": "cancelled",
  "reply_message_id": "msg_img_2",
  "message": null,
  "object_seq": null,
  "sync_seq": null,
  "sync_hints": []
}
```

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
| `action` | string，可选 | 恢复提示，如 `retry_smaller_range`、`rebuild_recent`、`rebuild_full`。最终行为仍按错误码和本地任务状态判断。 |
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

| HTTP / code | 恢复行为 |
|---|---|
| 400 INVALID_ARGUMENT / INVALID_READ_POSITION | 修正参数；包括格式错误、被篡改或与分页参数不匹配的page_token，以及权威源确认的未来游标或不合法区间 |
| 401 TOKEN_EXPIRED | 刷新登录凭证，保留仍有效的本地进度 |
| 403 FORBIDDEN | 停止相应会话读写，清除已撤销权限会话的内存快照，保留 Outbox |
| 404 NOT_FOUND | 业务Mutation/生成任务不存在或不可访问 |
| 409 IDEMPOTENCY_CONFLICT | 同业务操作ID使用了不同冻结参数；创建读取batch没有该幂等语义 |
| 409 BASE_SYNC_SEQ_CONFLICT / OBJECT_DELETED | 同步后交业务处理，保留草稿/原操作 |
| 409 SYNC_EPOCH_MISMATCH | 停止旧任务，重建相应Scope |
| 409 SYNC_CURSOR_EXPIRED | 读取位置已过期。重新创建 recent 或 full |
| 409 DELETE_REQUIRES_MAINTENANCE | 删除超出事务保护预算，整笔未执行，停止自动重试并交内部维护 |
| 410 PAGE_TOKEN_EXPIRED | 会话列表分页标识或绑定集合已过期、被回收；丢弃该轮分页进度，page_token重置为null，携带当前local_read_positions从第一页重新查询 |
| 413 PARTITION_TOO_LARGE | 不返回部分records；给建议跨度/区间，客户端自行拆分 |
| 422 OBJECT_TOO_LARGE | 单对象违反硬限，停止并修复，不无限拆分 |
| 422 OPERATION_NOT_SUPPORTED | 当前 Scope 不支持该业务操作，包括客户端写入长期记忆 |
| 429 RATE_LIMITED / 503 TEMPORARY_SERVER_ERROR | 按 `retry_after_ms` 有界退避。读源落后时保留当前读取进度。消息创建结果未知时重试原冻结消息 ID 和正文，人设请求版本按当前会话重新获取 |
