# OpenCode Memory 详细技术分析

本文档对 OpenCode Memory 项目进行深度技术拆解，面向希望阅读源码并复用其设计的开发者。

---

## 1. 核心模块概述与技术选型理由

OpenCode Mem 是一个**持久化混合检索向量记忆插件**，为 OpenCode 编码代理提供长期记忆能力。核心模块划分：

| 模块                     | 职责          | 技术选型                             | 选型理由                                                                               |
| ------------------------ | ------------- | ------------------------------------ | -------------------------------------------------------------------------------------- |
| **ShardManager**         | 分片管理      | SQLite + 文件分片                    | SQLite 天生单文件事务，按 scope/hash 分组便于垂直扩展，避免单个大文件导致 VFS 性能下降 |
| **VectorSearch**         | 向量检索协调  | 混合后端架构                         | 主后端用 USearch 近似最近邻，fallback 用 ExactScan 暴力搜索 —— 优雅降级保证可用性      |
| **ConnectionManager**    | SQLite 连接池 | LRU 风格缓存（实际保持所有连接打开） | Node.js 环境下更好的复用性，WAL 模式支持并发读写                                       |
| **EmbeddingService**     | 文本向量化    | Transformers.js 本地 / 远程 API      | 支持本地 CPU 推理，也支持外部 API，灵活适配部署环境                                    |
| **AutoCapture**          | 自动捕获记忆  | 触发式 LLM 摘要生成                  | 利用 `session.idle` 事件在空闲时异步处理，不阻塞主交互                                 |
| **UserProfileManager**   | 用户画像      | 置信度衰减算法                       | 随时间推移降低旧偏好置信度，保持画像动态更新                                           |
| **DeduplicationService** | 去重          | 两遍去重（精确+近邻）                | 先去 exact match，再通过余弦相似度去近重复，控制存储效率                               |

**整体架构思想：** 分层模块化 + 容错降级设计。每个核心服务都导出单例，顶层通过 `LocalMemoryClient` 统一入口。

---

## 2. SQLite 分片存储的具体实现逻辑

### 存储架构

```
storage/
├── metadata.db              # 元数据数据库：记录所有分片信息
├── users/                   # 用户级分片目录
│   ├── user_<hash>_shard_0.db
│   └── ...
└── projects/                # 项目级分片目录
    ├── project_<hash>_shard_0.db
    └── ...
```

### 核心数据结构（metadata.db 中的 shards 表）

```sql
CREATE TABLE IF NOT EXISTS shards (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  scope TEXT NOT NULL,           -- "user" 或 "project"
  scope_hash TEXT NOT NULL,      -- 作用域哈希（路径/用户ID哈希）
  shard_index INTEGER NOT NULL,  -- 分片序号
  db_path TEXT NOT NULL,         -- 相对路径
  vector_count INTEGER DEFAULT 0,-- 当前向量计数
  is_active INTEGER DEFAULT 1,   -- 1=当前可写，0=只读
  created_at INTEGER NOT NULL,
  UNIQUE(scope, scope_hash, shard_index)
);
```

### 创建分片流程

```typescript
// 来自 ShardManager.createShard
1. 根据 scope、scopeHash、shardIndex 生成路径: {scope}s/{scope}_{scopeHash}_shard_{shardIndex}.db
2. 在 metadata.db 的 shards 表插入记录，vector_count=0，is_active=1
3. 获取连接，初始化分片数据库的表结构：
   - shard_metadata: 存储嵌入维度、模型名称
   - memories: 存储实际记忆记录 + 向量 BLOB
   - 创建索引: container_tag, type, created_at, is_pinned
4. 返回 ShardInfo 对象
```

### 获取可写分片（getWriteShard）

