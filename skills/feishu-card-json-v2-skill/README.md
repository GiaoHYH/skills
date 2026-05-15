# feishu-card-json-v2-skill

飞书消息卡片 JSON 2.0 构建技能。

用于按官方 schema 2.0 规范产出可直接通过 IM/群机器人/Webhook/SDK 发送的飞书消息卡片 JSON。

## 能力概览

- 覆盖 schema、config、header、body 顶层结构
- 支持各类内容/容器/交互组件的组装
- 支持配置行为（behaviors）、回传（callback）
- 支持流式更新（streaming_mode）和多语言（i18n）
- 提供 1.0 到 2.0 的迁移清单与卡片发送示例

## 适用场景

- 要求生成或制作飞书/Lark 消息卡片 JSON
- 制作交互式（interactive）消息内容，如审批、告警、通知、AI 回复卡片
- 把卡片对接到 webhook、群机器人、IM API 或 Node SDK
- 处理卡片回调（`card.action.trigger`）
- 从 1.0 卡片升级至 2.0 规范

## 不建议使用的场景

- 只是发送普通文本消息（`msg_type: text`），不需要卡片 UI
- 需要的是文档/多维表格的可视化，而非 IM 消息卡片

## 前置要求

- 理解卡片 schema 必须显式声明 `"2.0"`
- 注意单卡片总元素数不能超过 200

## 文件说明

- `SKILL.md`: 技能主定义文件，包含完整的技能规范与示例 JSON
- `README.md`: 本技能的展示说明文件
