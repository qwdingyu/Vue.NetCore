# Node.js（Bun + Express + Knex）迁移落地路线与技术方案

## 1. 迁移目标与总体原则
- **保持核心能力对等**：现有 .NET 后端在 `Startup` 中集成了认证、会话、缓存、Swagger、Quartz、SignalR、静态资源与文件上传等中间件，需要在 Node 侧提供等效的中间件与服务，确保接口行为一致。【F:vol.api.sqlsugar/VOL.WebApi/Startup.cs†L51-L284】
- **复刻 ServiceBase 横切逻辑**：`ServiceBase<T, TRepository>` 将缓存、多租户、分页排序、动态 SQL 等能力统一封装，迁移时需在 Node 层抽象基类或装饰器以复现这些通用规则。【F:vol.api.sqlsugar/VOL.Core/BaseProvider/ServiceBase.cs†L29-L192】
- **保留代码生成生态**：`Sys_TableInfoService` 负责读取多库表结构、维护配置并生成后端/前端代码，是平台差异化能力，需规划在 Node 环境下的模板、数据访问与生成策略，可在 MVP 后补齐但设计阶段需预留接口。【F:vol.api.sqlsugar/VOL.Builder/Services/Core/Partial/Sys_TableInfoService.cs†L270-L420】【F:vol.api.sqlsugar/VOL.Builder/Services/Core/Partial/Sys_TableInfoService.cs†L661-L900】
- **渐进式切换**：通过阶段性并行运行与自动化比对，降低一次性替换风险，遵循“先最小可行、再迭代拓展”的原则。

## 2. 现有能力与 Node 技术映射
| 现有能力 | Node/Bun 推荐方案 | 迁移要点 |
| --- | --- | --- |
| ASP.NET Core 中间件：JWT、会话、Cors、Swagger、Quartz、SignalR、静态文件/Upload 目录、异常处理中间件。【F:vol.api.sqlsugar/VOL.WebApi/Startup.cs†L51-L284】 | 运行时使用 **Bun**，框架层采用 **Express 5**；JWT 选用 `jsonwebtoken`；会话用 `express-session` + Redis；跨域使用 `cors`；API 文档使用 `swagger-ui-express`；定时任务选用 `node-cron`/`agenda`；实时通信用 `socket.io`；静态文件与上传通过 `express.static` + `multer`；异常拦截用自定义中间件。 | 需验证 Express 在 Bun 下的兼容性并通过单测覆盖；文件上传限制与目录结构与 .NET 配置保持一致。 |
| ServiceBase 抽象：多租户过滤、缓存、分页排序、动态查询、仓储模式。【F:vol.api.sqlsugar/VOL.Core/BaseProvider/ServiceBase.cs†L29-L192】 | 使用 `knex` 封装基础 `Repository`，结合 `objection`-like 自研模型或直接封装 QueryBuilder；多租户策略使用中间件注入租户上下文 + 查询构造器；缓存采用 Redis（`ioredis`）；分页排序封装在服务基类；动态查询使用统一的筛选构建器。 | 先实现最常用的 CRUD、分页、排序，再补充多租户/动态 SQL，逐步替换各业务模块。 |
| 工作流容器：在启动时注册实体，异步加载流程配置并缓存。【F:vol.api.sqlsugar/VOL.Core/WorkFlow/WorkFlowContainer.cs†L18-L170】 | 使用 `agenda` 或 `bullmq` 执行流程加载任务，结合 `knex` 查询流程配置；以单例服务保留流程-表映射，并提供 Hooks 给业务服务调用。 | 初期保留查询流程配置的只读能力，审批流可在后续阶段迁移。 |
| 代码生成器：多数据库元数据读取、模板渲染、Vue/Vite/App 页面生成。【F:vol.api.sqlsugar/VOL.Builder/Services/Core/Partial/Sys_TableInfoService.cs†L270-L420】【F:vol.api.sqlsugar/VOL.Builder/Services/Core/Partial/Sys_TableInfoService.cs†L661-L900】 | MVP 阶段先迁移 API 基础模板（控制器/服务/仓储），使用 `handlebars` 模板引擎；元数据读取通过 `knex` + 各数据库系统表；Vue/App 模板延后，待后端稳定后补齐。 | 需要拆分模板与逻辑，避免巨石方法；注意多语言、主从表及 Vite 模式兼容。 |

## 3. 分阶段实施路线
### 3.1 阶段一：基线搭建（MVP 第 1 月）
1. **环境与脚手架**
   - 使用 Bun 初始化项目，建立 `apps/api` 结构，配置 `express`, `knex`, `dotenv`, `cors`, `jsonwebtoken`, `winston` 等依赖。
   - 设计统一配置管理（.env + Bun 原生导入），并提供运行/调试脚本。
2. **基础中间件对齐**
   - 实现全局错误处理中间件，对齐 .NET 中的 `ExceptionHandlerMiddleWare` 行为，格式化返回 `status`, `message`。
   - 配置跨域、静态文件目录与上传路径 `/Upload`，保持与现有 `UseStaticFiles`、上传目录创建逻辑一致。【F:vol.api.sqlsugar/VOL.WebApi/Startup.cs†L223-L284】
3. **数据库连接与基础仓储**
   - 搭建 `knex` 连接工厂，抽象多数据库配置；提供基础 CRUD 仓储，覆盖分页、排序与软删除，为后续业务提供共用基类。
