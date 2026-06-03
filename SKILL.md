# Skill: 会议纪要自动化 · 完整闭环（Meeting Flow）

> 版本：v2.0（bitable 表单入口 + 群聊事件驱动）
> 配置说明见同目录 README.md

---

## ⚙️ 配置区（使用前必填）

使用前将以下所有占位符替换为你的实际值：

```
YOUR_USER_OPEN_ID       → 你的飞书 open_id（格式：ou_xxxxxxxx）
YOUR_GROUP_CHAT_ID      → 工作群 chat_id（格式：oc_xxxxxxxx）
YOUR_BITABLE_APP_TOKEN  → 多维表格 app_token
YOUR_TABLE_ID           → 数据表 table_id（tbl 开头）
YOUR_FORM_VIEW_ID       → 表单视图 view_id（vew 开头）
YOUR_FORM_URL           → 表单完整链接
YOUR_VOLCENGINE_APP_ID  → 火山引擎录音识别 APP_ID
YOUR_VOLCENGINE_TOKEN   → 火山引擎录音识别 ACCESS_TOKEN
YOUR_CODEX_PATH         → Codex CLI 完整路径（如 ~/.nvm/versions/node/v22.x.x/bin/codex）
YOUR_PROXY_PORT         → 代理端口（不需要代理可删除代理相关两行）
YOUR_OBSIDIAN_PATH      → Obsidian 会议纪要存放路径
```

---

## 这个 Skill 是什么

会议纪要的完整自动化流程。从说"记会议纪要"开始，到纪要写好回填结束，全在这一个 skill 里管。

三个 prompt 模板在各自的 skill 里：
- `meeting-business` → 商务版 prompt
- `meeting-detail` → 详细版 prompt
- `meeting-action` → 行动项版 prompt（默认）

> **调用说明**：以上三个 skill 仅提供 prompt 框架，Codex 调用和结果写入均由本 skill 统一控制。

---

## 整体架构

```
你私聊说"记会议纪要"
       ↓
Bot 弹飞书卡片（表单链接）
       ↓
填写表单：会议名称 + 模板选择 + 上传录音
       ↓
提交到飞书多维表格
       ↓
多维表格自动化 → webhook 通知到工作群 → @Bot
       ↓
Bot 在群里收到通知 → 从 bitable 获取待处理记录
       ↓
获取录音公网直链 → 豆包 ASR 转写（带说话人分离）
       ↓
读取对应 prompt skill → Codex CLI 生成纪要
       ↓
创建飞书文档 + 写 Obsidian + 回填 bitable + 群里通知完成
```

---

## 关键资源

| 资源 | 值 |
|------|-----|
| 多维表格 app_token | `YOUR_BITABLE_APP_TOKEN` |
| 数据表 table_id | `YOUR_TABLE_ID` |
| 表单视图 view_id | `YOUR_FORM_VIEW_ID` |
| 表单链接 | `YOUR_FORM_URL` |
| 工作群 chat_id | `YOUR_GROUP_CHAT_ID` |
| 你的 open_id | `YOUR_USER_OPEN_ID` |

---

## 阶段一：私聊入口（弹卡片）

### 触发关键词
"会议纪要"、"帮我写会议纪要"、"记会议纪要"、"我要记会议"

### 动作
用 `exec` + `lark-cli` 发飞书互动卡片（**禁止用 message 工具的 card 参数**）：

```python
python3 -c "
import json, subprocess

form_url = 'YOUR_FORM_URL'

card = {
    'config': {'wide_screen_mode': True},
    'header': {
        'title': {'tag': 'plain_text', 'content': '📝 会议纪要'},
        'template': 'blue'
    },
    'elements': [
        {'tag': 'div', 'text': {'tag': 'lark_md', 'content': '点击下方按钮打开表单，填写会议信息并上传录音文件。'}},
        {'tag': 'action', 'actions': [{'tag': 'button', 'text': {'tag': 'plain_text', 'content': '📋 填写会议纪要表单'}, 'type': 'primary', 'url': form_url}]},
        {'tag': 'hr'},
        {'tag': 'div', 'text': {'tag': 'lark_md', 'content': '**流程说明**\n1. 填写会议名称，选择模板类型\n2. 上传录音文件\n3. 提交后系统自动处理（ASR → 生成纪要）\n4. 处理完成后纪要链接会回填到表格\n\n⏱ 通常 5-15 分钟内完成'}},
    ]
}

body = {
    'receive_id': 'YOUR_USER_OPEN_ID',
    'msg_type': 'interactive',
    'content': json.dumps(card, ensure_ascii=False)
}

subprocess.run([
    'lark-cli', 'api', 'POST', '/open-apis/im/v1/messages',
    '--params', json.dumps({'receive_id_type': 'open_id'}),
    '--as', 'bot',
    '--data', json.dumps(body, ensure_ascii=False)
])
"
```

