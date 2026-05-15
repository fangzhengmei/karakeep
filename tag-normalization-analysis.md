# 标签归一化机制分析报告

## 概述

Karakeep 实现了**两层标签归一化机制**，但各入口的实现并不一致。本报告详细梳理每个入库与查询路径的实际行为，明确哪些路径能正确命中同一条记录，哪些路径仍会创建重复标签。

---

## 归一化层次结构

### 第一层：展示层归一化 (`normalizeTagName`)

**位置**：`packages/shared/utils/tag.ts:6-8`

```typescript
export function normalizeTagName(raw: string): string {
  return raw.trim().replace(/^#+/, ""); // strip every leading #
}
```

**行为**：
- 去除标签名首尾的空白字符
- 去除所有前置的 `#` 符号
- **不处理**大小写、空格、连字符、下划线差异

---

### 第二层：匹配层归一化 (数据库 `normalizedName`)

**位置**：`packages/db/schema.ts:426-433`

```typescript
normalizedName: text("normalizedName").generatedAlwaysAs(
  (): SQL =>
    sql`lower(replace(replace(replace(${bookmarkTags.name}, ' ', ''), '-', ''), '_', ''))`,
  {
    mode: "virtual",
  },
),
```

**归一化步骤**：
1. 去除所有空格 ` `
2. 去除所有连字符 `-`
3. 去除所有下划线 `_`
4. 转换为全小写

**示例**：
| 原始标签名 | normalizedName 值 |
|-----------|-------------------|
| `Machine Learning` | `machinelearning` |
| `machine-learning` | `machinelearning` |
| `machine_learning` | `machinelearning` |
| `MachineLearning`  | `machinelearning` |

**索引**：建有 `bookmarkTags_normalizedName_idx` 索引支持高性能匹配。

---

## 各入口归一化行为详解

### ✅ 路径 1：AI 自动标签匹配（能正确匹配）

**位置**：`apps/workers/workers/inference/tagging.ts:401-448`

**归一化行为**：
1. 对每个推断标签应用完整归一化：`tag.toLowerCase().replace(/[ \-_]/g, "")`
2. **使用 `normalizedName` 列**进行 `IN` 查询匹配
3. 匹配成功则复用现有标签 ID，失败则创建新标签

**关键代码**：
```typescript
const matchedTags = await tx.query.bookmarkTags.findMany({
  where: and(
    eq(bookmarkTags.userId, userId),
    inArray(
      bookmarkTags.normalizedName,  // ✅ 使用 normalizedName 列
      normalizedInferredTags.map((t) => t.normalizedTag),
    ),
  ),
});
```

**可复现示例**：
```
现有标签: "Machine Learning" (normalizedName: "machinelearning")
AI 推断标签: "machine-learning"

✅ 结果：命中现有标签，不会创建新标签
```

**边界说明**：
- 能正确处理大小写、空格、连字符、下划线的任意组合
- 创建新标签时保留 AI 输出的原始格式作为显示名

---

### ❌ 路径 2：updateTags API（会创建重复标签）

**位置**：`packages/trpc/routers/bookmarks.ts:1030-1057`

**归一化行为**：
1. 仅应用展示层归一化 `normalizeTagName`（去空白和 # 前缀）
2. **使用 `name` 列**进行精确匹配（`inArray(bookmarkTags.name, tagNames)`）
3. 不匹配则创建新标签

**关键代码**：
```typescript
// ❌ 仅展示层归一化
const normalizedAttachTags = input.attach.map((tag) => ({
  tagId: tag.tagId,
  tagName: tag.tagName ? normalizeTagName(tag.tagName) : undefined,
  attachedBy: tag.attachedBy,
}));

// ❌ 使用 name 列精确匹配，不是 normalizedName
tagNames.length > 0
  ? ctx.db
      .select({ id: bookmarkTags.id, name: bookmarkTags.name })
      .from(bookmarkTags)
      .where(
        and(
          eq(bookmarkTags.userId, ctx.user.id),
          inArray(bookmarkTags.name, tagNames),  // ❌ 精确匹配 name
        ),
      )
  : Promise.resolve([]),
```

**可复现示例**：
```
现有标签: "Machine Learning"
用户输入标签: "machine-learning"

❌ 结果：不匹配，创建新标签 "machine-learning"
   数据库中出现两条记录，normalizedName 都是 "machinelearning"
```

**边界说明**：
- 仅能匹配完全相同的字符串（除了空白和 # 前缀）
- 大小写、空格、连字符、下划线的差异都会导致不匹配
- 这是**最容易产生重复标签**的路径

---

### ❌ 路径 3：创建标签 API（会创建重复标签）

