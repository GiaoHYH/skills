# apaas-oapi-skill

aPaaS OpenAPI 操作技能包。基于 `apaas-oapi-client` Node.js SDK 封装，提供 aPaaS 平台完整的 OpenAPI 调用能力。

## 能力概览

- 提供数据对象（Schema）、记录（Record）、附件、全局选项/环境变量、页面、部门、自动化流程、云函数等模块的完整 CRUD 操作。
- 强调规范化操作：写记录前先获取字段定义，批量操作强制使用分页 Iterator，避免限流问题。
- 内置代码片段：对象创建、批量导入、自动化流程触发、云函数调用等。

## 适用场景

- 需要操作 aPaaS / apaas 平台数据
- 管理 aPaaS 数据对象：创建/更新/删除对象，添加/修改/删除字段
- 操作 aPaaS 记录：单条/批量 增删改查
- 执行 aPaaS 自动化流程或调用云函数
- 操作 aPaaS 附件、页面、部门、全局选项等

## 不建议使用的场景

- 操作飞书多维表格（Base/bitable），应使用专门的多维表格技能
- 泛化谈论"数据库"或"低代码平台"，未明确指明 aPaaS 时

## 前置要求

- 准备好 `clientId`、`clientSecret`、`namespace` 凭证
- 敏感凭证必须通过环境变量传入，严禁硬编码

## 文件说明

- `SKILL.md`: 技能主定义文件，包含完整的技能说明与参考脚本代码
- `README.md`: 本技能的展示说明文件