发完卡片后简短确认："卡片已发，点按钮填表单就行。"

---

## 阶段二：群聊事件驱动

### 触发条件
在工作群收到包含"录音文件更新提醒"的消息。

群聊配置：`requireMention: false`（需在 openclaw.json 里直接配置，见 README）。

### 识别逻辑
收到群消息后判断：
1. 是否包含"录音文件更新提醒"？
2. 是否包含 `feishu.cn/record/` 链接？

都满足 → 进入处理流程。普通聊天 → 按群聊规则处理。

### 获取待处理记录

捞"空"和"失败"两种状态（"失败"是显式失败后打的标，可重试）：

```
feishu_bitable_app_table_record.list(
  app_token="YOUR_BITABLE_APP_TOKEN",
  table_id="YOUR_TABLE_ID",
  filter={"conjunction": "and", "conditions": [
    {"field_name": "录音文件", "operator": "isNotEmpty"},
    {"field_name": "状态", "operator": "isNoneOf", "value": ["处理中", "已完成"]}
  ]}
)
```

> **"处理中"卡死的人工恢复**：如果某条记录状态停在"处理中"超过 30 分钟，说明 Bot 崩溃时未完成清理。在 bitable 里手动将该记录状态改为"失败"，下次触发时会自动重试。

### 事件驱动踩坑

⚠️ **坑14：webhook 通知发到群但 Bot 收不到**
- 飞书 websocket 只把消息推给对应 app；webhook 机器人（另一个 app）发的消息不会触发 Bot 的事件监听
- 解决：bitable 自动化通知时配置 @Bot，配合 `requireMention: false` 确保消息能被处理

⚠️ **坑15：`requireMention` 是 protected path，不能用 gateway 改**
- `gateway config.patch` 无法修改 `groups.*.requireMention`，命令不报错但设置不生效
- 解决：直接编辑 `openclaw.json` 对应群的配置，改完重启

⚠️ **坑16：群聊默认 `visibleReplies: "message_tool"`，回复不会自动发出**
- 即使 Bot 在群里收到消息并处理完成，用 message tool 回复也不会实际发送
- 解决：处理完成后必须主动用 `lark-cli` 在群里发消息通知，不能依赖自动回复

---

## 阶段三：ASR 转写（详细步骤 + 踩坑）

这一步坑最多。每一步都有具体注意事项，严格按步骤执行。

### 3.1 从 bitable 记录提取 file_token

阶段二查询到的记录中，"录音文件"字段（Attachment 类型，type=17）返回格式：

```json
{
  "录音文件": [
    {
      "file_token": "xxxxxxxxxxxxxxx",
      "name": "录音.m4a",
      "size": 39558344,
      "type": "audio/x-m4a",
      "tmp_url": "https://open.feishu.cn/open-apis/drive/v1/medias/batch_get_tmp_download_url?file_tokens=xxx"
    }
  ]
}
```

**提取方式**：
```python
file_token = record["fields"]["录音文件"][0]["file_token"]
file_name  = record["fields"]["录音文件"][0]["name"]
file_size  = record["fields"]["录音文件"][0]["size"]
```

⚠️ **坑1：`tmp_url` 字段名有误导性**
- 它不是下载链接，而是「获取下载链接的 API 端点」
- 不能直接 curl 下载，必须再调 API

⚠️ **坑2：附件字段是数组**
- `record["fields"]["录音文件"]` 是数组，取 `[0]` 拿第一个文件

### 3.2 用 file_token 获取公网直链

用 lark-cli 调 API，**file_tokens 参数是数组**：

```bash
lark-cli api GET "/open-apis/drive/v1/medias/batch_get_tmp_download_url" \
  --params '{"file_tokens":["your_file_token_here"]}' \
  --as bot
```

