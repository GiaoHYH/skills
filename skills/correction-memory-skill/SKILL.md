---
name: correction-memory-skill
description: 自动沉淀和复用“纠错经验”的技能。每当用户指出回答、代码、流程、文件生成或工具使用中的问题，并给出纠正意见时，必须使用本技能把问题模式、根因、修正方式和可复用规则记录到本地案例库；每当遇到报错、返工、用户说“不对/不是这样/应该/改成/刚才错了/以后遇到类似问题”等场景时，也必须先用本技能检索是否有类似历史案例，再决定修复方案。本技能支持自动迭代：相似问题会合并、计数、更新规则，而不是重复堆积。
---

# Correction Memory Skill

> 用于把“被用户纠正过的问题”沉淀为可检索、可合并、可迭代的经验库。目标是减少重复犯错：遇到新问题时先找相似历史，纠正后再把新经验写回。

---

## 1. 使用时机

### 1.1 必须使用

当出现以下任一情况时，先使用本 skill：

1. 用户指出结果不对：例如“不是这个”“你搞错了”“不应该这样”“格式不对”“继续改”“不要那些 js，全部集成到 md 文件中”。
2. 用户给出纠正规则：例如“以后遇到 X 要 Y”“每次都要先做 A”“不要再 B”。
3. 工具/命令/代码执行失败，需要分析错误并修复。
4. 正在开始一个可能重复踩坑的任务：例如打包 skill、生成文档、修改代码、处理 API 调用、处理文件结构。
5. 用户要求“记录类似问题”“下次遇到类似问题先查一下”“自动迭代”。

### 1.2 不需要使用

- 一次性常识问答，没有纠错、返工或错误风险。
- 用户明确说“不要记录”“临时处理，不作为规则”。
- 内容包含敏感凭证、个人隐私或不应持久化的信息；此时只记录抽象模式，不记录原文敏感值。

---

## 2. 核心原则

1. **先检索，再行动**：遇到问题或返工前，先查案例库有没有相似问题。
2. **纠正后写回**：用户纠正后，把“问题模式 → 根因 → 正确做法”沉淀下来。
3. **合并优先，不重复堆积**：相似案例更新 `hit_count`、`last_seen`、`rule` 和 `examples`，不要每次新建。
4. **抽象成规则**：记录可复用的行为约束，而不是只记录当次现象。
5. **保护隐私**：不要保存 token、secret、邮箱、工号、真实用户标识、内部链接全文等敏感数据；必要时脱敏。
6. **短而准**：检索结果只给执行者看，最终回复用户时只说明“已吸收/已更新规则”，不要暴露内部存储细节。

---

## 3. 案例库位置

默认使用当前用户持久目录：

```text
/data/userdata/correction-memory/cases.jsonl
/data/userdata/correction-memory/index.json
```

- `cases.jsonl`：一行一个案例，便于追加和 grep。
- `index.json`：聚合索引，便于合并、统计和检索。

如果目录不存在，创建它。

---

## 4. 案例结构

每条案例使用 JSON 对象：

```json
{
  "id": "cm_20260513_001",
  "created_at": "2026-05-13T14:00:00+08:00",
  "last_seen": "2026-05-13T14:20:00+08:00",
  "hit_count": 3,
  "status": "active",
  "severity": "medium",
  "domain": "skill_packaging",
  "problem_pattern": "用户要求 Trae skill 只保留 SKILL.md，但生成了额外 scripts/*.js 文件",
  "symptoms": ["用户说不要那些 js", "zip 中包含 scripts 目录"],
  "root_cause": "没有把示例脚本内联到 Markdown，而是按传统 skill 结构创建了脚本文件",
  "correction_rule": "当用户要求技能内容全部集成到 md 文件中时，zip 根目录只允许 SKILL.md；示例代码以内联 fenced code block 形式写入 SKILL.md",
  "recommended_action": "重新生成单文件 skill，删除 scripts 目录，打包前用 unzip -l 校验只有 SKILL.md",
  "anti_patterns": ["保留 scripts/", "额外生成 package.json"],
  "examples": [
    {
      "user_correction": "不要那些 js，全部集成到 md 文件中",
      "fixed_action": "把 6 个 JS 示例迁移到 SKILL.md 第 7 节，zip 内仅一个 SKILL.md"
    }
  ],
  "tags": ["trae-skill", "markdown-only", "packaging"],
  "confidence": 0.95
}
```