```typescript
getWriteShard(scope, scopeHash): ShardInfo {
  1. 查询当前活跃分片：SELECT * FROM shards WHERE scope=? AND scope_hash=? AND is_active=1 ORDER BY shard_index DESC LIMIT 1
  2. 若无分片 → 创建新分片 (index=0)
  3. 校验分片：检查文件存在 + memories 表存在
  4. 若无效 → 删除元数据，关闭连接，创建同序号新分片（容错）
  5. 若当前活跃分片向量数 >= CONFIG.maxVectorsPerShard (默认 10000):
     - 标记当前分片为只读 (is_active=0)
     - 创建新分片 (shard_index + 1)
  6. 返回分片信息
}
```

**关键点：** 只有**最后一个分片是可写的**，所有之前的分片都标记为只读。这样写入始终只追加到一个文件。

### 查询分片流程

```typescript
// 跨分片搜索:
1. 根据 scope + scopeHash 获取该分组下所有分片 (getAllShards)
2. 对每个分片并行执行 searchInShard
3. 合并所有结果，按相似度排序，截断到 limit
4. 返回 top-K

// searchInShard 内部:
- 从连接池获取分片数据库连接
- 使用向量后端（USearch/ExactScan）搜索获取 top (limit*4) 结果
- 从 SQLite 加载完整记忆记录
- 计算最终相似度（内容向量相似度 * 0.6 + 标签相似度 * 0.4，支持关键词 boost）
- 返回该分片结果
```

---

## 3. 向量存储的二进制格式细节

### 存储格式

**Float32Array → SQLite BLOB 的转换：**

```typescript
// vector-search.ts 第 13-15 行
function toBlob(vector?: Float32Array): Uint8Array | null {
  return vector ? new Uint8Array(vector.buffer) : null;
}
```

这是关键：直接**零拷贝**转换。`Float32Array.buffer` 给你底层的 `ArrayBuffer`，然后包装成 `Uint8Array` 写入 SQLite 作为 BLOB。**没有 JSON 序列化，没有 base64 编码** —— 原始二进制直接存，空间效率和读取速度都是最优。

### 读取解码

```typescript
// usearch-backend.ts 第 220-228 行
private decodeVector(value: Uint8Array | ArrayBuffer | null | undefined): Float32Array {
  if (!value) return new Float32Array();
  if (value instanceof Uint8Array) {
    return new Float32Array(
      value.buffer.slice(value.byteOffset, value.byteOffset + value.byteLength)
    );
  }
  return new Float32Array(value);
}
```

读取时，从 SQLite BLOB 得到 `Uint8Array`，然后利用其 `.buffer` 直接构造 `Float32Array`，不需要额外拷贝。

### 每条记忆存储两个向量

```typescript
// 表结构
CREATE TABLE memories (
  ...
  vector BLOB NOT NULL,        -- 内容嵌入向量
  tags_vector BLOB,           -- 标签嵌入向量（可选）
  ...
);
```

搜索时，分别对内容和标签做向量检索，然后加权融合相似度：`contentSim * 0.6 + tagsSim * 0.4`。

### 空间计算

- 每个向量维度默认 1024（`Xenova/all-MiniLM-L6-v2` 输出）
- 每个浮点数 4 字节 → 单向量 4KB
- 双向量就是 8KB
- 10000 向量/分片 → 约 80MB 纯向量数据，加上文本内容，分片文件通常在 100-200MB，SQLite 处理起来很轻松。

---

## 4. USearch 和 ExactScan 两个后端的具体实现和切换逻辑

### 后端工厂与切换架构

```typescript
// backend-factory.ts
- FallbackAwareBackend 包装器: 任何操作失败自动切换到 fallback
- createVectorBackend() 根据配置和 probe 结果选择:
  - "exact-scan": 只使用 ExactScan
  - "usearch-first" / "usearch": 尝试加载 USearch，失败降级到 ExactScan
```

### FallbackAware 降级机制

```typescript
async search(args) {
  try {
    return await this.activeBackend.search(args);
  } catch (error) {
    this.logDegrade("search", error);
    this.activeBackend = this.fallback;  // 永久降级
    return this.fallback.search(args);
  }
}
```

**特点：** 一旦主后端出错，**永久切换**到 fallback，不会再尝试主后端，避免反复失败。

---