返回：
```json
{
  "tmp_download_urls": [{
    "file_token": "xxxxxxxxxxxxxxx",
    "tmp_download_url": "https://internal-api-drive-stream.feishu.cn/space/api/box/stream/download/authcode/?code=xxxxx"
  }]
}
```

提取真正的下载 URL：
```python
tmp_download_url = result["tmp_download_urls"][0]["tmp_download_url"]
```

**这个 URL 的特点**：
- ✅ 无需认证，curl 直接 200 OK
- ✅ 国内 CDN，豆包 ASR（火山引擎）可直接访问
- ✅ 支持大文件
- ⚠️ 有时效性，获取后尽快使用

⚠️ **坑3：为什么只能走这条路**
- 飞书云盘文件 → 需要 Bearer token → 豆包 ASR 不带认证头 → 403
- Catbox/Litterbox 等国外托管 → 豆包服务器在国内被墙 → download error
- tmpfiles.org → 需要认证
- gofile.io → 302 重定向，豆包不跟随
- **只有多维表格附件的 tmp_download_url 满足「无认证 + 国内CDN + 大文件」**

### 3.3 提交 ASR 任务（Python requests）

**Endpoint**：`POST https://openspeech.bytedance.com/api/v1/auc/submit`

```python
import requests, json

APP_ID       = "YOUR_VOLCENGINE_APP_ID"
ACCESS_TOKEN = "YOUR_VOLCENGINE_TOKEN"
CLUSTER      = "volc_auc_common"
USER_UID     = "YOUR_USER_OPEN_ID"   # 任意字符串即可，用于请求追踪

def submit_asr(audio_url):
    url = "https://openspeech.bytedance.com/api/v1/auc/submit"
    headers = {
        "Content-Type": "application/json",
        "Authorization": f"Bearer; {ACCESS_TOKEN}"
    }
    body = {
        "app": {"appid": APP_ID, "token": ACCESS_TOKEN, "cluster": CLUSTER},
        "user": {"uid": USER_UID},
        "audio": {"format": "m4a", "url": audio_url},
        "additions": {
            "use_punc": "True",
            "use_itn": "True",
            "with_speaker_info": "True"   # 开启说话人分离
        }
    }
    resp = requests.post(url, headers=headers, data=json.dumps(body))
    return resp.json()["resp"]["id"]   # 返回 task_id

task_id = submit_asr(tmp_download_url)
```

⚠️ **坑4：说话人分离参数名（最重要的坑）**
- ✅ 正确：`with_speaker_info`（放在 `additions` 里）
- ❌ 全部被 API 静默忽略，不报错：`speaker_diarization`、`enable_speaker_info`、`enable_diarization`、`with_speaker`
- 用错参数不报错，但返回结果里没有说话人标记

⚠️ **坑5：submit 和 query 的参数位置不一样**
- submit：`{"app": {"appid":..., "token":..., "cluster":...}, ...}`（嵌套在 app 里）
- query：`{"appid":..., "token":..., "cluster":..., "id":...}`（外层平铺）
- 搞混了会报 `appid does not exist`

### 3.4 轮询 ASR 结果

**Endpoint**：`POST https://openspeech.bytedance.com/api/v1/auc/query`

```python
import time

def query_asr(task_id, max_wait=600):
    url = "https://openspeech.bytedance.com/api/v1/auc/query"
    headers = {"Content-Type": "application/json", "Authorization": f"Bearer; {ACCESS_TOKEN}"}
    # ⚠️ 注意：query 的 appid/token/cluster 在外层，不嵌套在 app 里
    body = {"appid": APP_ID, "token": ACCESS_TOKEN, "cluster": CLUSTER, "id": task_id}

    start = time.time()
    while time.time() - start < max_wait:
        resp = requests.post(url, headers=headers, data=json.dumps(body))
        result = resp.json()
        code = result.get("resp", {}).get("code", -1)

        if code == 1000:
            data = result["resp"]
            utterances = data.get("utterances", [])
            if utterances:
                lines = []
                for u in utterances:
                    spk = u.get("additions", {}).get("speaker", "?")
                    txt = u.get("text", "").strip()
                    if txt:
                        lines.append(f"[speaker_{spk}] {txt}")
                return "\n".join(lines)
            return data.get("text", "")
        elif code in (0, 1001, 1002, 2000):
            time.sleep(15)   # 每15秒轮询一次
        elif code == 1015:
            raise Exception(f"ASR download failed, URL may be expired: {result}")
        else:
            raise Exception(f"ASR error: code={code}, msg={result}")

    raise TimeoutError("ASR timeout after 10 minutes")

transcript = query_asr(task_id)
```

