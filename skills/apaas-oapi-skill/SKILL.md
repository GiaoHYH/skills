---
name: apaas-oapi-skill
description: "aPaaS OpenAPI 操作技能包。基于 apaas-oapi-client Node.js SDK，封装 aPaaS 平台 RESTful API 调用能力。提供数据对象（Schema）、记录（Record）、附件、全局选项/环境变量、页面、部门、自动化流程、云函数等模块的完整 CRUD 操作。**触发词**：aPaaS、apaas、apaas-oapi、数据对象、object_、自动化流程、云函数、全局选项、环境变量"
---

# aPaaS OpenAPI Skill (apaas-oapi-skill)

> 基于 [apaas-oapi-client](https://www.npmjs.com/package/apaas-oapi-client) Node.js SDK 封装，提供 aPaaS 平台完整的 OpenAPI 调用能力。

---

## 1. 何时使用本 Skill

### 1.1 触发条件

**应使用本 skill**：
- 用户明确要操作 aPaaS / apaas 平台数据
- 用户要管理 aPaaS 数据对象（object）：创建/更新/删除对象，添加/修改/删除字段
- 用户要操作 aPaaS 记录：单条/批量 增删改查（CRUD）
- 用户要执行 aPaaS 自动化流程（automation flow）或调用云函数（function）
- 用户要操作 aPaaS 附件、页面、部门、全局选项、环境变量
- 用户提供了 `clientId` / `clientSecret` / `namespace` 等 aPaaS 凭证

**不应使用本 skill**：
- 用户操作的是飞书多维表格（Base/bitable），应使用 `lark-base-skill`
- 用户只是泛化谈论"数据库"或"低代码平台",未明确 aPaaS

### 1.2 前置约束

1. **凭证三件套必须齐全**：执行任何调用前必须确认 `clientId`、`clientSecret`、`namespace` 已就绪
2. **写操作前先拿结构**：写/更新记录前必须先 `client.object.metadata.fields` 获取对象字段定义
3. **API 名称精确匹配**：`object_name`、`api_name` 必须与 aPaaS 平台一致，禁止凭自然语言猜测
4. **批量操作单页上限 100**：超过 100 条必须使用 `*WithIterator` 系列方法，由 SDK 自动分页
5. **敏感凭证不得写入仓库**：`clientSecret` 仅通过环境变量传入，禁止硬编码到脚本中

---

## 2. 用户意图路由

| 用户意图 | 模块 | 方法 | 备注 |
|---------|------|------|------|
| "初始化 aPaaS 客户端" / "登录 aPaaS" | Auth | `new apaas.Client + client.init()` | 必须最先调用 |
| "查看所有数据对象" / "列出 object" | Object | `client.object.list` / `listWithIterator` | |
| "查看对象字段" / "获取字段元数据" | Object Metadata | `client.object.metadata.fields` | 写记录前必做 |
| "查看单个字段元数据" | Object Metadata | `client.object.metadata.field` | |
| "导出对象文档" / "对象转 Markdown" | Object Metadata | `client.object.metadata.export2markdown` | |
| "创建数据对象" / "新建 object" | Schema | `client.schema.create` | 批量创建对象 |
| "添加字段" / "修改对象" / "删除字段" | Schema | `client.schema.update` | operator: add/update/delete |
| "删除数据对象" | Schema | `client.schema.delete` | 批量删除 |
| "查询单条记录" | Record Search | `client.object.search.record` | 按 record_id |
| "查询多条记录" / "批量查询" | Record Search | `client.object.search.records` | ≤100/页 |
| "查询所有记录" / "分页迭代" | Record Search | `client.object.search.recordsWithIterator` | 自动翻页 |
| "创建单条记录" | Record Create | `client.object.create.record` | |
| "批量创建" / "导入数据" | Record Create | `client.object.create.records` (≤100) / `recordsWithIterator` | |
| "更新单条记录" | Record Update | `client.object.update.record` | |
| "批量更新" | Record Update | `client.object.update.records` (≤100) / `recordsWithIterator` | |
| "删除单条记录" | Record Delete | `client.object.delete.record` | |
| "批量删除" | Record Delete | `client.object.delete.records` (≤100) / `recordsWithIterator` | |
| "上传文件" / "上传附件" | Attachment | `client.attachment.file.upload` | |
| "下载文件" | Attachment | `client.attachment.file.download` | 返回二进制流 |
| "删除文件" | Attachment | `client.attachment.file.delete` | |
| "上传/下载头像" | Attachment | `client.attachment.avatar.upload/download` | |
| "查询全局选项" | Global | `client.global.options.list/detail/listWithIterator` | |
| "查询环境变量" | Global | `client.global.variables.list/detail/listWithIterator` | |
| "获取页面列表 / 详情 / URL" | Page | `client.page.list/detail/url/listWithIterator` | |
| "部门 ID 交换" | Department | `client.department.exchange/batchExchange` | 单/批量 |
| "执行自动化流程 V1" | Automation | `client.automation.v1.execute` | |
| "执行自动化流程 V2" | Automation | `client.automation.v2.execute` | 支持重新提交 |
| "调用云函数" | Function | `client.function.invoke` | |

---

## 3. 标准工作流

### 3.1 初始化（每次会话第一步）

```javascript
const { apaas } = require('apaas-oapi-client');

const client = new apaas.Client({
  clientId: process.env.APAAS_CLIENT_ID,
  clientSecret: process.env.APAAS_CLIENT_SECRET,
  namespace: process.env.APAAS_NAMESPACE,
  // disableTokenCache: false   // 可选，默认 false
});

await client.init();
client.setLoggerLevel(3);   // 0=fatal, 1=error, 2=warn, 3=info(默认), 4=debug, 5=trace
```

### 3.2 写记录前的强制流程

1. 调 `client.object.metadata.fields({ object_name })` 获取字段列表
2. 校验：`required` 字段全部覆盖、字段类型与传入值匹配、`unique` 字段无冲突
3. 再发起 create/update

### 3.3 批量数据处理（>100 条）

强制使用 `*WithIterator` 方法，SDK 内置 Bottleneck 限流，自动按 100/批拆分：
- `client.object.create.recordsWithIterator`
- `client.object.update.recordsWithIterator`
- `client.object.delete.recordsWithIterator`
- `client.object.search.recordsWithIterator`

返回结构 `{ total, items }`,所有子请求结果聚合。

---

## 4. 字段类型速查

aPaaS 支持 20+ 字段类型，常用类型：

| 类型名 | 用途 | settings 关键字段 |
|--------|------|-------------------|
| `text` | 单行文本 | `required`, `unique`, `max_length` |
| `multilineText` | 多行文本 | `required`, `max_length` |
| `number` | 数字 | `required`, `decimal_places` |
| `boolean` | 布尔值 | `required` |
| `dateTime` | 日期时间 | `required`, `display_format` |
| `option` | 单选 | `options[]`, `required` |
| `multiOption` | 多选 | `options[]`, `required` |
| `lookup` | 关联单值 | `reference_object_api_name` |
| `lookup_multi` | 关联多值 | `reference_object_api_name` |
| `referenceField` | 引用字段 | `reference_object_api_name`, `reference_field_api_name` |
| `rollup` | 汇总 | `rollup_type`(count/sum/avg/min/max), `target_field` |
| `formula` | 公式 | `expression`, `return_type` |
| `attachment` | 附件 | `required`, `multiple` |
| `image` | 图片 | `required`, `multiple` |
| `phone` | 电话 | `required` |
| `email` | 邮箱 | `required` |

> 完整规则参考 SDK 内 `FIELD_SCHEMA_RULES.md`,可通过 `npm root -g` 找到 `apaas-oapi-client/src/FIELD_SCHEMA_RULES.md`。

---

## 5. 常用代码片段

### 5.1 创建数据对象 + 字段

```javascript
await client.schema.create({
  objects: [{
    api_name: 'product',
    label: { zh_cn: '产品', en_us: 'Product' },
    settings: {
      display_name: 'code',
      allow_search_fields: ['_id', 'code']
    },
    fields: [{
      api_name: 'code',
      label: { zh_cn: '编号', en_us: 'Code' },
      type: { name: 'text', settings: { required: true } },
      encrypt_type: 'none'
    }]
  }]
});
```

### 5.2 给已存在对象增删字段

```javascript
await client.schema.update({
  objects: [{
    api_name: 'product',
    fields: [
      { operator: 'add',    api_name: 'name',  label: {...}, type: {...}, encrypt_type: 'none' },
      { operator: 'update', api_name: 'code',  label: {...}, type: {...} },
      { operator: 'delete', api_name: 'old_field' }
    ]
  }]
});
```

### 5.3 分页查询所有记录

```javascript
const { total, items } = await client.object.search.recordsWithIterator({
  object_name: 'object_store',
  data: {
    need_total_count: true,
    page_size: 100,
    offset: 0,
    filter: { /* 自定义筛选 */ },
    select: ['_id', 'name', 'code']
  }
});
```

### 5.4 批量导入超大数据集

```javascript
const records = [/* 任意数量,可超过 100 */];
const { total, items } = await client.object.create.recordsWithIterator({
  object_name: 'object_event_log',
  records
});
```

### 5.5 调用云函数 / 触发流程

```javascript
// 云函数
const fnRes = await client.function.invoke({
  name: 'StoreMemberUpdate',
  params: { storeId: 100 }
});

// 自动化流程 V2(支持重新提交)
const flowRes = await client.automation.v2.execute({
  flow_api_name: 'automation_a9ec6ee5fb1',
  operator: { _id: 100, email: 'sample@feishu.cn' },
  params: { storeId: 100 },
  is_resubmit: true,
  pre_instance_id: '1835957428957195'
});
```

### 5.6 导出对象元数据为 Markdown

```javascript
const fs = require('fs');

// 导出所有对象
const md = await client.object.metadata.export2markdown();
fs.writeFileSync('all_objects.md', md, 'utf-8');

// 仅导出自定义对象
const all = await client.object.listWithIterator();
const customNames = all.items.filter(o => !o.apiName.startsWith('_')).map(o => o.apiName);
const md2 = await client.object.metadata.export2markdown({ object_names: customNames });
fs.writeFileSync('custom_objects.md', md2, 'utf-8');
```

---

## 6. 执行约定（给执行体的硬性要求）

1. **所有 SDK 调用包裹 try/catch**,失败时打印 `err.response?.data` 并退出非零码
2. **凭证仅来自环境变量** `APAAS_CLIENT_ID` / `APAAS_CLIENT_SECRET` / `APAAS_NAMESPACE`,不得写死
3. **执行前安装依赖**：若 `node_modules/apaas-oapi-client` 不存在,先 `npm install apaas-oapi-client --save`
4. **脚本默认使用 CommonJS** (`require`),除非用户明确要求 ESM
5. **大批量任务必须**：先 `count` 总量 → 提示用户预计耗时 → 使用 `*WithIterator` → 打印进度
6. **删除操作**：必须先二次确认目标 `object_name` 与匹配条件,严禁无 filter 全表删

---

## 7. 调试与排障

| 现象 | 可能原因 | 排查动作 |
|------|---------|---------|
| `init()` 抛 401/403 | clientId/Secret 错误或权限不足 | 核对应用授权范围,确认 namespace 正确 |
| token 频繁过期 | `disableTokenCache: true` 或时钟漂移 | 关闭禁缓存; 检查系统时间 |
| 写记录报字段不存在 | api_name 与平台不一致 | 调 `metadata.fields` 比对 |
| 写记录报字段类型错误 | 入参类型与字段定义不符 | 参考 `FIELD_SCHEMA_RULES.md` |
| 批量请求被限流 / 排队慢 | Bottleneck 限流生效 | 这是预期行为,等待即可 |
| 大数据量导入超时 | 单批失败导致重试 | 改用 `recordsWithIterator`,SDK 已分批 |

通过 `client.setLoggerLevel(4)` 或 `5` 可输出详细 debug / trace 日志。

---

## 8. 参考脚本（已内嵌）

以下 JavaScript 参考脚本已全部集成在本文档中。需要执行时，将对应代码保存为本地 `.js` 文件后运行。

### 8.1 执行方式

```bash
export APAAS_CLIENT_ID=xxx
export APAAS_CLIENT_SECRET=xxx
export APAAS_NAMESPACE=app_xxx
npm install apaas-oapi-client --save
node init-and-list.js
```

### 8.2 `init-and-list.js`

```javascript
/**
 * 初始化 aPaaS Client 并列出所有数据对象
 * 用法: node scripts/init-and-list.js
 */
const { apaas } = require('apaas-oapi-client');

async function main() {
  const client = new apaas.Client({
    clientId: process.env.APAAS_CLIENT_ID,
    clientSecret: process.env.APAAS_CLIENT_SECRET,
    namespace: process.env.APAAS_NAMESPACE,
  });

  await client.init();
  client.setLoggerLevel(3);

  console.log('Token:', client.token);
  console.log('Token Expire (s):', client.tokenExpireTime);
  console.log('Namespace:', client.currentNamespace);

  const { total, items } = await client.object.listWithIterator({ limit: 100 });
  console.log(`\nTotal objects: ${total}`);
  for (const obj of items) {
    console.log(`  - ${obj.apiName}\t${obj.label?.zh_cn || ''}`);
  }
}

main().catch(err => {
  console.error('Error:', err.response?.data || err.message);
  process.exit(1);
});
```
### 8.3 `export-schema.js`

```javascript
/**
 * 导出 aPaaS 数据对象 Schema 为 Markdown 文档
 * 用法:
 *   node scripts/export-schema.js                       # 导出所有对象
 *   node scripts/export-schema.js obj1 obj2             # 仅导出指定对象
 *   node scripts/export-schema.js --custom              # 仅导出自定义对象(排除以 _ 开头)
 */
const fs = require('fs');
const path = require('path');
const { apaas } = require('apaas-oapi-client');

async function main() {
  const args = process.argv.slice(2);
  const onlyCustom = args.includes('--custom');
  const objectNames = args.filter(a => !a.startsWith('--'));

  const client = new apaas.Client({
    clientId: process.env.APAAS_CLIENT_ID,
    clientSecret: process.env.APAAS_CLIENT_SECRET,
    namespace: process.env.APAAS_NAMESPACE,
  });
  await client.init();

  let targetNames = objectNames;
  if (onlyCustom) {
    const all = await client.object.listWithIterator({ limit: 100 });
    targetNames = all.items.filter(o => !o.apiName.startsWith('_')).map(o => o.apiName);
    console.log(`Custom objects: ${targetNames.length}`);
  }

  const opts = targetNames.length ? { object_names: targetNames } : undefined;
  const md = await client.object.metadata.export2markdown(opts);

  const outFile = path.resolve(process.cwd(), 'apaas_schema.md');
  fs.writeFileSync(outFile, md, 'utf-8');
  console.log(`Schema exported -> ${outFile}`);
}

main().catch(err => {
  console.error('Error:', err.response?.data || err.message);
  process.exit(1);
});
```
### 8.4 `bulk-import.js`

```javascript
/**
 * 批量导入记录到 aPaaS 对象（支持 >100 条，自动分批）
 * 用法: node scripts/bulk-import.js <object_name> <data.json>
 *   data.json 必须是 record 数组 [{...}, {...}, ...]
 */
const fs = require('fs');
const { apaas } = require('apaas-oapi-client');

async function main() {
  const [objectName, dataPath] = process.argv.slice(2);
  if (!objectName || !dataPath) {
    console.error('Usage: node scripts/bulk-import.js <object_name> <data.json>');
    process.exit(1);
  }
  const records = JSON.parse(fs.readFileSync(dataPath, 'utf-8'));
  if (!Array.isArray(records)) {
    throw new Error('data.json must be an array of records');
  }

  const client = new apaas.Client({
    clientId: process.env.APAAS_CLIENT_ID,
    clientSecret: process.env.APAAS_CLIENT_SECRET,
    namespace: process.env.APAAS_NAMESPACE,
  });
  await client.init();
  client.setLoggerLevel(3);

  // 写前校验：先拉字段元数据
  const meta = await client.object.metadata.fields({ object_name: objectName });
  console.log(`Target object: ${objectName}`);
  console.log(`Field count: ${meta?.data?.items?.length || meta?.data?.length || 'unknown'}`);
  console.log(`Records to import: ${records.length}`);

  const startedAt = Date.now();
  const { total, items } = await client.object.create.recordsWithIterator({
    object_name: objectName,
    records,
  });
  const cost = ((Date.now() - startedAt) / 1000).toFixed(2);
  console.log(`Imported ${total} records in ${cost}s`);
  console.log(`Sub-request results: ${items.length}`);
}

main().catch(err => {
  console.error('Error:', err.response?.data || err.message);
  process.exit(1);
});
```
### 8.5 `run-automation.js`

```javascript
/**
 * 触发 aPaaS 自动化流程
 * 用法:
 *   node scripts/run-automation.js <flow_api_name> <params.json> [--v1]
 *   默认使用 V2,加 --v1 走 V1 接口
 */
const fs = require('fs');
const { apaas } = require('apaas-oapi-client');

async function main() {
  const argv = process.argv.slice(2);
  const useV1 = argv.includes('--v1');
  const [flowApiName, paramsPath] = argv.filter(a => !a.startsWith('--'));
  if (!flowApiName) {
    console.error('Usage: node scripts/run-automation.js <flow_api_name> [params.json] [--v1]');
    process.exit(1);
  }
  const params = paramsPath ? JSON.parse(fs.readFileSync(paramsPath, 'utf-8')) : {};

  const client = new apaas.Client({
    clientId: process.env.APAAS_CLIENT_ID,
    clientSecret: process.env.APAAS_CLIENT_SECRET,
    namespace: process.env.APAAS_NAMESPACE,
  });
  await client.init();

  const operator = {
    _id: Number(process.env.APAAS_OPERATOR_ID || 100),
    email: process.env.APAAS_OPERATOR_EMAIL || 'sample@feishu.cn',
  };

  const runner = useV1 ? client.automation.v1 : client.automation.v2;
  const res = await runner.execute({ flow_api_name: flowApiName, operator, params });
  console.log(JSON.stringify(res, null, 2));
}

main().catch(err => {
  console.error('Error:', err.response?.data || err.message);
  process.exit(1);
});
```
### 8.6 `invoke-function.js`

```javascript
/**
 * 调用 aPaaS 云函数
 * 用法: node scripts/invoke-function.js <function_name> [params.json]
 */
const fs = require('fs');
const { apaas } = require('apaas-oapi-client');

async function main() {
  const [name, paramsPath] = process.argv.slice(2);
  if (!name) {
    console.error('Usage: node scripts/invoke-function.js <function_name> [params.json]');
    process.exit(1);
  }
  const params = paramsPath ? JSON.parse(fs.readFileSync(paramsPath, 'utf-8')) : {};

  const client = new apaas.Client({
    clientId: process.env.APAAS_CLIENT_ID,
    clientSecret: process.env.APAAS_CLIENT_SECRET,
    namespace: process.env.APAAS_NAMESPACE,
  });
  await client.init();

  const res = await client.function.invoke({ name, params });
  console.log(JSON.stringify(res, null, 2));
}

main().catch(err => {
  console.error('Error:', err.response?.data || err.message);
  process.exit(1);
});
```