### USearchBackend 实现细节

**内存结构：**

```typescript
// 每个分片 + 每种类型(content/tags)对应一个独立索引
private readonly indexes = new Map<string, CachedIndex>();

interface CachedIndex {
  index: USearchIndex;              // USearch 原生索引对象
  idToKey: Map<string, bigint>;     // 内存中 string ID → USearch bigint key 映射
  keyToId: Map<bigint, string>;     // 反向映射
  nextKey: bigint;                  // 下一个可用 key
  indexKey: string;
  initialized: boolean;             // 是否已从 SQLite 重建
}
```

**索引键生成：**

```typescript
private getIndexKey(shard: ShardInfo, kind: VectorKind): string {
  return `${shard.scope}_${shard.scopeHash}_${shard.shardIndex}_${kind}`;
}
```

**重建机制 (rebuildFromShard):**

```typescript
async rebuildFromShard({ db, shard, kind }): Promise<void> {
  const indexKey = getIndexKey(...);
  if (existing?.initialized) return;  // 已初始化 → 跳过，缓存命中

  const column = kind === "tags" ? "tags_vector" : "vector";
  const rows = db.prepare(`SELECT id, ${column} FROM memories WHERE ${column} IS NOT NULL`).all();

  const cache = createEmptyIndex(indexKey);
  for each row:
    decode vector from BLOB
    upsert into USearch index
  cache.initialized = true;
}
```

**关键点：** USearch 索引只存在于**内存**，每次启动需要从 SQLite 重新构建。这是设计决策：USearch 文件格式不稳定，不如直接从 SQLite 主存储重建可靠。

---

### ExactScanBackend 实现细节

ExactScan 是**无状态暴力扫描**，不需要维护内存索引：

```typescript
async search(args): Promise<BackendSearchResult[]> {
  1. 从 SQLite SELECT id, vector FROM memories WHERE vector IS NOT NULL
  2. 对每一行解码得到 Float32Array
  3. 计算与查询向量的余弦相似度（1 - distance）
  4. 按 distance 排序，取前 limit
  5. 返回结果
}
```

**优点：**

- 零内存开销
- 永远与 SQLite 数据一致，不需要重建
- 永远不会失效，作为 fallback 非常可靠

**缺点：**

- 每次搜索都扫描全表，O(N) 复杂度
- 分片大小控制在 1 万向量以内，对于大多数用例可接受

---

### 切换逻辑总结

| 场景                         | 行为                                   |
| ---------------------------- | -------------------------------------- |
| USearch 编译失败 / 无法导入  | 启动时直接用 ExactScan                 |
| USearch 初始化成功但搜索报错 | 第一次失败触发永久降级到 ExactScan     |
| 查询时跨多个分片             | 每个分片独立处理，各自使用已降级的后端 |

---

## 5. 自动捕获的完整工作流

### 触发时机

在 OpenCode 插件中，监听 `session.idle` 事件（用户停止活动后 10 秒超时触发）：

```typescript
// 插件入口监听
ctx.on("session.idle", (event) => {
  if (CONFIG.autoCaptureEnabled) {
    performAutoCapture(ctx, event.sessionID, event.directory);
  }
});
```

### 完整工作流

```
performAutoCapture()
├─ 1. 防并发锁：isCaptureRunning 检查
├─ 2. 获取未捕获的最新提示：userPromptManager.getLastUncapturedPrompt(sessionID)
├─ 3. 争用处理：claimPrompt() → 只有一个赢得竞争
├─ 4. 从 OpenCode API 获取完整会话消息：ctx.client.session.messages({ id: sessionID })
├─ 5. 定位用户提示位置，截取从提示之后的所有 AI 消息
├─ 6. 提取 AI 内容：
│   ├─ textResponses: 收集所有文本响应
│   └─ toolCalls: 收集工具调用，截断长输入到 100 字符
├─ 7. 若没有内容 → 跳过返回
├─ 8. 获取项目标签（从目录路径提取）
├─ 9. 获取最近一条记忆上下文（用于连贯摘要）
├─ 10. 构建 Markdown 上下文：Previous Memory → User Request → AI Response → Tools Used
├─ 11. 调用 LLM 生成结构化摘要（generateSummary）
├─ 12. 如果 type="skip" → 不保存（非技术内容）
├─ 13. 否则 → 调用 memoryClient.addMemory() 存入向量数据库
├─ 14. 标记提示已捕获，关联记忆 ID
└─ 15. 显示成功 Toast 通知（可选）
```