⚠️ **坑6：code 2000 不是错误码（最坑的坑）**
- 2000 = "音频已下载完成，正在转写中"
- 误判为错误而停止轮询是常见问题
- **正确做法：2000 时继续轮询，等 code=1000**

⚠️ **坑7：超时处理**
- 超过 10 分钟没完成 → 可能是 tmp_download_url 过期了
- 重新获取 tmp_download_url，重新 submit，再轮询
- 累计超过 30 分钟仍失败 → 判定 ASR 卡死，群里通知失败，将 bitable 状态改为"失败"

### 3.5 ASR 完整状态码参考

| code | 含义 | 处理 |
|------|------|------|
| 1000 + text有内容 | ✅ 完成 | 保存转写文本，进入阶段四 |
| 1000 + text为空 | ⚠️ 完成但空 | 音频太短或格式问题，报错 |
| 0 / 1001 / 1002 | 处理中 | 继续轮询 |
| **2000** | 正在转写（不是错误） | **继续轮询！** |
| 1015 | 下载失败 | URL 过期，重新获取 tmp_download_url |
| 其他 | 未知错误 | 记录日志，群里通知失败 |

### 3.6 保存转写文本

```python
# transcript 是 query_asr() 返回的字符串，用 Python 写入文件
with open("/tmp/meeting_transcript.txt", "w") as f:
    f.write(transcript)
```

路径统一用 `/tmp/meeting_transcript.txt`，与阶段四 Codex 调用和各 prompt skill 保持一致。

---

## 阶段四：生成纪要

### 意图路由

根据记录中的"纪要模板类型"选择对应 prompt skill，将 SKILL.md 内容写入 `/tmp/meeting_prompt.md`：

| 模板类型 | 读取 skill | 默认？ |
|---------|-----------|-------|
| 商务版纪要 | `meeting-business` | |
| 详细版纪要 | `meeting-detail` | |
| 行动项版纪要 | `meeting-action` | ✅ 默认 |

---

### 方案 A：直接用 openclaw 生成（推荐）

无需额外安装，直接读文件生成。**转写文本只传路径，不要内联展开**（大文本内联会超出 context 限制）：

```
读取 /tmp/meeting_prompt.md 获取分析框架，再读取 /tmp/meeting_transcript.txt 完成分析，输出完整 Markdown 格式的会议纪要。
```

---

### 方案 B：使用 Codex CLI（需额外安装，见 README 第四步）

#### 前置操作（每次必须）
```bash
pkill -f codex 2>/dev/null && sleep 1
unset http_proxy https_proxy HTTP_PROXY HTTPS_PROXY
export http_proxy=http://127.0.0.1:YOUR_PROXY_PORT    # 不需要代理则删除这两行
export https_proxy=http://127.0.0.1:YOUR_PROXY_PORT
CODEX=YOUR_CODEX_PATH
```

⚠️ **坑18：pkill 不能省**
- 之前的 codex session 没退出 → 后续所有 `codex exec` 都 hang，无任何错误输出
- 每次调用前必须执行，不能因为"上次应该已经退出了"就跳过

#### 调用

```bash
$CODEX exec --full-auto "$(cat /tmp/meeting_prompt.md)

---
转写文本见 /tmp/meeting_transcript.txt，请读取后完成分析，输出完整 Markdown 格式。"
```

⚠️ **坑17：不要尝试把转写文本也内联展开**
- ❌ `$(cat /tmp/meeting_prompt.md) $(cat /tmp/meeting_transcript.txt)` → context 爆炸
- ✅ 只展开 prompt，转写文本传路径让 Codex 自己 `cat` 读取

必须加 `pty=true`。

---

## 阶段五：结果回填

### 创建飞书文档

**短文档（< 5000字）**：直接用 `feishu_create_doc`
```
feishu_create_doc(title="会议纪要 - {名称} - {日期}", markdown=纪要内容)
```

**长文档（≥ 5000字）**：`feishu_create_doc` 的 markdown 模式对长文本不稳定，需要用 **lark-cli block API 分块写入**：