**位置**：
- Schema: `packages/shared/types/tags.ts:8-11`
- Model: `packages/trpc/models/tags.ts:59-82`

**归一化行为**：
1. Zod schema 仅应用 `normalizeTagName`（去空白和 # 前缀）
2. 数据库唯一约束是 `(userId, name)`，不是 `(userId, normalizedName)`
3. 不同格式但归一化后相同的标签会被视为不同标签

**关键代码**：
```typescript
const zTagNameSchemaWithValidation = z
  .string()
  .transform((s) => normalizeTagName(s).trim())  // ❌ 仅展示层归一化
  .pipe(z.string().min(1));
```

**可复现示例**：
```
第一次请求: POST /tags { "name": "Machine Learning" } → ✅ 创建成功
第二次请求: POST /tags { "name": "machine-learning" } → ✅ 创建成功（不冲突！）

❌ 结果：数据库中有两条记录，normalizedName 相同但 name 不同
```

**边界说明**：
- 数据库唯一约束仅基于 `name` 列，不是 `normalizedName`
- 用户可以创建任意数量格式不同但归一化后相同的标签

---

### ❌ 路径 4：更新标签名称（可能导致冲突）

**位置**：`packages/trpc/models/tags.ts:333-384`

**归一化行为**：
1. 同样仅应用展示层归一化
2. 基于 `name` 列检查唯一性冲突

**可复现示例**：
```
现有标签 A: "Machine Learning"
现有标签 B: "machine-learning"

尝试将标签 A 重命名为 "machine-learning" → ❌ 冲突错误
```

---

### ❌ 路径 5：按标签名搜索书签（精确匹配）

**位置**：`packages/trpc/lib/search.ts:102-128`

**归一化行为**：
1. 不进行任何归一化
2. 直接使用 `eq(bookmarkTags.name, matcher.tagName)` 精确匹配

**关键代码**：
```typescript
case "tagName": {
  const comp = matcher.inverse ? notExists : exists;
  return db
    .selectDistinct({ id: bookmarks.id })
    .from(bookmarks)
    .where(
      and(
        eq(bookmarks.userId, userId),
        comp(
          db
            .select()
            .from(tagsOnBookmarks)
            .innerJoin(
              bookmarkTags,
              eq(tagsOnBookmarks.tagId, bookmarkTags.id),
            )
            .where(
              and(
                eq(tagsOnBookmarks.bookmarkId, bookmarks.id),
                eq(bookmarkTags.userId, userId),
                eq(bookmarkTags.name, matcher.tagName),  // ❌ 精确匹配
              ),
            ),
        ),
      ),
    );
}
```

**可复现示例**：
```
书签带有标签: "Machine Learning"
搜索查询: tag:"machine-learning"

❌ 结果：找不到该书签
```

---

### ❌ 路径 6：标签列表搜索（部分匹配）

**位置**：`packages/trpc/models/tags.ts:84-182`

**归一化行为**：
1. 使用 `LIKE` 进行子字符串匹配
2. 不进行归一化处理

**关键代码**：
```typescript
opts.nameContains
  ? like(bookmarkTags.name, `%${opts.nameContains}%`)  // ❌ 直接 LIKE 匹配
  : undefined,
```

**可复现示例**：
```
现有标签: "Machine Learning"
搜索: nameContains: "machine"

✅ 能找到（因为是子字符串匹配，SQLite LIKE 默认不区分大小写）

搜索: nameContains: "machinelearning"
❌ 找不到（因为原始名称中有空格）
```

---

### ✅ 路径 7：重复标签检测工具（能正确识别）

**位置**：`apps/web/components/dashboard/cleanups/TagDuplicationDetention.tsx:36-247`

**归一化行为**：
1. 在前端应用完整归一化：`tag.toLocaleLowerCase().replace(/[ -_]/g, "")`
2. 按归一化值排序，检测编辑距离
3. 建议用户合并相似标签

**这是一个补救工具，不是防止重复的机制。**

---

### 其他路径

#### MCP Server 标签操作
**位置**：`apps/mcp/src/tags.ts`
- 通过 API 调用 `updateTags`，行为同路径 2

#### CLI 标签操作
**位置**：`apps/cli/src/commands/tags.ts`
- 通过 API 调用，行为同上

#### 导入功能
**位置**：`apps/workers/workers/importWorker.ts`、`packages/shared/import-export/parsers.ts`
- 直接使用解析出的标签字符串，不做归一化处理
- 行为同路径 2

---

## 行为对比矩阵