### LLM 调用方式

支持两种调用路径：

**路径 1：使用当前 OpenCode 已连接的 provider（推荐）**

```typescript
if (CONFIG.opencodeProvider && CONFIG.opencodeModel) {
  - 使用 OpenCode 内置的 generateStructuredOutput
  - Zod schema 约束：{ summary: string, type: string, tags: string[] }
  - 系统提示指定规则：只捕获技术内容，SKIP 非技术，生成 2-4 个标签
  - 返回解析后的结构化结果
}
```

**路径 2：使用外部配置的 LLM API**

```typescript
else if (CONFIG.memoryModel && CONFIG.memoryApiUrl) {
  - 通过 AIProviderFactory 创建 provider（支持 OpenAI, Anthropic, Gemini 等）
  - 使用 function calling 输出结构化结果
  - toolSchema 定义参数与上面相同
}
```

**提示语设计要点：**

- 要求用目标语言输出（检测用户提示语言或配置强制）
- 强制格式：`## Request` + `## Outcome`
- 明确过滤规则：SKIP 问候/闲聊，CAPTURE 代码修改/bug修复/功能添加/决策

---

## 6. 用户画像系统中置信度衰减的具体算法实现

### 数据结构

```typescript
// 偏好结构
interface Preference {
  category: string; // 例如：coding-style, language, tooling
  description: string; // 具体描述
  confidence: number; // 0.0 - 1.0 置信度
  lastUpdated: number; // 最后更新时间戳（毫秒）
  evidence: string[]; // 证据来源
}
```

### 衰减算法（applyConfidenceDecay）

```typescript
// user-profile-manager.ts 第 221-231 行
const decayThreshold = CONFIG.userProfileConfidenceDecayDays * 24 * 60 * 60 * 1000;
// 默认 decayThreshold = 30 天

profileData.preferences = profileData.preferences
  .map((pref) => {
    const age = now - pref.lastUpdated;
    if (age > decayThreshold) {
      const decayFactor = Math.max(0.5, 1 - (age - decayThreshold) / decayThreshold);
      return { ...pref, confidence: pref.confidence * decayFactor };
    }
    return pref;
  })
  .filter((pref) => pref.confidence >= 0.3); // 置信度低于 0.3 直接删除
```

### 衰减公式详解

设：

- `age` = 当前时间戳 - 最后更新时间戳
- `threshold` = 衰减阈值（N 毫秒，默认 30 天）

当 `age > threshold`：

```
decayFactor = max(0.5, 1 - (age - threshold) / threshold)
新置信度 = 原置信度 × decayFactor
```

**解读：**

- 超过阈值后，每多经过一个阈值周期，衰减至少 50%（floor 0.5）
- 如果 age = 1.5 × threshold → (age - threshold)/threshold = 0.5 → decayFactor = 0.5
- 如果 age = 2 × threshold → decayFactor = 0.5（被 max 钳制）
- 也就是说，**最大衰减就是减半**，不会直接清零，保留历史信息
- 低于 0.3 被删除，清理低置信度噪声

### 增量更新策略

```typescript
// mergeProfileData 中合并偏好
当发现已存在同分类同描述偏好：
  - 置信度 +0.1（上限 cap 到 1.0）
  - 合并证据（保留最多 5 条）
  - 更新 lastUpdated = now
排序：按置信度降序
截断：保留最多 CONFIG.userProfileMaxPreferences（默认 50）
```

---

## 7. 去重服务的具体实现步骤

`DeduplicationService.detectAndRemoveDuplicates()` 采用**两遍去重策略**：