4. **认证授权 MVP**
   - 基于 `jsonwebtoken` 实现登录签发与中间件校验，先覆盖最常用的 `Bearer` 认证流程，对齐 `Startup` 中的 JWT 验证逻辑。【F:vol.api.sqlsugar/VOL.WebApi/Startup.cs†L74-L158】

### 3.2 阶段二：服务层与系统能力（第 2~3 月）
1. **ServiceBase 能力迁移**
   - 设计 `BaseService` & `BaseRepository`，补齐多租户过滤、动态排序（参考 `GetPageDataSort`）、分页查询封装等能力。【F:vol.api.sqlsugar/VOL.Core/BaseProvider/ServiceBase.cs†L90-L180】
   - 引入 Redis 缓存服务，封装类似 `CacheContext` 接口，支持热点数据与权限缓存。
2. **权限体系与用户管理**
   - 迁移角色、菜单、数据权限接口；抽象 RBAC 中间件，保证菜单/按钮权限判定逻辑可配置。
3. **作业调度与后台任务**
   - 使用 `agenda` 替代 Quartz，重新实现作业注册、依赖注入与生命周期管理，映射 `HttpResultfulJob` 等后台任务注入方式。【F:vol.api.sqlsugar/VOL.WebApi/Startup.cs†L174-L284】
4. **实时通信**
   - 以 `socket.io` 实现消息推送，兼容现有 `/message` Hub 路由与 CORS 校验逻辑，确保前端无需改动即可接入。【F:vol.api.sqlsugar/VOL.WebApi/Startup.cs†L270-L281】
5. **工作流与审批**
   - 重建流程配置加载器，沿用 `WorkFlowContainer` 的实体注册与筛选字段概念，并提供 API 查询流程状态。【F:vol.api.sqlsugar/VOL.Core/WorkFlow/WorkFlowContainer.cs†L45-L170】

### 3.3 阶段三：代码生成器与高级特性（第 4 月起）
1. **模板拆分与渲染引擎**
   - 提取现有模板资源，转换为 Handlebars/EJS 模板；根据 `CreateEntityModel`、`CreateVuePage` 的流程拆解生成步骤，避免单个方法承担多职责。【F:vol.api.sqlsugar/VOL.Builder/Services/Core/Partial/Sys_TableInfoService.cs†L270-L420】【F:vol.api.sqlsugar/VOL.Builder/Services/Core/Partial/Sys_TableInfoService.cs†L661-L900】
2. **元数据适配层**
   - 基于 `knex` 的 `information_schema` 查询构建统一接口，支持 SqlServer/MySQL/PgSQL，与 `GetCurrentSql` 等方法对应。【F:vol.api.sqlsugar/VOL.Builder/Services/Core/Partial/Sys_TableInfoService.cs†L351-L399】
3. **增量发布策略**
   - 先生成后端 API & 仓储模板，确保 Node 服务可消费；再补充 Vue/Vite/App 模板，最后实现配置管理与同步逻辑。
4. **自动化验证**
   - 构建生成结果的快照测试，对比 .NET 版本输出，保证命名、目录与文件内容一致。

### 3.4 并行验证与切换策略
- **双跑阶段**：在 Bun 服务上线初期，与原 .NET 后端并行运行，通过代理或网关进行灰度切换，确保关键接口响应一致。
- **回归测试**：为迁移的每个模块编写 Postman/Newman 套件，覆盖 CRUD、权限、上传、实时消息等关键路径。
- **部署规划**：使用 PM2 或 Bun 的原生进程守护，结合 Docker/Windows 服务部署；监控层引入 Prometheus + Grafana 或 APM（Elastic、Jaeger）完成链路追踪。

## 4. 功能优先级建议
1. **最高优先**：认证登录、基础 CRUD、权限校验、日志与异常处理，保障核心业务可用。
2. **高优先**：多租户、缓存、调度、实时通信、流程查询，支撑平台差异化能力。
3. **中优先**：代码生成器的后端模板、Vue/Vite 主流程；复杂模板、App 生成与高级流程后置。
4. **低优先**：PDF、信号推送拓展、边缘业务模块，待主线稳定后迭代。

## 5. 风险与缓解
- **兼容性风险**：Bun 对部分 Node 模块兼容度需验证，关键依赖（Express、Knex、socket.io）必须通过集成测试。
- **多数据库差异**：`knex` 在不同数据库方言下的分页与类型映射存在差异，需要建立统一适配层并保留回退策略。【F:vol.api.sqlsugar/VOL.Builder/Services/Core/Partial/Sys_TableInfoService.cs†L292-L371】
- **流程与生成器复杂度**：工作流、代码生成涉及大量历史配置，应编写迁移脚本同步老数据，并在双跑期使用对比工具验证输出。【F:vol.api.sqlsugar/VOL.Core/WorkFlow/WorkFlowContainer.cs†L70-L170】【F:vol.api.sqlsugar/VOL.Builder/Services/Core/Partial/Sys_TableInfoService.cs†L661-L900】

通过以上步骤，可在确保核心功能平滑迁移的同时，为后续的代码生成器与高级能力保留演进空间，最终实现从 .NET SqlSugar 架构到 Bun + Express + Knex 技术栈的全面转换。