字段说明：
- `domain`：任务域，例如 `skill_packaging`、`code_editing`、`api_calling`、`doc_generation`、`tool_usage`。
- `problem_pattern`：可泛化的问题模式。
- `correction_rule`：下次遇到类似问题时应遵循的规则。
- `recommended_action`：可执行修复步骤。
- `anti_patterns`：应避免的做法。
- `hit_count`：相似问题重复出现次数，用于自动迭代。

---

## 5. 工作流

### 5.1 遇到问题时：检索相似案例

1. 提取当前问题关键词：任务类型、文件类型、错误信息、用户纠正语句、期望输出。
2. 调用检索脚本或用 Bash/Python 搜索案例库。
3. 找到相似案例后，优先应用其 `correction_rule` 和 `recommended_action`。
4. 如果没有相似案例，按常规分析并解决。

推荐命令：

```bash
python /data/userdata/correction-memory/search_cases.py "用户纠正语句或错误摘要"
```

如果脚本不存在，按第 7 节创建。

### 5.2 用户纠正后：记录或合并案例

1. 总结这次错误：
   - 我做错了什么？
   - 用户真正想要什么？
   - 根因是什么？
   - 下次应如何避免？
2. 检索是否已有相似案例。
3. 如果相似度高：更新旧案例。
   - `hit_count += 1`
   - `last_seen = now`
   - 补充 `examples`
   - 必要时强化 `correction_rule`
4. 如果没有相似案例：新增案例。

推荐命令：

```bash
python /data/userdata/correction-memory/upsert_case.py --domain skill_packaging --problem "..." --rule "..." --action "..." --tags "trae-skill,markdown-only"
```

### 5.3 自动迭代机制

每次 upsert 时执行：

1. **相似合并**：按关键词 Jaccard + 字符串包含判断相似度。
2. **规则增强**：新纠正比旧规则更具体时，更新 `correction_rule`。
3. **频次提升**：`hit_count` 增加后，如果 >= 3，将 `severity` 提升一级。
4. **反模式归纳**：把重复出现的错误做法加入 `anti_patterns`。
5. **索引重建**：写入后刷新 `index.json`。

---

## 6. 检索结果使用方式

检索返回相似案例时，执行者应这样处理：

1. 不要把完整案例 JSON 暴露给用户。
2. 在内部决策中遵守案例规则。
3. 如果用户正在纠正问题，可以简短回应：
   - “收到，我会按这个规则处理，并把这类问题合并到纠错经验里。”
4. 如果案例直接影响修复方案，可以自然说明：
   - “我会重新打包为单文件 SKILL.md，并先校验 zip 内没有额外脚本。”

---

## 7. 内置脚本模板

> 首次使用时，如果 `/data/userdata/correction-memory/search_cases.py` 或 `upsert_case.py` 不存在，创建下面两个脚本。脚本可持久化复用。

### 7.1 search_cases.py

```python
#!/usr/bin/env python3
import json, os, re, sys
from pathlib import Path

BASE = Path('/data/userdata/correction-memory')
CASES = BASE / 'cases.jsonl'

def tokenize(s):
    s = (s or '').lower()
    return set(re.findall(r'[a-z0-9_\-]+|[一-鿿]+', s))

def score(query, case):
    q = tokenize(query)
    fields = ' '.join([
        case.get('domain',''),
        case.get('problem_pattern',''),
        case.get('root_cause',''),
        case.get('correction_rule',''),
        case.get('recommended_action',''),
        ' '.join(case.get('symptoms', [])),
        ' '.join(case.get('tags', [])),
    ])
    c = tokenize(fields)
    if not q or not c:
        return 0.0
    jaccard = len(q & c) / max(1, len(q | c))
    contains = 0.25 if query.lower() in fields.lower() or fields.lower() in query.lower() else 0
    tag_bonus = 0.05 * len(q & set(case.get('tags', [])))
    hit_bonus = min(case.get('hit_count', 1), 10) * 0.01
    return min(1.0, jaccard + contains + tag_bonus + hit_bonus)

def load_cases():
    if not CASES.exists():
        return []
    out = []
    for line in CASES.read_text(encoding='utf-8').splitlines():
        if line.strip():
            try:
                out.append(json.loads(line))
            except Exception:
                pass
    return out

def main():
    query = ' '.join(sys.argv[1:]).strip()
    if not query:
        print('Usage: search_cases.py <query>', file=sys.stderr)
        sys.exit(2)
    cases = load_cases()
    ranked = sorted(((score(query, c), c) for c in cases), key=lambda x: x[0], reverse=True)
    results = [{'score': round(s, 3), **c} for s, c in ranked if s >= 0.08][:5]
    print(json.dumps({'query': query, 'count': len(results), 'results': results}, ensure_ascii=False, indent=2))

if __name__ == '__main__':
    main()
```

