# 标签归一化机制分析报告

## 概述

Karakeep 实现了**两层标签归一化机制**，确保不同格式的标签名能够正确匹配到同一条记录，避免重复标签的产生。该机制在**入库**和**查询**两个阶段同时生效。

---

## 归一化层次结构

### 第一层：展示层归一化 (`normalizeTagName`)

**位置**：`packages/shared/utils/tag.ts:6-8`

```typescript
export function normalizeTagName(raw: string): string {
  return raw.trim().replace(/^#+/, ""); // strip every leading #
}
```

**作用**：
- 去除标签名首尾的空白字符
- 去除所有前置的 `#` 符号
- 用于标签创建、更新和附加操作的输入预处理

**示例**：
| 输入 | 输出 |
|------|------|
| `  #Machine Learning  ` | `Machine Learning` |
| `##web-development` | `web-development` |
| `  ai_tag  ` | `ai_tag` |

---

### 第二层：匹配层归一化 (数据库 `normalizedName`)

**位置**：
- 模式定义：`packages/db/schema.ts:426-433`
- 迁移文件：`packages/db/drizzle/0071_add_normalized_tag_name.sql`

```typescript
normalizedName: text("normalizedName").generatedAlwaysAs(
  (): SQL =>
    sql`lower(replace(replace(replace(${bookmarkTags.name}, ' ', ''), '-', ''), '_', ''))`,
  {
    mode: "virtual",
  },
),
```

**SQL 等价**：
```sql
lower(replace(replace(replace(name, ' ', ''), '-', ''), '_', ''))
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

**索引**：在 `normalizedName` 列上建有索引 `bookmarkTags_normalizedName_idx`，确保匹配查询的高性能。

---

## 归一化应用场景

### 场景 1：AI 自动标签匹配

**位置**：`apps/workers/workers/inference/tagging.ts:391-443`

**工作流程**：
1. AI 推断出标签列表（可能格式不一致）
2. 对每个推断标签应用归一化函数：
   ```typescript
   function normalizeTag(tag: string) {
     return tag.toLowerCase().replace(/[ \-_]/g, "");
   }
   ```
3. 使用 `normalizedName` 列查询用户现有标签
4. 匹配成功则复用现有标签 ID，匹配失败则创建新标签

```typescript
const matchedTags = await tx.query.bookmarkTags.findMany({
  where: and(
    eq(bookmarkTags.userId, userId),
    inArray(
      bookmarkTags.normalizedName,
      normalizedInferredTags.map((t) => t.normalizedTag),
    ),
  ),
});
```

### 场景 2：重复标签检测与合并建议

**位置**：`apps/web/components/dashboard/cleanups/TagDuplicationDetention.tsx:36-247`

**工作流程**：
1. 获取用户所有标签
2. 对每个标签名应用本地归一化函数
3. 按归一化值排序并检测编辑距离
4. 建议用户合并相似标签（如 "Machine Learning" 和 "machine-learning"）

```typescript
function normalizeTag(tag: string) {
  return tag.toLocaleLowerCase().replace(/[ -_]/g, "");
}
```

### 场景 3：标签创建与更新

**位置**：
- `packages/shared/types/tags.ts:8-11` (Zod schema transform)
- `packages/trpc/models/tags.ts:59-82` (create)
- `packages/trpc/models/tags.ts:333-384` (update)

**工作流程**：
1. Zod schema 在验证时自动应用 `normalizeTagName`
2. 数据库唯一约束 `(userId, name)` 确保同一用户不会有相同显示名的标签
3. 更新时同样进行归一化处理

```typescript
const zTagNameSchemaWithValidation = z
  .string()
  .transform((s) => normalizeTagName(s).trim())
  .pipe(z.string().min(1));
```

### 场景 4：书签标签附加

**位置**：`packages/trpc/routers/bookmarks.ts:1030-1051`

**工作流程**：
1. 对要附加的标签名应用 `normalizeTagName`
2. 创建不存在的标签（使用 `onConflictDoNothing` 避免重复）
3. 通过归一化后的名称匹配获取标签 ID
4. 建立书签与标签的关联

```typescript
const normalizedAttachTags = input.attach.map((tag) => ({
  tagId: tag.tagId,
  tagName: tag.tagName ? normalizeTagName(tag.tagName) : undefined,
  attachedBy: tag.attachedBy,
}));
```

---

## 关键设计要点

### 1. 同步性要求

数据库生成列的归一化逻辑必须与应用层的归一化逻辑**完全一致**：

- **数据库端**：`lower(replace(replace(replace(name, ' ', ''), '-', ''), '_', ''))`
- **应用端**：`tag.toLowerCase().replace(/[ \-_]/g, "")`

两处代码都有注释明确标注了这种同步依赖关系。

### 2. 性能优化

- `normalizedName` 是**虚拟列**（不占用存储空间）
- 建有专门的 B-tree 索引支持快速匹配查询
- 标签创建使用 `onConflictDoNothing` 避免错误处理开销

### 3. 用户体验

- 保留用户输入的原始格式作为显示名
- 透明处理格式差异，用户无需关心归一化细节
- 提供重复标签检测和合并工具供用户主动清理

---

## 归一化覆盖范围

### 能够匹配的格式差异

| 差异类型 | 示例 | 归一化后 |
|---------|------|---------|
| 大小写 | `Machine Learning` vs `machine learning` | `machinelearning` |
| 空格分隔 | `machine learning` | `machinelearning` |
| 连字符分隔 | `machine-learning` | `machinelearning` |
| 下划线分隔 | `machine_learning` | `machinelearning` |
| 驼峰命名 | `machineLearning` | `machinelearning` |
| 帕斯卡命名 | `MachineLearning` | `machinelearning` |

### 不处理的差异

- 拼写错误（如 `machin learning` vs `machine learning`）
- 同义词（如 `AI` vs `Artificial Intelligence`）
- 单复数差异（如 `tag` vs `tags`）
- 其他标点符号（如 `.`, `,`, `!` 等）

---

## 代码位置索引

| 功能模块 | 文件路径 | 关键行号 |
|---------|---------|---------|
| 展示层归一化函数 | `packages/shared/utils/tag.ts` | 6-8 |
| 数据库归一化列 | `packages/db/schema.ts` | 426-433 |
| 数据库迁移 | `packages/db/drizzle/0071_add_normalized_tag_name.sql` | 1-2 |
| AI 标签匹配逻辑 | `apps/workers/workers/inference/tagging.ts` | 74-77, 391-443 |
| 重复标签检测 | `apps/web/components/dashboard/cleanups/TagDuplicationDetention.tsx` | 36-38, 219-227 |
| Zod schema 归一化 | `packages/shared/types/tags.ts` | 8-11 |
| 书签标签附加 | `packages/trpc/routers/bookmarks.ts` | 1030-1051 |
| 标签创建/更新 | `packages/trpc/models/tags.ts` | 59-82, 333-384 |

---

## 注意事项

1. **修改归一化逻辑需要数据迁移**：如果更改归一化规则，需要对现有标签重新计算 `normalizedName` 并更新关联关系。

2. **新增分隔符需要同步更新**：如果需要支持更多分隔符（如 `.`、`/` 等），需要同时更新：
   - 数据库生成列的 SQL 表达式
   - `tagging.ts` 中的正则表达式
   - `TagDuplicationDetention.tsx` 中的正则表达式

3. **跨语言支持**：当前归一化主要针对拉丁字符，非拉丁字符（如中文、日文）的空格和大小写处理需要特别考虑。