| 入口/路径 | 使用 normalizedName | 能跨格式匹配 | 会创建重复标签 | 备注 |
|---------|-------------------|------------|--------------|------|
| AI 自动标签 | ✅ 是 | ✅ 能 | ❌ 不会 | 唯一正确实现完整归一化的路径 |
| updateTags API | ❌ 否 | ❌ 不能 | ✅ 会 | 仅展示层归一化 |
| create tag API | ❌ 否 | ❌ 不能 | ✅ 会 | 唯一约束基于 name 列 |
| update tag name | ❌ 否 | ❌ 不能 | - | 基于 name 检查冲突 |
| tagName 搜索书签 | ❌ 否 | ❌ 不能 | - | 精确匹配 |
| tags list 搜索 | ❌ 否 | 部分（子字符串） | - | LIKE 匹配 |
| 重复标签检测工具 | ✅ 是（前端） | ✅ 能 | - | 仅用于检测，不防止 |
| 导入功能 | ❌ 否 | ❌ 不能 | ✅ 会 | 直接使用源数据 |
| MCP/CLI | ❌ 否 | ❌ 不能 | ✅ 会 | 通过 API |

---

## 核心问题总结

### 1. 归一化逻辑不一致
- AI 标签匹配：使用完整归一化 + `normalizedName` 列 ✅
- 所有其他路径：仅展示层归一化 + `name` 列 ❌

### 2. 数据库约束设计缺陷
- 当前唯一约束：`UNIQUE(userId, name)` ❌
- 应该是：`UNIQUE(userId, normalizedName)` ✅

### 3. 搜索体验不一致
- 不同格式的标签实际上是"同一个"标签，但搜索和附加都无法命中

---

## 边界情况说明

### 情况 1：大小写差异
```
标签 A: "Machine Learning"
标签 B: "machine learning"

normalizedName 相同："machinelearning"

AI 匹配：✅ 命中同一个
updateTags：❌ 创建重复
搜索：❌ 不匹配
```

### 情况 2：分隔符差异
```
标签 A: "machine learning" (空格)
标签 B: "machine-learning" (连字符)
标签 C: "machine_learning" (下划线)

normalizedName 相同："machinelearning"

AI 匹配：✅ 全部命中同一个
updateTags：❌ 创建 3 个不同标签
搜索：❌ 3 个标签互相不匹配
```

### 情况 3：混合差异
```
标签 A: "Machine Learning"
标签 B: "machine-learning"
标签 C: "MACHINE_LEARNING"

normalizedName 相同："machinelearning"

AI 匹配：✅ 全部命中同一个
updateTags：❌ 创建 3 个不同标签
```

### 情况 4：其他标点符号
```
标签 A: "machine.learning" (点号)
标签 B: "machine/learning" (斜杠)

normalizedName 不同："machine.learning" vs "machine/learning"
（因为 SQL 表达式只替换空格、连字符、下划线）

⚠️ 即使是 AI 匹配也会创建两个不同标签
```

---

## 代码位置索引

| 功能模块 | 文件路径 | 关键行号 |
|---------|---------|---------|
| 展示层归一化函数 | `packages/shared/utils/tag.ts` | 6-8 |
| 数据库 normalizedName 列 | `packages/db/schema.ts` | 426-433 |
| AI 标签匹配逻辑 | `apps/workers/workers/inference/tagging.ts` | 401-448 |
| updateTags 标签匹配 | `packages/trpc/routers/bookmarks.ts` | 990-1028, 1030-1057 |
| Zod schema 归一化 | `packages/shared/types/tags.ts` | 8-11 |
| create tag 模型 | `packages/trpc/models/tags.ts` | 59-82 |
| update tag 模型 | `packages/trpc/models/tags.ts` | 333-384 |
| tagName 搜索书签 | `packages/trpc/lib/search.ts` | 102-128 |
| 标签列表搜索 | `packages/trpc/models/tags.ts` | 84-182 |
| 重复标签检测工具 | `apps/web/components/dashboard/cleanups/TagDuplicationDetention.tsx` | 36-38, 219-227 |

---

## 改进建议

### 短期修复（最小改动）
1. **修改 `fetchTagIdsWithNames`**：在 `updateTags` 中使用 `normalizedName` 匹配
2. **修改标签搜索**：在 `search.ts` 的 `tagName` 匹配中使用 `normalizedName`

### 中期修复（数据库约束）
1. **添加唯一约束**：`UNIQUE(userId, normalizedName)`
2. **数据迁移**：合并现有重复标签

### 长期修复（统一架构）
1. **创建归一化工具函数**：在 shared 包中提供统一的 `normalizeTagForMatching` 函数
2. **所有路径统一使用**：确保所有入口都使用相同的归一化逻辑
3. **添加测试覆盖**：为每个入口添加归一化行为测试