### 7.2 upsert_case.py

```python
#!/usr/bin/env python3
import argparse, json, os, re
from datetime import datetime, timezone, timedelta
from pathlib import Path

BASE = Path('/data/userdata/correction-memory')
CASES = BASE / 'cases.jsonl'
INDEX = BASE / 'index.json'
TZ = timezone(timedelta(hours=8))

def now():
    return datetime.now(TZ).isoformat(timespec='seconds')

def tokenize(s):
    s = (s or '').lower()
    return set(re.findall(r'[a-z0-9_\-]+|[一-鿿]+', s))

def sim(a, b):
    ta, tb = tokenize(a), tokenize(b)
    if not ta or not tb:
        return 0.0
    return len(ta & tb) / max(1, len(ta | tb))

def load_cases():
    if not CASES.exists():
        return []
    out = []
    for line in CASES.read_text(encoding='utf-8').splitlines():
        if line.strip():
            try:
                out.append(json.loads(line))
            except Exception:
                pass
    return out

def save_cases(cases):
    BASE.mkdir(parents=True, exist_ok=True)
    CASES.write_text('\n'.join(json.dumps(c, ensure_ascii=False) for c in cases) + ('\n' if cases else ''), encoding='utf-8')
    index = {
        'updated_at': now(),
        'total': len(cases),
        'domains': {},
        'tags': {},
        'top_cases': sorted(
            [{'id': c['id'], 'hit_count': c.get('hit_count', 1), 'problem_pattern': c.get('problem_pattern', '')} for c in cases],
            key=lambda x: x['hit_count'], reverse=True
        )[:20]
    }
    for c in cases:
        index['domains'][c.get('domain','unknown')] = index['domains'].get(c.get('domain','unknown'), 0) + 1
        for t in c.get('tags', []):
            index['tags'][t] = index['tags'].get(t, 0) + 1
    INDEX.write_text(json.dumps(index, ensure_ascii=False, indent=2), encoding='utf-8')

def severity_after(hit_count, current='medium'):
    order = ['low', 'medium', 'high', 'critical']
    if hit_count >= 6:
        return 'critical'
    if hit_count >= 3:
        return 'high'
    return current if current in order else 'medium'

def main():
    p = argparse.ArgumentParser()
    p.add_argument('--domain', default='general')
    p.add_argument('--problem', required=True)
    p.add_argument('--rule', required=True)
    p.add_argument('--action', default='')
    p.add_argument('--root-cause', default='')
    p.add_argument('--symptoms', default='')
    p.add_argument('--anti-patterns', default='')
    p.add_argument('--tags', default='')
    p.add_argument('--user-correction', default='')
    p.add_argument('--fixed-action', default='')
    args = p.parse_args()

    BASE.mkdir(parents=True, exist_ok=True)
    cases = load_cases()
    target_text = ' '.join([args.domain, args.problem, args.rule, args.action, args.tags])

    best_i, best_s = None, 0.0
    for i, c in enumerate(cases):
        old_text = ' '.join([c.get('domain',''), c.get('problem_pattern',''), c.get('correction_rule',''), ' '.join(c.get('tags', []))])
        s = sim(target_text, old_text)
        if s > best_s:
            best_i, best_s = i, s

    tags = [x.strip() for x in args.tags.split(',') if x.strip()]
    symptoms = [x.strip() for x in args.symptoms.split('|') if x.strip()]
    anti_patterns = [x.strip() for x in args.anti_patterns.split('|') if x.strip()]
    example = {}
    if args.user_correction:
        example['user_correction'] = args.user_correction
    if args.fixed_action:
        example['fixed_action'] = args.fixed_action

    if best_i is not None and best_s >= 0.28:
        c = cases[best_i]
        c['last_seen'] = now()
        c['hit_count'] = c.get('hit_count', 1) + 1
        c['severity'] = severity_after(c['hit_count'], c.get('severity', 'medium'))
        if len(args.rule) > len(c.get('correction_rule', '')):
            c['correction_rule'] = args.rule
        if args.action and len(args.action) > len(c.get('recommended_action', '')):
            c['recommended_action'] = args.action
        if args.root_cause and not c.get('root_cause'):
            c['root_cause'] = args.root_cause
        c['symptoms'] = sorted(set(c.get('symptoms', []) + symptoms))
        c['anti_patterns'] = sorted(set(c.get('anti_patterns', []) + anti_patterns))
        c['tags'] = sorted(set(c.get('tags', []) + tags))
        if example:
            c.setdefault('examples', []).append(example)
            c['examples'] = c['examples'][-5:]
        action = 'updated'
        out = c
    else:
        cid = f"cm_{datetime.now(TZ).strftime('%Y%m%d_%H%M%S')}"
        out = {
            'id': cid,
            'created_at': now(),
            'last_seen': now(),
            'hit_count': 1,
            'status': 'active',
            'severity': 'medium',
            'domain': args.domain,
            'problem_pattern': args.problem,
            'symptoms': symptoms,
            'root_cause': args.root_cause,
            'correction_rule': args.rule,
            'recommended_action': args.action,
            'anti_patterns': anti_patterns,
            'examples': [example] if example else [],
            'tags': tags,
            'confidence': 0.8,
        }
        cases.append(out)
        action = 'created'

    save_cases(cases)
    print(json.dumps({'action': action, 'similarity': round(best_s, 3), 'case': out}, ensure_ascii=False, indent=2))

if __name__ == '__main__':
    main()
```