```python
python3 -c "
import json, subprocess

text = open('/tmp/meeting_transcript.txt').read()
chunk_size = 5500
chunks = [text[i:i+chunk_size] for i in range(0, len(text), chunk_size)]
DOC_ID = 'your_doc_id_here'   # 先用 feishu_create_doc 创建空文档拿到 DOC_ID

for chunk in chunks:
    body = {
        'children': [{
            'block_type': 2,
            'text': {
                'style': {},
                'elements': [{'text_run': {'content': chunk}}]
            }
        }]
    }
    subprocess.run([
        'lark-cli', 'api', 'POST',
        f'/open-apis/docx/v1/documents/{DOC_ID}/blocks/{DOC_ID}/children',
        '--params', json.dumps({'document_revision_id': -1}),
        '--as', 'user',
        '--data', json.dumps(body, ensure_ascii=False)
    ])
"
```

⚠️ **坑11：feishu_update_doc 追加大文本不稳定**
- 长文本用 `feishu_update_doc(mode="append", markdown=...)` 只写入一小段
- 原因：markdown 转富文本过程中长内容被截断
- 解决：block API（block_type=2 文本块）按 5500 字分块写入

⚠️ **坑13：feishu_doc_media 文件路径限制**
- 传 `/tmp/xxx` → "Local file access denied"
- 文件必须放在 `~/.openclaw/tmp/` 下

**会议纪要文档**（AI 输出，通常 3000-8000 字）：
- 如果 < 5000 字 → `feishu_create_doc` 直接创建
- 如果 ≥ 5000 字 → 创建空文档 + block API 分块写入

**转写文本文档**（通常 10000-50000 字）：
- 一律用 block API 分块写入
- 同时上传原始 txt 附件（用 feishu_doc_media，文件放 `~/.openclaw/tmp/`）

### 回填 bitable

**完整流程**：
1. 阶段三开始时：状态 → "处理中"
2. 阶段五创建完飞书文档后：拿到文档 URL，状态 → "已完成"，回填链接

**从 feishu_create_doc 获取文档 URL**：
```python
result = feishu_create_doc(title="...", markdown="...")
doc_url = result["data"]["document"]["url"]
```

**回填操作**：
```python
feishu_bitable_app_table_record.update(
  app_token="YOUR_BITABLE_APP_TOKEN",
  table_id="YOUR_TABLE_ID",
  record_id=record_id,   # 从阶段二拿到的记录 ID
  fields={
    "状态": "已完成",
    "转写文字链接": {"link": transcript_doc_url, "text": "转写文本"},
    "会议纪要链接": {"link": meeting_doc_url, "text": "纪要文档"}
  }
)
```

⚠️ **坑12：超链接字段创建时带 property 参数报错**
- 报错：`1254087 URLFieldPropertyError`
- 解决：省略 `property` 参数，只传 `{"link": "...", "text": "..."}`

⚠️ **坑19：超链接字段（type=15）的值格式**
- ✅ 正确：`{"link": "https://xxx", "text": "显示文本"}`
- ❌ 错误：直接传字符串 URL → 报错
- ❌ 错误：`{"url": "https://xxx"}` → 字段名是 `link` 不是 `url`，传错了静默失败

### 写 Obsidian（可选）
如果使用 Obsidian，将纪要写入：`YOUR_OBSIDIAN_PATH/YYYY-MM-DD-{名称}-{版本}.md`
不使用则跳过此步骤。

### 工作群通知完成
用 lark-cli 在群里发消息通知（不能用 message tool 回复，见坑16）。

---

## 错误处理

| 步骤 | 开始时 | 失败时 |
|------|--------|--------|
| 状态字段 | 改为"处理中" | 改为"失败"（不重置回空，保留可见性，下次自动重试） |
| 群里通知 | — | 通知失败原因 |
| 私聊通知 | — | 私聊通知用户 |

---

## 多维表格字段结构

| 字段名 | 类型 | 说明 |
|--------|------|------|
| 创建时间 | DateTime | 自动 |
| 会议名称 | Text | 用户填写 |
| 录音文件 | Attachment | 用户上传 |
| 纪要模板类型 | SingleSelect | 商务版纪要 / 详细版纪要 / 行动项版纪要 |
| 状态 | SingleSelect | 空 / 处理中 / 失败 / 已完成 |
| 转写文字链接 | Url | Bot 回填 |
| 会议纪要链接 | Url | Bot 回填 |
| 标签 | Text | 可选 |