### 第一遍：精确去重

```typescript
// 按 (container_tag + 内容) 分组
const contentMap = new Map<string, any[]>();
for (const memory of memories) {
  const key = `${memory.container_tag}:${memory.content}`;
  contentMap.get(key)!.push(memory);
}

// 每组只保留最新的一个
for (const [, duplicates] of contentMap) {
  if (duplicates.length > 1) {
    duplicates.sort((a, b) => Number(b.created_at) - Number(a.created_at));
    const toDelete = duplicates.slice(1); // 删除所有旧的
    for (const dup of toDelete) {
      await vectorSearch.deleteVector(db, dup.id, shard);
      shardManager.decrementVectorCount(shard.id);
      exactDeleted++;
    }
  }
}
```

**设计要点：** 按容器标签（项目/用户）分组去重，不同容器的相同内容允许存在。排序后保留最新创建的。

---

### 第二遍：近重复检测

```typescript
// uniqueMemories = 每组精确去重后保留的那个
const processedIds = new Set<string>(); // 已标记为重复的跳过

for (i = 0; i < uniqueMemories.length; i++) {
  const mem1 = uniqueMemories[i];
  if (!mem1.vector || processedIds.has(mem1.id)) continue;

  const vector1 = new Float32Array(new Uint8Array(mem1.vector).buffer);
  const similarGroup = { representative: mem1, duplicates: [] };

  for (j = i + 1; j < uniqueMemories.length; j++) {
    const mem2 = uniqueMemories[j];
    if (!mem2.vector || processedIds.has(mem2.id)) continue;
    if (mem1.container_tag !== mem2.container_tag) continue; // 只在同容器内找近重复

    const vector2 = new Float32Array(new Uint8Array(mem2.vector).buffer);
    const similarity = cosineSimilarity(vector1, vector2);

    // 相似度 >= 阈值 且 < 1.0（排除精确重复，已经删了）
    if (similarity >= CONFIG.deduplicationSimilarityThreshold && similarity < 1.0) {
      similarGroup.duplicates.push({ id: mem2.id, content: mem2.content, similarity });
      processedIds.add(mem2.id); // 标记为已处理，避免重复分组
    }
  }

  if (similarGroup.duplicates.length > 0) {
    nearDuplicateGroups.push(similarGroup);
  }
}
```

**复杂度分析：** O(N²) 的两两比对，每个分片内部独立进行。因为分片最大 1 万向量，`1万² = 1亿` 比较在 Node.js 中还是能完成，但运行耗时较长。这是后台任务，不影响交互。

**阈值配置：** 默认 `deduplicationSimilarityThreshold = 0.95`，只有非常相似才会被检测。

**输出：** 返回精确删除数 + 近重复分组，由调用者决定是否删除（当前设计保留，供人工检查）。

---

## 8. 连接池管理 SQLite 连接的具体策略

### 连接池设计

```typescript
export class ConnectionManager {
  private connections: Map<string, typeof Database.prototype> = new Map();

  getConnection(dbPath: string): Database {
    if (this.connections.has(dbPath)) {
      return this.connections.get(dbPath)!; // 复用已有连接
    }
    // 否则新建连接
    const dir = dirname(dbPath);
    if (!existsSync(dir)) mkdirSync(dir, { recursive: true });
    const db = new Database(dbPath);
    this.initDatabase(db);
    this.connections.set(dbPath, db);
    return db;
  }

  closeConnection(dbPath: string): void {
    const db = this.connections.get(dbPath);
    if (db) {
      db.run("PRAGMA wal_checkpoint(TRUNCATE)");
      db.close();
      this.connections.delete(dbPath);
    }
  }
}
```

### 初始化 PRAGMA 配置