---

## 8. 首次初始化命令

执行者可用下面命令初始化目录：

```bash
mkdir -p /data/userdata/correction-memory
```

然后用 `Write` 工具把第 7 节两个脚本分别写入：

```text
/data/userdata/correction-memory/search_cases.py
/data/userdata/correction-memory/upsert_case.py
```

并设置可执行权限：

```bash
chmod +x /data/userdata/correction-memory/search_cases.py /data/userdata/correction-memory/upsert_case.py
```

---

## 9. 操作范例

### 9.1 检索类似问题

```bash
python /data/userdata/correction-memory/search_cases.py "用户要求 skill 不要 js 全部集成到 md"
```

期望返回：

```json
{
  "query": "用户要求 skill 不要 js 全部集成到 md",
  "count": 1,
  "results": [
    {
      "score": 0.42,
      "problem_pattern": "用户要求 Trae skill 只保留 SKILL.md，但生成了额外 scripts/*.js 文件",
      "correction_rule": "当用户要求技能内容全部集成到 md 文件中时，zip 根目录只允许 SKILL.md..."
    }
  ]
}
```

### 9.2 记录/合并纠错案例

```bash
python /data/userdata/correction-memory/upsert_case.py \
  --domain skill_packaging \
  --problem "用户要求 Trae skill 只保留 SKILL.md，但生成了额外 scripts/*.js 文件" \
  --rule "当用户要求技能内容全部集成到 md 文件中时，zip 根目录只允许 SKILL.md；示例代码以内联 fenced code block 写入 SKILL.md" \
  --action "删除 scripts 目录，重新打包，并用 unzip -l 校验 zip 内只有 SKILL.md" \
  --symptoms "用户说不要那些 js|zip 中出现 scripts 目录" \
  --anti-patterns "保留 scripts/|额外生成 package.json" \
  --tags "trae-skill,markdown-only,packaging" \
  --user-correction "不要那些 js，全部集成到 md 文件中" \
  --fixed-action "重构为单文件 SKILL.md 并重新打包"
```

---

## 10. 输出习惯

使用本 skill 后，对用户保持简洁：

- 如果只是检索并应用：
  - “我会先按类似问题的经验处理：先校验结构，再生成结果。”
- 如果用户刚纠正了你：
  - “收到，这类问题我会记录为规则：以后遇到同类需求优先按 X 处理。”
- 如果新增/更新案例：
  - “已把这类纠错合并到经验库，后续遇到类似场景会先检索再执行。”

不要输出完整 JSON、内部路径或脚本细节，除非用户明确要求。