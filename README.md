# skills

这个仓库用于存放多个可复用的技能定义，每个技能目录下通常包含：

- `SKILL.md`：技能主定义文件，描述触发条件、能力边界、执行约束和示例。
- `README.md`：技能的简要说明文件，便于快速浏览和仓库展示。

## 目录结构

当前 `skills/` 目录下包含以下技能：

- `apaas-oapi-skill`：aPaaS OpenAPI 操作技能，覆盖对象、记录、附件、自动化流程和云函数等常见能力。
- `correction-memory-skill`：纠错经验沉淀技能，用于记录、检索和复用历史修正规则。
- `feishu-card-json-v2-skill`：飞书消息卡片 JSON 2.0 构建技能，用于生成可直接发送的卡片内容。
- `script-readme-generator`：脚本 README 自动生成技能，用于在生成脚本时同步补充说明文档。

## 使用说明

1. 进入目标技能目录。
2. 阅读对应的 `SKILL.md` 了解该技能的使用时机和执行规范。
3. 通过 `README.md` 快速查看技能概览。

## Git 说明

仓库根目录的 `.gitignore` 已配置忽略 `.idea/`，避免 IDE 配置文件被提交到 Git。