```typescript
private initDatabase(db: Database): void {
  db.run("PRAGMA busy_timeout = 5000");       // 锁等待超时 5 秒
  db.run("PRAGMA journal_mode = WAL");        // WAL 写日志模式，并发更好
  db.run("PRAGMA synchronous = NORMAL");      // 同步正常，性能较好
  db.run("PRAGMA cache_size = -64000");       // 64MB 页缓存（负值单位 KB）
  db.run("PRAGMA temp_store = MEMORY");       // 临时表放内存
  db.run("PRAGMA foreign_keys = ON");         // 开启外键约束
}
```

### 策略要点

1. **LRU 风格但实际是全缓存**：所有打开的连接一直保持打开，不主动淘汰。因为分片数量有限，每个分片一个连接，内存可控。

2. **WAL 模式检查点**：关闭连接时执行 `PRAGMA wal_checkpoint(TRUNCATE)`，把 WAL 日志合并回主文件并截断，控制日志大小。

3. **批量检查点**：`checkpointAll()` 对所有打开的连接执行被动检查点，用于定期清理。

4. **关闭时**：`closeAll()` 关闭所有连接，进程退出前调用。

### 优缺点

| 特点                                                                     | 说明 |
| ------------------------------------------------------------------------ | ---- |
| ✅ 优点：打开连接一次，永久复用，避免反复 open/close 开销                |      |
| ✅ 优点：WAL 允许读者和写者并发，不用自己加锁                            |      |
| ⚠️ 缺点：连接数 = 分片数，极端情况会有几十上百个连接，但 SQLite 处理得动 |      |

---

## 9. 项目整体目录结构和代码组织方式

```
opencode-mem/
├── src/
│   ├── index.ts                 # 主入口，导出所有公共 API
│   ├── config.ts                # 配置读取、默认值、类型定义（~600 行）
│   ├── plugin.ts                # OpenCode 插件入口
│   ├── types/
│   │   └── index.ts             # 全局类型定义
│   ├── services/
│   │   ├── api-handlers.ts      # HTTP API 端点（web 服务启用时）
│   │   ├── auto-capture.ts      # 自动捕获核心逻辑（~390 行）
│   │   ├── client.ts            # LocalMemoryClient 对外接口（~350 行）
│   │   ├── cleanup-service.ts   # 自动清理旧记忆服务
│   │   ├── deduplication-service.ts  # 去重服务（~160 行）
│   │   ├── embedding.ts         # 嵌入服务：本地 Transformers / 远程 API（~130 行）
│   │   ├── logger.ts            # 日志工具
│   │   ├── tags.ts              # 项目标签提取（从路径）
│   │   ├── sqlite/              # SQLite 核心层
│   │   │   ├── shard-manager.ts      # 分片管理（~330 行）
│   │   │   ├── vector-search.ts       # 向量检索协调（~370 行）
│   │   │   ├── connection-manager.ts  # 连接池管理（~85 行）
│   │   │   ├── sqlite-bootstrap.ts    # better-sqlite3 初始化
│   │   │   └── types.ts          # 类型定义
│   │   ├── vector-backends/     # 向量检索后端
│   │   │   ├── usearch-backend.ts  # USearch 实现（~240 行）
│   │   │   ├── exact-scan-backend.ts # ExactScan 暴力扫描（~120 行）
│   │   │   ├── backend-factory.ts    # 工厂 + 降级包装（~120 行）
│   │   │   └── types.ts
│   │   ├── ai/                  # LLM 提供者抽象
│   │   │   ├── ai-provider-factory.ts
│   │   │   ├── base-provider.ts
│   │   │   ├── providers/       # 各提供者实现：openai, anthropic, gemini
│   │   │   └── opencode-provider.ts  # 集成当前 OpenCode LLM
│   │   ├── user-profile/        # 用户画像系统
│   │   │   ├── user-profile-manager.ts # ~390 行，核心包含信衰减
│   │   │   ├── profile-analyzer.ts    # 利用 LLM 从对话提取画像
│   │   │   └── types.ts
│   │   └── ...
│   └── web/                     # Web 服务器代码
├── tests/                       # ~60+ 单元测试
└── package.json
```

**组织原则：**

