---
name: feishu-card-json-v2-skill
description: "飞书/Lark 消息卡片 JSON 2.0 构建技能。用于按官方 schema 2.0 规范产出可直接通过 IM/群机器人/Webhook/SDK 发送的飞书消息卡片 JSON,覆盖 schema、config、header、body、各类内容/容器/交互组件、行为(behaviors)、回传(callback)、流式更新(streaming_mode)、多语言(i18n)。**触发词**:飞书卡片、Lark 卡片、message card、interactive 卡片、卡片 JSON、card JSON 2.0、schema 2.0、card.action.trigger、卡片回调、流式卡片"
---

# 飞书消息卡片 JSON 2.0 构建技能

> 参考: [卡片 JSON 2.0 结构 - 飞书开放平台](https://open.feishu.cn/document/feishu-cards/card-json-v2-structure)。本技能用于按 **schema = "2.0"** 规范产出**可直接发送**的卡片 JSON,并给出对应的 OpenAPI 调用方式。所有示例均为完整可运行 JSON,复制即可用。

---

## 1. 何时使用本技能

### 1.1 必须使用
- 用户要求"做一个飞书/Lark 消息卡片"、"生成一段 card JSON"、"interactive 消息内容"
- 用户要发审批/告警/通知/AI 回复卡片,或要求按钮、表单、流式输出
- 用户要把卡片接到 webhook、群机器人、IM v1 send/patch 接口、Lark Node SDK
- 用户要从 1.0 卡片升级到 2.0
- 用户要做卡片回调(`card.action.trigger`)处理

### 1.2 不应使用
- 用户只是发普通文本消息(`msg_type: text`),不需要卡片
- 用户要的是文档/多维表格/电子表格的可视化,不是 IM 消息卡片

### 1.3 前置约束
1. **schema 必须显式声明 "2.0"**:不写默认按 1.0 解析,字段会全部错位。
2. **总元素数 ≤ 200**(单卡片所有 element/组件之和),超过会被截断。
3. **流式更新**(`config.streaming_mode = true`)只接受 `plain_text` 与 `markdown` 的全量内容覆盖。
4. **2.0 不再支持 1.0 的 `tag:"action"` 容器**,按钮直接放在 `body.elements` 或 `column/interactive_container/form` 内。
5. **回传交互**统一通过组件的 `behaviors[{type:"callback",value:{...}}]`,旧版 `value` 顶层字段不要再用。
6. **海外 Lark** 域名为 `open.larksuite.com`,字段结构与飞书一致,`schema` 同样为 `"2.0"`。

---

## 2. 顶层结构速查

```jsonc
{
  "schema": "2.0",                  // 必填,固定 "2.0"
  "config": { /* 全局行为/样式 */ },
  "card_link": { /* 整卡跳转 */ },
  "header": { /* 标题区 */ },
  "body": { "elements": [ /* 组件按数组顺序纵向排列 */ ] },
  "i18n_elements": { "zh_cn": [...], "en_us": [...] }   // 多语言时使用,优先级高于 body
}
```

### 2.1 顶层字段

| 字段 | 必填 | 说明 |
|------|------|------|
| `schema` | 是 | 固定 `"2.0"` |
| `config` | 否 | 全局开关:转发、流式、宽度模式、汇总、按钮组样式等 |
| `card_link` | 否 | 整张卡片可点击跳转 `{ url, pc_url, ios_url, android_url }` |
| `header` | 否 | 标题区组件 |
| `body` | 是* | 内容主体,内含 `elements[]` |
| `i18n_elements` | 否 | 多语言主体,与 `body` 二选一 |

### 2.2 config 常用字段

```jsonc
"config": {
  "streaming_mode": false,           // AI 流式输出场景设 true,只支持 plain_text/markdown 全量推送
  "summary": { "content": "卡片摘要" }, // 摘要,通知中心和右上角提示展示
  "enable_forward": true,            // 是否允许转发,默认 true
  "update_multi": true,              // 共享卡片:多人共享同一份状态;独享卡片设 false
  "width_mode": "compact",           // compact(默认)/fill,fill 宽屏占满
  "style": {
    "text_size": { "normal_v2": { "default": "normal", "pc": "normal", "mobile": "heading" } },
    "color": { "primary_v2": { "light_mode": "blue", "dark_mode": "blue" } }
  }
}
```

### 2.3 header 字段

```jsonc
"header": {
  "title":    { "tag": "plain_text", "content": "卡片主标题" },
  "subtitle": { "tag": "plain_text", "content": "卡片副标题" },
  "template": "blue",                 // 主题色: blue/wathet/turquoise/green/yellow/orange/red/carmine/violet/purple/indigo/grey/default
  "icon":   { "tag": "standard_icon", "token": "larkcommunity_colorful", "color": "blue" },
  "ud_icon":{ "tag": "standard_icon", "token": "chat-forbidden_outlined" },
  "text_tag_list": [
    { "tag": "text_tag", "text": { "tag": "plain_text", "content": "标签 1" }, "color": "blue" }
  ],
  "padding": "12px"
}
```

> `title.tag` 取 `plain_text`(纯文本)或 `lark_md`(支持 emoji/at/飞书表情等富文本)。

---

## 3. 组件目录(body.elements 内可放)

按官方分类整理,组件统一通过 `tag` 区分。容器组件可继续嵌套(表单和表格不能再嵌套到 column/interactive_container)。

### 3.1 内容组件 (Content)
| tag | 用途 | 关键字段 |
|-----|------|---------|
| `markdown` | 富文本 | `content` 支持飞书 markdown 扩展(at、person、number_tag、text_tag、link、color 等) |
| `div` | 文本块 | `text:{tag:"plain_text"|"lark_md", content}`,可挂 `fields[]` 双列展示 |
| `plain_text` | 纯文本(行内) | `tag/content` |
| `hr` | 分割线 | 无 |
| `img` | 图片 | `img_key`, `alt`, `mode` |
| `chart` | 图表 | `chart_spec`(VChart Spec) |
| `table` | 表格 | `columns[]` + `rows[]`,单列 `data_type:text|markdown|options|persons|date_time|number` |
| `note` | 备注 | `elements[]` 支持 `plain_text/lark_md/img` |
| `person` | 人员 | `user_id`,可与 markdown 内联 `<person>` 互换 |
| `person_list` | 人员列表 | `persons[]` |
| `column_set` | 分栏 | 见容器 |

### 3.2 容器组件 (Container)
| tag | 用途 | 关键字段 |
|-----|------|---------|
| `column_set` | 分栏 | `flex_mode: none/stretch/flow/bisect/trisect`、`columns[].tag:"column"` |
| `column` | 列 | `width: auto/weighted`、`weight`、`vertical_align`、`elements[]` |
| `form` | 表单容器 | `name`(回传 key)、内嵌交互组件统一提交 |
| `interactive_container` | 整体可点击的容器 | `behaviors[]` 可挂 open_url 或 callback |
| `collapsible_panel` | 折叠面板 | `header.title`、`expanded`、`elements[]` |
| `loop_container` | 循环容器 | 用 `_varloop` 渲染数组 |

### 3.3 交互组件 (Interactive)
| tag | 用途 |
|-----|------|
| `button` | 按钮(`type: default/primary/success/danger/laser/text`、`size: tiny/small/medium`、`width: default/fill/<px>`) |
| `input` | 输入框 |
| `select_static` | 单选下拉 |
| `multi_select_static` | 多选下拉 |
| `select_person` | 人员单选 |
| `multi_select_person` | 人员多选 |
| `picker_date` / `picker_time` / `picker_datetime` | 日期/时间选择 |
| `checker` | 勾选框(任务清单) |
| `overflow` | 折叠菜单 |

> 所有交互组件统一通过 `behaviors[]` 声明动作:`open_url` 跳链 / `callback` 回传 / `form_action`(form 内 submit/reset)。`element_id` 是接口操作组件的唯一键,建议显式给。

---

## 4. 标准范式

### 4.1 行为 behaviors(替代 1.0 的 multi_url + value)

```jsonc
"behaviors": [
  { "type": "open_url",
    "default_url": "https://example.com",
    "android_url": "https://developer.android.com/",
    "ios_url":     "lark://msgcard/unsupported_action",
    "pc_url":      "https://www.example.com" },
  { "type": "callback", "value": { "action": "approve", "ticket_id": "T123" } }
]
```

### 4.2 文本节点统一形态

```jsonc
{ "tag": "plain_text", "content": "...", "i18n": { "en_us": "..." } }
{ "tag": "lark_md",    "content": "**粗体**\n<at id=all></at>" }
{ "tag": "markdown",   "content": "# 标题\n[link](https://feishu.cn)" }
```

### 4.3 spacing/padding/margin 数值规范
- 关键字: `small`(4px)、`medium`(8px)、`large`(12px)、`extra_large`(16px)
- 自定义: `"8px"` 或 `"4px 8px 4px 8px"`(上右下左),范围 `[0,99]px`,`margin` 支持 `[-99,99]px`

### 4.4 颜色枚举
`blue / wathet / turquoise / green / yellow / orange / red / carmine / violet / purple / indigo / grey / default`,部分组件还支持 `light_mode/dark_mode` 双模式。

### 4.5 元素上限与可读性
- 单卡总元素 ≤ **200**
- `text_size` 推荐: `heading-0..heading-4 / heading / normal(14px) / notation(12px)`
- 移动端窄屏要靠 `column_set.flex_mode = stretch/flow/bisect/trisect` 兜底

---

## 5. 完整可用示例

### 5.1 最小可发送卡片(Hello)

```json
{
  "schema": "2.0",
  "config": { "update_multi": true },
  "header": {
    "title":    { "tag": "plain_text", "content": "Hello Feishu Card 2.0" },
    "template": "blue"
  },
  "body": {
    "elements": [
      { "tag": "markdown", "content": "**这是一张 schema=2.0 的卡片**\n点击下方按钮试试 →" },
      {
        "tag": "button", "element_id": "btn_open",
        "type": "primary", "size": "medium",
        "text": { "tag": "plain_text", "content": "打开飞书开放平台" },
        "behaviors": [
          { "type": "open_url", "default_url": "https://open.feishu.cn" }
        ]
      }
    ]
  }
}
```

### 5.2 审批通知卡片(双列 + 操作按钮 + 回调)

```json
{
  "schema": "2.0",
  "config": { "update_multi": true, "summary": { "content": "❗️待审批: 发布上线" } },
  "header": {
    "title":    { "tag": "plain_text", "content": "❗️你有一份待审批文件" },
    "subtitle": { "tag": "plain_text", "content": "工单号 #T-20260514-001" },
    "template": "orange",
    "text_tag_list": [
      { "tag": "text_tag", "text": { "tag": "plain_text", "content": "发布审核" }, "color": "orange" },
      { "tag": "text_tag", "text": { "tag": "plain_text", "content": "高优" }, "color": "red" }
    ]
  },
  "body": {
    "elements": [
      {
        "tag": "column_set", "flex_mode": "bisect", "background_style": "grey",
        "columns": [
          { "tag": "column", "width": "weighted", "weight": 1,
            "elements": [{ "tag": "markdown", "content": "**👤 提交人**\n<at email=test@example.com></at>" }] },
          { "tag": "column", "width": "weighted", "weight": 1,
            "elements": [{ "tag": "markdown", "content": "**📅 提交时间**\n2026-05-14 14:32" }] }
        ]
      },
      { "tag": "markdown", "content": "**📚 留言**\n工作需要,请尽快通过,谢谢!" },
      { "tag": "hr" },
      {
        "tag": "column_set", "flex_mode": "none", "horizontal_spacing": "medium",
        "columns": [
          { "tag": "column", "width": "weighted", "weight": 1, "elements": [
            { "tag": "button", "element_id": "btn_approve",
              "type": "primary", "width": "fill",
              "text": { "tag": "plain_text", "content": "同意" },
              "behaviors": [{ "type": "callback",
                "value": { "action": "approve", "ticket_id": "T-20260514-001" } }],
              "confirm": {
                "title": { "tag": "plain_text", "content": "确认通过?" },
                "text":  { "tag": "plain_text", "content": "审批通过后将立即生效" }
              }
            }
          ]},
          { "tag": "column", "width": "weighted", "weight": 1, "elements": [
            { "tag": "button", "element_id": "btn_reject",
              "type": "danger", "width": "fill",
              "text": { "tag": "plain_text", "content": "拒绝" },
              "behaviors": [{ "type": "callback",
                "value": { "action": "reject", "ticket_id": "T-20260514-001" } }]
            }
          ]}
        ]
      }
    ]
  }
}
```

### 5.3 表单容器(form):一次性提交多字段

```json
{
  "schema": "2.0",
  "header": { "title": { "tag": "plain_text", "content": "新建工单" }, "template": "indigo" },
  "body": {
    "elements": [
      {
        "tag": "form", "name": "ticket_form",
        "elements": [
          { "tag": "input", "element_id": "title",
            "label": { "tag": "plain_text", "content": "标题" },
            "placeholder": { "tag": "plain_text", "content": "请输入工单标题" },
            "required": true, "name": "title" },
          { "tag": "select_static", "element_id": "priority", "name": "priority",
            "label": { "tag": "plain_text", "content": "优先级" },
            "placeholder": { "tag": "plain_text", "content": "请选择" },
            "options": [
              { "value": "p0", "text": { "tag": "plain_text", "content": "P0 紧急" } },
              { "value": "p1", "text": { "tag": "plain_text", "content": "P1 高" } },
              { "value": "p2", "text": { "tag": "plain_text", "content": "P2 中" } }
            ] },
          { "tag": "picker_datetime", "element_id": "due", "name": "due",
            "label": { "tag": "plain_text", "content": "截止时间" } },
          { "tag": "column_set", "flex_mode": "none", "horizontal_spacing": "medium", "columns": [
            { "tag": "column", "width": "weighted", "weight": 1, "elements": [
              { "tag": "button", "element_id": "submit", "type": "primary", "width": "fill",
                "text": { "tag": "plain_text", "content": "提交" },
                "behaviors": [{ "type": "form_action", "behavior": "submit" }] }
            ]},
            { "tag": "column", "width": "weighted", "weight": 1, "elements": [
              { "tag": "button", "element_id": "reset", "type": "default", "width": "fill",
                "text": { "tag": "plain_text", "content": "重置" },
                "behaviors": [{ "type": "form_action", "behavior": "reset" }] }
            ]}
          ]}
        ]
      }
    ]
  }
}
```

> 提交后回调里 `event.action.form_value = { "title": "...", "priority": "p1", "due": "..." }`,以 `name` 为 key。

### 5.4 折叠面板 + 表格

```json
{
  "schema": "2.0",
  "header": { "title": { "tag": "plain_text", "content": "本周指标周报" }, "template": "green" },
  "body": {
    "elements": [
      { "tag": "markdown", "content": "**核心结论**: 总单量同比 +18%, 平均时长下降 12%。" },
      {
        "tag": "collapsible_panel", "expanded": false, "background_color": "grey",
        "header": {
          "title": { "tag": "markdown", "content": "**详细数据(点击展开)**" },
          "padding": "4px 0px 4px 8px",
          "icon": { "tag": "standard_icon", "token": "down_outlined" }
        },
        "elements": [
          {
            "tag": "table",
            "columns": [
              { "name": "team",   "display_name": "团队", "data_type": "text",   "width": "auto" },
              { "name": "count",  "display_name": "单量", "data_type": "number", "width": "auto" },
              { "name": "owner",  "display_name": "负责人", "data_type": "persons", "width": "auto" }
            ],
            "rows": [
              { "team": "增长",   "count": 128, "owner": "ou_xxxxx1" },
              { "team": "基础",   "count":  76, "owner": "ou_xxxxx2" },
              { "team": "国际化", "count":  54, "owner": ["ou_xxxxx3","ou_xxxxx4"] }
            ]
          }
        ]
      }
    ]
  }
}
```

### 5.5 AI 流式回复卡片(streaming_mode)

```json
{
  "schema": "2.0",
  "config": { "streaming_mode": true, "summary": { "content": "AI 正在回答..." } },
  "header": { "title": { "tag": "plain_text", "content": "🤖 AI 助手" }, "template": "violet" },
  "body": {
    "elements": [
      {
        "tag": "markdown", "element_id": "ai_answer",
        "content": "正在思考中...",
        "text_size": "normal"
      }
    ]
  }
}
```

> 流式更新流程:
> 1. 先以上面 JSON 调 `im.v1.message.create` 创建卡片。
> 2. 后续用 `cardkit.v1.cardElement.content` 或 `im.v1.message.patch` **全量覆盖** `element_id=ai_answer` 的 `content`,模拟"打字机"。
> 3. 流式结束后,可继续追加按钮/下拉等组件,做反馈采集。

### 5.6 多语言 i18n_elements

```json
{
  "schema": "2.0",
  "header": {
    "title": { "tag": "plain_text", "content": "Welcome", "i18n": { "zh_cn": "欢迎", "ja_jp": "ようこそ" } },
    "template": "wathet"
  },
  "i18n_elements": {
    "zh_cn": [{ "tag": "markdown", "content": "**你好,世界**" }],
    "en_us": [{ "tag": "markdown", "content": "**Hello, world**" }],
    "ja_jp": [{ "tag": "markdown", "content": "**こんにちは、世界**" }]
  }
}
```

---

## 6. 把卡片发出去

> 所有调用都要把卡片 JSON **字符串化** 后塞进 `content`(IM v1)或 `card`(CardKit/某些 webhook)。

### 6.1 IM v1: 发送卡片消息

```bash
TOKEN="$TENANT_ACCESS_TOKEN"
CARD=$(cat card.json)             # 你保存的卡片 JSON
CONTENT=$(jq -nc --argjson c "$CARD" '$c | tostring')

curl -sS -X POST "https://open.feishu.cn/open-apis/im/v1/messages?receive_id_type=open_id" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json; charset=utf-8" \
  -d "{
    \"receive_id\": \"ou_xxxxxxxxxxxxx\",
    \"msg_type\":   \"interactive\",
    \"content\":    $CONTENT
  }"
```

### 6.2 群机器人 Webhook(自定义机器人)

```bash
curl -sS -X POST "$LARK_BOT_WEBHOOK" \
  -H "Content-Type: application/json" \
  -d "{ \"msg_type\": \"interactive\", \"card\": $(cat card.json) }"
```

### 6.3 Node SDK(@larksuiteoapi/node-sdk)

```javascript
const lark = require('@larksuiteoapi/node-sdk');
const fs   = require('fs');

const client = new lark.Client({ appId: process.env.APP_ID, appSecret: process.env.APP_SECRET });
const card   = JSON.parse(fs.readFileSync('card.json', 'utf-8'));

await client.im.message.create({
  params: { receive_id_type: 'open_id' },
  data: {
    receive_id: 'ou_xxx',
    msg_type:   'interactive',
    content:    JSON.stringify(card),    // 必须是字符串
  },
});
```

### 6.4 更新已发送卡片(全量覆盖)

```bash
curl -sS -X PATCH "https://open.feishu.cn/open-apis/im/v1/messages/$MESSAGE_ID" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json; charset=utf-8" \
  -d "{ \"content\": $(jq -Rs . < card.json) }"
```

---

## 7. 处理卡片回调(card.action.trigger)

订阅事件 `card.action.trigger`,SDK 注册示例:

```javascript
const dispatcher = new lark.EventDispatcher({}).register({
  'card.action.trigger': async (data) => {
    // data.event.action 结构示例:
    // { tag: 'button', value: { action: 'approve', ticket_id: 'T-001' }, form_value: {...} }
    const action = data.event.action.value?.action;
    const tid    = data.event.action.value?.ticket_id;

    if (action === 'approve') {
      // 1. 业务处理...
      // 2. 通过 toast 给用户即时反馈
      return { toast: { type: 'success', content: `已通过 ${tid}` } };
    }
    return { toast: { type: 'info', content: '操作收到' } };
  },
});
```

回调可返回的字段:
- `toast`: `{ type: 'info|success|warning|error', content }` 即时弹窗
- `card`: 直接整卡替换为新的卡片 JSON
- `delete_message`: `true` 撤回原卡片

---

## 8. 1.0 → 2.0 迁移清单

| 1.0 现象 | 2.0 做法 |
|---------|----------|
| 顶层直接放 `elements: [...]` | 改为 `body: { elements: [...] }`,并加 `schema: "2.0"` |
| `tag: "action"` 包裹按钮 | 直接把按钮放在 `body.elements` / `column.elements` / `form.elements` |
| 按钮用顶层 `value: {...}` 回传 | 改为 `behaviors: [{ type: "callback", value: {...} }]` |
| 按钮用 `multi_url` 跳转 | 改为 `behaviors: [{ type: "open_url", default_url, pc_url, ios_url, android_url }]` |
| markdown `href.urlVal` 差异化跳转 | 不再支持,改用 `interactive_container` + `behaviors.open_url` |
| `header.padding` 默认值 | 2.0 默认变更:有边框/背景时上下 4px、左右 8px;无边框时 0 |

---

## 9. 排障 Checklist

| 现象 | 排查 |
|------|------|
| `230099 Failed to create card content` | JSON 不合法 / `schema` 没写成 `"2.0"` / 字段名拼错 |
| 按钮不响应回调 | 应用未订阅 `card.action.trigger`、`encrypt_key`/`verification_token` 没配 |
| 流式更新文字不刷新 | `config.streaming_mode` 未设 `true`,或更新时不是全量覆盖 `markdown.content` |
| 海外租户报权限错 | 域名要换为 `https://open.larksuite.com`,SDK `domain: lark.Domain.Lark` |
| at 不到人 | `<at id=open_id></at>` 必须用 open_id;`lark_md` 才能解析 at 语法 |
| 元素超过 200 个被截断 | 拆成多张卡 / 折叠面板收纳 / 用 table 替代多条 markdown |
| 表格里的人员显示"未知用户" | 应用未开通查询用户权限,或传入 ID 非本租户用户 |

---

## 10. 输出建议(给执行者)

1. 默认产出 **schema 2.0** 卡片;1.0 仅在用户明确要求时产出。
2. 卡片 JSON 单独成文件(`card.json`),不要嵌在调用脚本里,便于复用。
3. 回复用户时,**先展示卡片 JSON**,再附上对应的 `curl` / Node SDK 调用片段;不要把这两件事混进同一个代码块。
4. 发送时,`content` 字段务必是 **JSON 字符串**(`JSON.stringify(card)` / `jq -Rs`),不要直接传对象,否则会报 `230099`。
5. 在脚本里别硬编码 `tenant_access_token`、`webhook` URL,统一走环境变量。