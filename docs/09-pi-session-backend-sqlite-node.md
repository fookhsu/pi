# pi-session-backend-sqlite-node — SQLite 会话后端

`packages/session-backends/sqlite-node`，包名 `@earendil-works/pi-session-backend-sqlite-node`。为 `pi-agent-core` 会话提供 Node `node:sqlite` 持久化。

**为什么独立成包**：核心包不默认引入运行时内建模块（`node:sqlite`）与原生 SQLite 依赖。后端接受运行时专属的 SQLite 工厂（`SqliteDatabaseFactory`），未来其他会话后端（bun:sqlite 等）可作为独立包发布。

## 组成

- **`node:sqlite` 适配器**（`src/index.ts`）：实现 `SqliteDatabase` / `SqliteStatement` / `SqliteRunResult` 接口，把 `DatabaseSync` 的语句参数（命名/位置参数）与结果规范化。依赖通过工厂注入，核心代码不直接 import `node:sqlite`。
- **SQLite 会话仓库**（`src/sqlite/repo.ts`）：`SqliteSessionRepository`，实现 agent-core 的会话接口（`create` / `appendMessage` / 查询等）。懒持有单个共享数据库连接。
- **迁移**（`src/sqlite/migrations.ts` + `migrations/`）：schema 版本化迁移。
- **物化视图**：派生数据的物化视图，避免热路径重算。
- **FTS 搜索**（`src/sqlite/search-backend.ts`）：可选全文搜索。`createSqliteSessionSearch(options)` 返回独立搜索服务，基于同一规范数据库：
  - 仓库不暴露 `search()`；搜索是独立服务。
  - FTS 表与触发器在首次非空搜索时懒创建；首次创建时对规范条目做一次性重建；之后 SQLite 触发器让 FTS 与条目 insert/delete/payload 更新保持同步。
- **分支缓存**（`src/sqlite/branch-cache.ts`）：会话分支查询的缓存。
- **存储抽象**（`src/sqlite/storage/`）：存储层。

## 用法

```ts
await using repository = new SqliteSessionRepository(options);
const search = createSqliteSessionSearch(options);
const session = await repository.create({ cwd });
await session.appendMessage(message);

const hits = [];
for await (const hit of search.search("needle")) hits.push(hit);
```