- 按**职责分层**：存储层 → 向量后端 → 业务服务 → API/插件入口
- 每个核心服务都是**单例**，在模块底部导出单例实例
- 依赖注入少，大部分是直接导入单例，简化设计
- 接口和实现分离：`VectorBackend` 接口，多个实现

---

## 10. 关键技术决策和 trade-off

| 决策                                             | 理由                                                                                   | 权衡                                                                                      |
| ------------------------------------------------ | -------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------- |
| **SQLite 分片存储**                              | 单文件事务，每个分片控制在 1 万向量以内，避免单个巨大数据库导致查询变慢                | 元数据需要单独维护，查询需要遍历所有分片 → 但并发查询可以并行，可接受                     |
| **USearch 索引全内存**                           | USearch 索引是内存数据结构，访问速度最快；持久化格式兼容风险高，不如从 SQLite 重建可靠 | 启动需要重建所有索引，冷启动时间稍长 → 但内存足够时运行时最快                             |
| **Float32Array 直接存为 BLOB**                   | 零序列化开销，空间利用率 100%，读取最快                                                | 需要手工编解码，但是代码只有几行，代价可忽略                                              |
| **双层后端架构（USearch + ExactScan fallback）** | USearch 是原生模块，某些平台可能编译失败；降级保证插件总能工作                         | 额外一层包装，但 fallback 实现简单，代码量很小                                            |
| **自动捕获在 idle 触发**                         | 不阻塞用户编码流程，后台异步处理                                                       | 用户可能 10 秒内继续输入，当前这条就会被跳过 → 设计如此，因为只有最终稳定的会话才值得保存 |
| **置信度衰减而非直接删除**                       | 旧偏好不一定错，只是证据不新鲜；保留但降低权重，新证据可以重新提升置信度               | 需要额外存储置信度和时间戳 → 数据结构很小，复杂度低                                       |
| **两遍去重（精确 + 近重复）**                    | 精确去重 O(1) 哈希分组，高效；近重复 O(N²) 是后台任务，不影响性能                      | 近重复比对慢，只在手动触发或定期清理运行 → 接受                                           |
| **连接池保持所有连接打开**                       | SQLite 打开连接快，连接句柄占用内存少；反复打开关闭反而有开销                          | 连接数 = 分片数，最多几十个，内存可控                                                     |
| **分离内容向量和标签向量**                       | 标签单独嵌入可以提升标签匹配的权重；最终得分加权融合                                   | 需要存储两份向量，存储空间翻倍 → 但标签向量通常只占少数，可接受                           |
| **LLM 生成摘要再存储**                           | 压缩对话，去除冗余，只保存关键结论；检索时命中率更高                                   | 需要调用 LLM，消耗 token → idle 后台运行，增量处理，开销可接受                            |

---

## 关键数据结构总结

供源码导航参考：

| 文件                       | 行数 | 核心职责                       |
| -------------------------- | ---- | ------------------------------ |
| `shard-manager.ts`         | 329  | 分片创建、切换、查询元数据     |
| `vector-search.ts`         | 372  | 协调检索，结果融合，结果水化   |
| `connection-manager.ts`    | 85   | SQLite 连接池 + PRAGMA 初始化  |
| `usearch-backend.ts`       | 241  | 内存索引维护，搜索             |
| `exact-scan-backend.ts`    | 120  | 暴力扫描 fallback              |
| `backend-factory.ts`       | 120  | 工厂 + 降级包装器              |
| `auto-capture.ts`          | 389  | 触发→收集上下文→LLM摘要→存储   |
| `user-profile-manager.ts`  | 387  | 画像存储，置信度衰减，增量合并 |
| `deduplication-service.ts` | 162  | 两遍去重实现                   |
| `embedding.ts`             | 130  | 本地/远程嵌入，LRU 缓存        |
| `client.ts`                | 353  | 对外 API 入口                  |

---

## 总结

这个设计的核心思想是：**可靠性优先，渐进降级，不强迫用户在特定平台折腾原生依赖**。USearch 能用就快，不能用就降级到 ExactScan 照样能用，数据都在 SQLite 里不会丢。
