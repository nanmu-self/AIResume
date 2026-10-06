# AIResume 重构方案 · v2

> **技术栈已确定**：React 19 + NestJS 11 + PostgreSQL（Prisma 8 contract 工作流）
> **基础设施**：`nest_monorepo` 脚手架（pnpm workspace + TS Project References + Tailwind 4 + shadcn 风格 UI）
> **落地方式**：用 `create-nest-monorepo` 生成新项目 → 新增独立前台 app `apps/web`
> **源项目**：`AIResume`（Vue 3 纯前端，27 个功能点，localStorage 存储）

---

## 一、为什么要重新生成而不是直接改

`nest_monorepo` 是一个**可发布的脚手架模板**（`create-nest-monorepo`，`private: false`，有 `bin` 入口和 `template.manifest.json`）。往里面塞业务代码会让模板失去通用性。

正确姿势：

```bash
# 1. 从脚手架生成新项目
node C:/Users/Administrator/Documents/Git/nest_monorepo/scripts/create/bin.mjs C:/Users/Administrator/Documents/Git/AIResume-v2

# 2. 删掉示例模块 users / storage 页，替换为业务模块
# 3. 按本方案逐层实现
```

---

## 二、现有功能点全清单（迁移台账）

### 2.1 简历编辑域

| 模块 | 源文件 | 功能点 | 迁移难度 |
|---|---|---|---|
| 个人信息 | `personalInfo.vue` | 姓名/性别/电话/邮箱/学校/政治面貌/网站/头像/专业/年龄/求职岗位 | 低 |
| 教育经历 | `education.vue` | 学校/学位/专业/起止时间 + CRUD | 低 |
| 项目经历 | `project.vue` | 项目名/角色/时间/简介/描述，支持 Markdown | 低 |
| 工作经历 | `workExperience.vue` | 公司/职位/起止时间/描述，按 `\n` 分割渲染 | 低 |
| 专业技能 | `skill.vue` | 技能条目列表，支持 Markdown/HTML | 低 |
| 荣誉奖项 | `honor.vue` | 奖项名/日期/描述 | 低 |
| 自我评价 | `selfEvaluation.vue` | 长文本 | 低 |
| 模块拖拽排序 | `resumeEdit.vue` | 7 模块拖拽，含前后位置判定 + 末尾投放区 | 中 |

**统一 CRUD 机制**：源项目用 `addItem/deleteItem/updateItem` 泛型方法 + `currentId` 自增。新方案改为服务端事务 + `sort_order` 字段。

### 2.2 模板与渲染域

| 功能点 | 源实现 | 迁移要点 |
|---|---|---|
| 模板市场 | `template/index.vue` | 卡片列表 + 预览图懒加载 + 作者链接 |
| 动态模板加载 | `import.meta.glob` 构建期扫描 | **改为运行时加载，这是最大改动点** |
| 模板元数据 | `templates.json` + `config.json` 双份 | **双份必须合并为单一数据源** |
| 主题色阶 | `colorUtils.ts` chroma-js LAB 6 级 | 移植为 TS 工具，或改用 oklch 色阶 |
| 版式参数 | 弹窗调节 6 项 | 存 `resume_settings` JSON 字段 |
| 预览缩放 | 滚轮 0.2-3 倍，鼠标中心锚点 | 逻辑可整体移植 |
| 预览拖拽 | 拖动 + 10% 可见性边界 | 逻辑可整体移植 |
| PDF 导出 | html2pdf.js 离屏二次挂载 | 见 §6.4，服务端渲染更稳 |

### 2.3 AI 能力域

| 功能点 | Prompt / 机制 | 迁移要点 |
|---|---|---|
| AI 润色 | `我现在求职的{position}岗位。{原文}` | Prompt 下沉服务端 |
| 扩展建议 | 同接口切换变体 | 同上 |
| 手动采纳 | 输出进编辑器，用户编辑后应用 | 前端交互，保留 |
| 经历深挖 | 多轮对话，AI 主导，要求 ` ```resume ``` ` 包裹 | **改 tool call，去正则** |
| 模拟面试 | 多轮追问，面试官角色 | Prompt 下沉 |
| 面试评分 | 正则提取 ` ```分数``` ` | **改 tool call** |
| 经历选择器 | 从 projects / workExperience 生成下拉 | 保留 |
| JD 输入 | provide/inject 共享 | React 用 Context 或 Zustand |
| 自动回填 | 正则提取 resume 块写回 | **改 tool call** |
| 新手引导 | `a-tour` 四步漫游 | 换 Radix/shadcn 实现 |

**技术链路现状**：`WorkerPool(4 线程)` → `Cloudflare Worker` → 阿里百炼 `qwen-turbo`

### 2.4 数据与配置域

| 功能点 | 源存储 | 迁移目标 |
|---|---|---|
| 简历数据 | `localStorage.resumeData` | PostgreSQL `resumes` + `resume_items` |
| ID 计数器 | `localStorage.currentId` | 移除，ID 由数据库生成 |
| 首次访问标记 | `localStorage.isFirstVisit` | 移除，新用户由 `users.id` 判断 |
| 主题偏好 | `localStorage.theme` | cookie `admin-theme`（脚手架已有） |
| API 配置 | pinia-persist 明文 | **移除，密钥只存服务端 env** |
| JSON 导入导出 | Blob 下载 / FileReader | 改为服务端接口 |
| 清空数据 | 重置为模板 | 保留 |
| 预览填充 | `fetch('/resumeData.json')` | 改为 seed 脚本或示例简历接口 |

### 2.5 未实现（README 已标注）

- AI 面试官语音通话
- 可视化简历设计（拖拽布局）
- 简历字段隐藏（数据保留但不展示）

---

## 三、源项目的 6 个结构性问题

这些是重构要顺手修掉的，不是照搬。

### 3.1 SSE 解析会丢字（`worker/aiWorker.ts:38`）

```ts
const lines = chunk.split('\n').filter(line => line.trim() !== '');
```

三个叠加问题：

1. **跨 chunk 半包**：一个 `data:` 行被网络切包时，后半段 JSON.parse 失败进 catch，**字符被静默丢弃**
2. **未过滤非 data 行**：`event:` / `id:` / 空行等直接进 JSON.parse
3. **帧分隔错误**：SSE 帧以 `\n\n` 分隔，只按 `\n` 切会拼错

**新方案**：后端直接透传厂商 SSE 流（`@Sse()` + rxjs），或用 `@nestjs/axios` 的 `responseType: 'stream'`。前端用 `EventSource` / `fetch` + `ReadableStream`，**不再手写解析**。

### 3.2 WorkerPool 的 onmessage 被赋值两次

`workerPool.ts:14` 注册的回调里 `this.queue.find()` 永远返回 `undefined`——任务在 `processQueue()` 第 62 行已被 `shift()` 移出队列。

**新方案**：整个 `worker/` 目录删除。`fetch` + SSE 解析本身开销极小，Web Worker 没有存在必要。

### 3.3 模板系统构建期硬编码

```ts
const templateModules = import.meta.glob('../../../template/**/index.vue')
```

- 新增模板必须改源码目录 → 触发重新构建
- `templates.json`（运行时 fetch）与 glob 扫描结果**可能不一致**
- `new URL(..., import.meta.url)` 缩略图是构建期资源
- `config.json` **从未被读取**，纯装饰

**这正是同类项目"构建产物缺失"的病根** —— 产物正确性依赖构建期文件扫描。

**新方案**：见 §5。

### 3.4 模板没有 props 契约

模板组件内部直接 `useResumeStore()` 拿全局状态。后果：`exportToPDF` 里 `createApp(Component)` 挂载时**没装 pinia**，靠单例侥幸能跑。

**新方案**：模板渲染器为**纯函数组件**，只接受 `{ data, theme, settings }` props，无全局依赖。

### 3.5 API Key 明文落盘

`useSettingsStore` 开启 `persist: true`，`aliApiKey` 序列化进 localStorage。

**新方案**：密钥只进 `apps/server/.env`，经 `@/config/env` 校验（脚手架已有此模式）。

### 3.6 数据模型缺失

- `currentId` 是条目计数器，不是简历 ID → **无法支持多份简历**
- 全量 `JSON.stringify` 写 localStorage → 每次按键重写全量
- 无 version → 字段演进靠浅合并，删字段留脏数据

**新方案**：见 §4。

---

## 四、数据模型设计

遵循脚手架的 **Prisma 8 contract 工作流**：模型写在 `apps/server/prisma/contract.prisma`，顶部 `// use prisma-next`，改完执行 `pnpm db:emit`。

> **注意**：PSL 8 无 `@updatedAt`，`updated_at` 用 `@default(now())`，更新语义由应用层写入。
> 原生类型写在类型位置（`VarChar(255)` 而非 `@db.VarChar`）。`VarChar(N)` 在 TS 侧是 branded string，写入处需 `as Varchar<N>` 断言。

### 4.1 契约草案

```prisma
// use prisma-next

/// 简历主记录（对应源项目 localStorage.resumeData 的整体）
model Resume {
  id             Int      @id @default(autoincrement())
  user_id        Int
  /// 简历标题，支持用户建多份（如"前端-阿里"、"前端-腾讯"）
  title          VarChar(128)
  /// 当前模板 slug（对应 templates.slug）
  template_slug  VarChar(64)
  /// 简历设置 JSON：字体大小/间距/页边距/主题色
  settings       Json
  /// 自我评价（单值字段，单独提出便于全文检索）
  summary        Text
  created_at     Timestamptz(6) @default(now())
  updated_at     Timestamptz(6) @default(now())

  @@map("resumes")
}

/// 简历条目（教育/项目/工作/技能/荣誉统一存这张表）
model ResumeItem {
  id          Int      @id @default(autoincrement())
  resume_id   Int
  /// 所属区块：education | projects | workExperience | skills | honors
  section     VarChar(32)
  /// 排序位，拖拽时重排
  sort_order  Int
  /// 各 section 的差异化字段
  payload     Json
  created_at  Timestamptz(6) @default(now())
  updated_at  Timestamptz(6) @default(now())

  @@map("resume_items")
}

/// 区块显示顺序（personalInfo / education / ... 的拖拽结果）
model ResumeSectionOrder {
  id          Int      @id @default(autoincrement())
  resume_id   Int
  section_key VarChar(32)
  sort_order  Int

  @@map("resume_section_orders")
}

/// 模板元数据（前台市场 + 后台管理共用）
model Template {
  id          Int      @id @default(autoincrement())
  slug        VarChar(64) @unique
  name        VarChar(128)
  description Text
  author      VarChar(64)
  homepage    VarChar(255)
  /// 布局 schema JSON（见 §5）
  schema      Json
  /// 预览图 URL（七牛）
  preview_url VarChar(512)
  /// draft | published | archived
  status      VarChar(16) @default("draft")
  /// 语义化版本，模板迭代用
  version     VarChar(16) @default("1.0.0")
  created_at  Timestamptz(6) @default(now())
  updated_at  Timestamptz(6) @default(now())

  @@map("templates")
}

/// AI 会话（模拟面试/经历深挖可续聊）
model AiSession {
  id          Int      @id @default(autoincrement())
  user_id     Int
  resume_id   Int?
  /// polish | deepen | interview
  mode        VarChar(16)
  /// 岗位 JD
  jd          Text
  /// 消息历史 [{ role, content }]
  messages    Json
  /// 面试评分（仅interview 模式）
  score       Int?
  /// 评估报告
  report      Text?
  created_at  Timestamptz(6) @default(now())
  updated_at  Timestamptz(6) @default(now())

  @@map("ai_sessions")
}

/// token 计量与配额
model AiUsage {
  id         Int      @id @default(autoincrement())
  user_id    Int
  session_id Int?
  model      VarChar(64)
  tokens_in  Int
  tokens_out Int
  created_at Timestamptz(6) @default(now())

  @@map("ai_usages")
}
```

### 4.2 为什么不沿用源项目的 `currentId` 自增

源项目 `currentId` 是**条目自增计数器**（`useResumeStore.ts` 的 `initializeCurrentId` 取所有 id 最大值 +1）。这个设计有三个问题：

1. 无法表达"多份简历"—— ID 全局唯一但简历只有一份
2. 删除后 ID 不复用，长跑会漂移
3. 前端生成 ID 无法保证并发安全

新方案：ID 由 PostgreSQL 自增列生成，`resume_id` 作为外键隔离多份简历。

### 4.3 迁移脚本要点

源数据在 `localStorage.resumeData`，结构已知（`ResumeState`）。迁移脚本做三件事：

1. `settings` 字段对应 `resumeSetting`，注意源项目用了 `String` 大写类型（`currentTemplate: String`），转换时需 `String(x)`
2. `sectionOrder` 拆入 `ResumeSectionOrder` 表
3. `education` / `projects` / `workExperience` / `skills` / `honors` 五个数组统一塞进 `ResumeItem`，`payload` 保留原字段（`id` 丢弃，数据库重新生成）

---

## 五、模板系统重构方案

这是整个重构**最需要做对决策**的地方。

### 5.1 源项目模板机制的问题

```
templates.json（手工维护的元数据）
config.json（从未被读取的元数据副本）
import.meta.glob（构建期扫目录）
src/template/*/index.vue（每个模板复制粘贴 resumeSetting 读取逻辑）
```

新增模板要改 3 个地方，且 `config.json` 和 `templates.json` 会漂移。

### 5.2 三种落地路径

| 方案 | 做法 | 优点 | 代价 |
|---|---|---|---|
| **A. JSON 驱动** | 模板存为布局schema JSON，通用渲染器解释执行 | 前后端同构；服务端渲 PDF 复用同 schema；模板可运行时上下线 | 表达力受限，复杂布局要设计好 schema |
| **B. 沙箱渲染** | 模板仍是代码，存七牛，运行时拉取 + iframe 隔离 | 表达力完整，现有 4 模板零改动迁移 | 需处理 postMessage + CSP；无法服务端渲染 PDF |
| **C. 混合** | 主流模板走 JSON schema；开放投稿走沙箱 | 兼顾效率与生态 | 两套渲染器，维护成本翻倍 |

**建议 A 起步，schema 设计时预留 B 的迁移口。**

理由：README 明确提到"欢迎前端开发者共创模板"，但那是**老项目**的诉求。新项目若走 JSON schema，共创门槛反而更高。若确定要开放投稿，直接选 B。

### 5.3 JSON schema 草案（方案 A）

```typescript
// packages/shared/src/contracts/template.ts

/** 区块布局配置 */
interface TemplateBlock {
  /** 区块标识，对应 ResumeSectionKey */
  section: 'personalInfo' | 'education' | 'projects' | 'workExperience' | 'skills' | 'honors' | 'summary'
  /** 布局槽位（12 栅格） */
  span: 1 | 2
  /** 渲染变体 */
  variant: string       // 'default' | 'timeline' | 'two-column' | 'compact'
  /** 是否显示 */
  visible: boolean
  /** 区块内条目排布 */
  layout?: 'list' | 'grid' | 'inline'
}

/** 模板完整定义 */
interface TemplateSchema {
  version: string
  /** 页面尺寸 */
  page: { width: number; height: number; padding: [number, number, number, number] }
  /** 栅格列数 */
  columns: 1 | 2
  /** 色板（语义色名，真实色值由用户主题色派生） */
  palette: { primary: string; text: string; muted: string; border: string; background: string }
  /** 区块列表（渲染顺序 = 数组顺序，可被用户拖拽覆盖） */
  blocks: TemplateBlock[]
  /** 字体 */
  typography: { family: string; baseSize: number; lineHeight: number }
}
```

**关键设计点**：

- `schema` 只描述**布局与样式**，不含数据。数据由渲染时注入
- `blocks` 数组顺序是默认顺序，用户拖拽后存 `ResumeSectionOrder` 覆盖
- `palette` 存**语义色名**，真实色值从用户 `settings.themeColor1` 经色阶算法派生（保留源项目 `colorUtils.ts` 的思路，但改用 oklch 体系以对齐脚手架的 `@app/config-tailwind` token）

### 5.4 通用渲染器设计

```tsx
// packages/resume-kit/src/TemplateRenderer.tsx

interface TemplateRendererProps {
  schema: TemplateSchema
  resume: ResumeData
  settings: ResumeSettings
}

/**
 * 纯函数组件——不依赖任何全局 store / context。
 * 这样服务端渲染 PDF 时可以复用同一组件。
 */
export function TemplateRenderer({ schema, resume, settings }: TemplateRendererProps) {
  const shades = useMemo(() => deriveColorScale(settings.themeColor1), [settings.themeColor1])
  const order = resume.sectionOrder ?? schema.blocks.map(b => b.section)

  return (
    <div style={{ ... }}>
      {order.map(key => {
        const block = schema.blocks.find(b => b.section === key)
        if (!block?.visible) return null
        return <BlockRenderer key={key} block={block} resume={resume} settings={settings} shades={shades} />
      })}
    </div>
  )
}
```

对比源项目：模板从"每个一个 Vue 文件 + 各自读全局 store"变成"一个渲染器 + 一份 JSON"。

---

## 六、目标架构

### 6.1 整体结构

```
AIResume-v2/
├─ apps/
│  ├─ server/                   NestJS 11（API + SSR 渲染）
│  │  └─ src/
│  │     ├─ resumes/            简历 CRUD
│  │     ├─ templates/          模板查询（前台）+ 管理（后台）
│  │     ├─ ai/                 AI 编排 + SSE 流式
│  │     ├─ export/             服务端 PDF 渲染
│  │     ├─ auth/               脚手架已有
│  │     └─ qiniu/              对象存储（脚手架已有）
│  ├─ web/                      【新增】用户前台
│  │  └─ src/
│  │     ├─ features/
│  │     │  ├─ editor/          简历编辑器
│  │     │  ├─ preview/         模板渲染容器
│  │     │  ├─ ai-chat/         AI 会话
│  │     │  └─ template-store/  模板市场
│  │     └─ ...
│  └─ web-admin/                运营后台（脚手架已有，加模板管理页）
├─ packages/
│  ├─ shared/                   契约 + 类型（前后端共用）
│  ├─ config-tailwind/          设计 token（脚手架已有）
│  ├─ ui/                       shadcn 组件（脚手架已有）
│  └─ resume-kit/               【新增】模板渲染器 + 色阶工具
└─ ...
```

**新增 `packages/resume-kit` 的理由**：模板渲染器要被 `apps/web`（预览）和 `apps/server`（PDF 导出）同时使用，必须放共享包。这是脚手架现有分包模式（`shared` / `ui` / `config-tailwind`）的自然延伸。

### 6.2 数据流

```
apps/web（用户前台）
  │
  ├─ 编辑数据 ──PATCH /api/v1/resumes/:id/sections/:key──> resumes 模块 ──> PG
  │
  ├─ AI 对话 ──POST /api/v1/ai/chat (SSE)──> ai 模块
  │                ├─ 注入 system prompt（服务端）
  │                ├─ 注入简历上下文（服务端读 PG）
  │                ├─ tool call 强制结构化输出
  │                └─ 大模型网关（多厂商 + 重试 + 计量）
  │                     └─>阿里百炼 / DeepSeek / OpenAI 兼容
  │
  └─ PDF 导出 ──POST /api/v1/export/pdf──> export 模块
                       └─ Playwright 渲染 TemplateRenderer（同构）
                            └─> 七牛存储，返回临时 URL
```

**关键变化**：前端永不接触任何大模型密钥。

### 6.3 前端状态管理

源项目是 Pinia + localStorage。React 侧建议：

| 关注点 | 方案 |
|---|---|
| 服务端数据 | **Alova**（脚手架已装，`useWatcher` 防竞态） |
| 简历编辑草稿 | `useReducer` + Context，或轻量 Zustand |
| UI 状态 | React 内置 state / Context |
| 主题 | 复用脚手架 `ThemeProvider`（cookie 持久化） |

**不要引入 Redux Toolkit** —— 脚手架已用 Alova，再叠一层 Redux 会造成双数据源。

草稿策略：本地 `useReducer` 实时响应输入，**防抖 800ms 后 PATCH** 到服务端。切页面时强制 flush。这比源项目"每次按键全量写 localStorage"好得多。

### 6.4 PDF 导出方案对比

| 方案 | 做法 | 优点 | 代价 |
|---|---|---|---|
| **前端 html2pdf** | 移植源项目逻辑 | 零迁移成本 | 字体不一致、版式不稳定、大文档卡顿 |
| **服务端 Playwright** | `apps/server` 渲染同一 React 组件出 PDF | 字体一致、可加水印、可后台排队 | 需常驻容器（约 500MB 内存）|
| **混合** | 前端优先，失败降级服务端 | 体验与稳定兼顾 | 两套代码路径 |

**建议**：先移植前端 html2pdf 快速打通，同构渲染器稳定后切服务端。因为渲染器是纯函数组件，两种方式能共用。

---

## 七、后端模块设计

### 7.1 新增模块清单（遵循 AGENTS.md 规范）

按脚手架 `users/` 的形状，每个模块三件套 + dto：

```
apps/server/src/
├─ resumes/
│  ├─ resumes.module.ts
│  ├─ resumes.controller.ts# CRUD + 排序 + 导入导出
│  ├─ resumes.service.ts
│  └─ dto/
│     ├─ create-resume.dto.ts
│     ├─ update-resume.dto.ts
│     ├─ update-section.dto.ts
│     └─ reorder-sections.dto.ts
├─ templates/
│  ├─ templates.module.ts
│  ├─ templates.controller.ts       # 前台查询（公开）
│  ├─ templates-admin.controller.ts # 后台管理（JWT + 角色）
│  ├─ templates.service.ts
│  └─ dto/
├─ ai/
│  ├─ ai.module.ts
│  ├─ ai.controller.ts              # @Sse() 流式
│  ├─ ai.service.ts                 # 编排：prompt 注入 + 上下文组装
│  ├─ ai-providers/
│  │  ├─ provider.interface.ts
│  │  ├─ qwen.provider.ts
│  │  ├─ deepseek.provider.ts
│  │  └─ openai-compatible.provider.ts
│  └─ dto/
└─ export/
    ├─ export.module.ts
    ├─ export.controller.ts
    └─ export.service.ts
```

### 7.2 契约文件（packages/shared）

```
packages/shared/src/contracts/
├─ resume.ts          # ResumeDTO / ResumeItemDTO / ResumeSettingsDTO
├─ template.ts        # TemplateSchema / TemplateDTO
├─ ai.ts              # AiSessionDTO / AiMessage / AiMode
└─ index.ts           # 统一 re-export
```

错误码按脚手架分段约定新增：

```typescript
// error-codes.ts 新增段
// 20300–20399 简历
RESUME_NOT_FOUND: 20300,
RESUME_TITLE_DUPLICATED: 20301,
RESUME_SECTION_INVALID: 20302,
// 20400–20499 模板
TEMPLATE_NOT_FOUND: 20400,
TEMPLATE_NOT_PUBLISHED: 20401,
// 20500–20599 AI
AI_PROVIDER_UNAVAILABLE: 20500,
AI_UPSTREAM_ERROR: 20501,
AI_QUOTA_EXCEEDED: 20502,
AI_SESSION_NOT_FOUND: 20503,
```

### 7.3 DTO 注意事项（脚手架强约定）

AGENTS.md 明确了两条，写代码时必须遵守：

1. **DTO 必须是 class**，interface 会被编译期擦除导致 `ValidationPipe` 静默失效
2. **Query DTO 的 number 字段必须 `@Type(() => Number)`**，`ValidationPipe` 未开 `enableImplicitConversion`，query 参数默认是 string

```typescript
// ✅ 正确
export class QueryResumesDto {
  @IsOptional() @IsNumber() @Type(() => Number)
  page?: number

  @IsOptional() @IsString()
  search?: string
}
```

### 7.4 Prisma 8 查询写法（与 Prisma 7 差异大）

```typescript
// ✅ 本项目写法（PSL8 Collection 链式）
const items = await this.prisma.orm.ResumeItem
  .select('id', 'section', 'payload')
  .where((r) => r.resume_id.eq(resumeId))
  .orderBy((r) => r.sort_order, 'asc')
  .all()

const one = await this.prisma.orm.Resume.where((r) => r.id.eq(id)).first()

// ❌ 不要写 Prisma 7 风格
// await this.prisma.resumeItem.findMany({ where: { resumeId } })
```

**count 聚合限制**：顶层集合无 `count()` 终端，需要计数时取主键列表后计长（脚手架 `users.service.ts` 已示范）。

### 7.5 SSE 实现

源项目的 SSE 解析 bug 一并修掉。后端用 Nest 原生 `@Sse()`：

```typescript
@Sse('chat')
chat(@Body() dto: ChatDto, @Req() req: Request): Observable<MessageEvent> {
  return this.aiService.streamChat(req.user.id, dto).pipe(
    map((chunk) => ({ data: chunk })),
  )
}
```

前端用 `EventSource` 或 `fetch` + `ReadableStream`，浏览器端不再手写 `data:` 行解析。

**注意**：Vite dev proxy 需支持 SSE（关闭缓冲），生产用 Nginx 时同样要配 `proxy_buffering off`。

---

## 八、认证与权限

脚手架已有 JWT + 本地策略，需要扩展的是**角色与资源归属**。

### 8.1 User 模型扩展

当前 `contract.prisma` 的 User 只有 `id / password_hash / email / created_at / updated_at`，**没有角色字段**。需要加：

```prisma
model User {
  // ... 现有字段
  /// 角色：user 普通用户 / admin 运营
  role VarChar(16) @default("user")
}
```

改完执行 `pnpm db:emit` → `pnpm migrate:create`。

### 8.2 资源归属校验

每个涉及用户数据的查询**必须带 `user_id` 过滤**：

```typescript
const resume = await this.prisma.orm.Resume
  .where((r) => r.id.eq(id).and(r.user_id.eq(currentUserId)))  // ← 关键
  .first()

if (!resume) throw new BizException(ErrorCode.RESUME_NOT_FOUND)
```

源项目是纯前端，没有这层。迁移时这是**最容易漏的地方** —— 所有 service 方法都要检查。

### 8.3 接口权限划分

| 接口 | 权限 |
|---|---|
| `GET /api/v1/templates` | 公开（只返 status=published） |
| `GET /api/v1/resumes` | JWT + 仅本人 |
| `POST /api/v1/ai/chat` | JWT + 配额校验 |
| `POST /api/v1/export/pdf` | JWT + 限流 |
| `POST /api/v1/admin/templates/*` | JWT + role=admin |

**注意**：脚手架的 `JwtAuthGuard` 已全局可用，但 `admin` 角色判定需新增 `RolesGuard` + `@Roles()` 装饰器。

---

## 九、分阶段实施路线

### 阶段 0：生成项目 + 清理示例

- [ ] `node nest_monorepo/scripts/create/bin.mjs <目标目录>`
- [ ] 删 `apps/web-admin/src/pages/users`、`storage` 示例页（AIResume 不需要用户管理后台）
- [ ] `apps/server` 删 `users` 模块（保留 auth，User 模型作为简历归属主体）
- [ ] 新建 `apps/web`（从 web-admin 复制 vite/tailwind/oxlint 配置）
- [ ] 更新根 `tsconfig.json` references + `pnpm-workspace.yaml`
- [ ] 验证 `pnpm typecheck` 通过

**产出**：空业务骨架，`apps/{server,web,web-admin}` 三端跑通。

### 阶段 1：共享契约 + 数据模型

- [ ] `packages/shared/src/contracts/{resume,template,ai}.ts`
- [ ] `packages/shared/src/constants/error-codes.ts` 新增 20300-20599 段
- [ ] `packages/resume-kit` 新建包（渲染器 + 色阶工具）
- [ ] `contract.prisma` 加 Resume / ResumeItem / ResumeSectionOrder / Template 模型
- [ ] `pnpm db:emit` → `pnpm migrate:create`
- [ ] User 模型加 `role` 字段

**产出**：类型链路从数据库贯通到前端，`pnpm typecheck` 能验证契约一致性。

### 阶段 2：简历服务 + 前台编辑器

- [ ] `apps/server/src/resumes/` 模块（CRUD + 排序 + 导入导出）
- [ ] 归属校验：所有查询带 `user_id`
- [ ] `apps/web` 简历编辑器（7 个区块 + 拖拽排序）
- [ ] 防抖自动保存（800ms PATCH 单 section）
- [ ] localStorage 迁移脚本（一次性导入工具）

**产出**：完整可用的简历编辑器，数据落 PG。这是第一个可交付里程碑。

### 阶段 3：模板系统

- [ ] 设计并冻结 `TemplateSchema`
- [ ] `packages/resume-kit` 实现 `TemplateRenderer`（纯函数组件）
- [ ] 迁移 4 个现有模板（A/B/C/D）为 schema JSON
- [ ] 色阶工具移植（chroma LAB → oklch 或保留 chroma）
- [ ] 版式参数面板（6 项滑块）
- [ ] 预览容器：缩放 + 拖拽逻辑移植
- [ ] `apps/web` 模板市场页
- [ ] `apps/web-admin` 模板管理页

**产出**：模板可运行时上下线，不再触发重新构建。

### 阶段 4：AI 服务化

- [ ] `apps/server/src/ai/` 模块 + provider 抽象
- [ ] Qwen provider（对齐现有 `qwen-turbo` 配置）
- [ ] `@Sse()` 流式转发，删掉前端 Worker 链路
- [ ] Prompt 下沉服务端，前端只传业务参数
- [ ] **tool call 强制结构化输出**，废弃 ` ```resume ``` ` 正则约定
- [ ] `ai_sessions` 持久化，支持多端续聊
- [ ] token 计量 + 用户配额
- [ ] `apps/web` AI 会话页（经历深挖 / 模拟面试 / 评分报告）

**产出**：AI 能力全部服务端化，前端零密钥。

### 阶段 5：导出与增强

- [ ] 前端 html2pdf 移植（快速打通）
- [ ] 服务端 Playwright 渲染（同构切换）
- [ ] 七牛存 PDF + 临时 URL（复用脚手架 qiniu 模块）
- [ ] 补齐 README 待实现项：字段隐藏、可视化布局调整

---

## 十、源项目 Bug 速查

重构时顺手修掉，原位置记录备查：

| 源文件 | 行号 | 问题 |
|---|---|---|
| `src/worker/aiWorker.ts` | 38 | SSE 跨 chunk 半包导致丢字 |
| `src/worker/aiWorker.ts` | 39 | 按 `\n` 切分未过滤非 `data:` 行 |
| `src/worker/workerPool.ts` | 14 / 62 | `onmessage` 双赋值，首次回调失效 |
| `src/views/resume/components/resumePreview.vue` | 126 | `templates.json` 与 glob 结果可能不一致 |
| `src/views/resume/components/resumePreview.vue` | ~200 | `createApp(Component)` 未装 pinia |
| `src/store/useSettingsStore.ts` | — | `persist: true` 导致 API Key 明文落盘 |
| `src/template/*/config.json` | — | 从未被读取，与 `templates.json` 双份漂移 |
| `src/store/useResumeStore.ts` | — | `currentId` 语义错误，无法支持多份简历 |

---

## 附录 A：源项目文件清单

```
src/
├─ api/qwenAPI.ts                AI 调用入口（将废弃）
├─ worker/
│  ├─ aiWorker.ts                 SSE 解析（有 bug，将废弃）
│  └─ workerPool.ts               线程池（有 bug，将废弃）
├─ store/
│  ├─ useResumeStore.ts           简历数据 + CRUD
│  └─ useSettingsStore.ts         主题 + AI 配置（将废弃）
├─ views/
│  ├─ resume/                     编辑器
│  │  ├─ index.vue                主容器
│  │  ├─ components/
│  │  │  ├─ resumeEdit.vue        拖拽排序容器
│  │  │  ├─ resumePreview.vue     预览 + 设置 + PDF（最复杂，~400 行）
│  │  │  ├─ AIEnhancePopover.vue  AI 润色
│  │  │  ├─ personalInfo.vue / education.vue / project.vue
│  │  │  └─ workExperience.vue / skill.vue / honor.vue / selfEvaluation.vue
│  │  └─ styles/styles.css
│  ├─ aiDeep/
│  │  ├─ index.vue                深挖容器 + 漫游引导
│  │  ├─ components/AIChat.vue    AI 对话核心（含评分逻辑）
│  │  └─ components/userInput.vue JD + 经历选择器
│  ├─ template/index.vue          模板市场
│  ├─ setting/index.vue           配置页（将废弃）
│  ├─ resumeDesign/index.vue占位页
│  └─ coding.vue                  占位页（未挂路由）
├─ template/                      4 个模板 + dev 脚手架
├─ utils/colorUtils.ts            色阶生成
├─ utils/getTemplates.ts          模板列表拉取
├─ utils/reorder.ts               排序工具
├─ types/                         类型定义
└─ constants/sectionOrder.ts      默认区块顺序

worker.js                         Cloudflare Worker 反代（将废弃）
public/templates.json             模板元数据（唯一生效源）
public/resumeData.json            预览填充数据
```

## 附录 B：关键命令速查

```bash
# 生成新项目
node nest_monorepo/scripts/create/bin.mjs <目标目录>

# 契约变更流程（脚手架约定）
cd apps/server && pnpm db:emit      # 生成 contract.json + contract.d.ts（需提交）
pnpm migrate:create                 # 生成迁移
pnpm migrate:apply                  # 应用迁移

# 开发
pnpm dev                # 并行启动所有包
pnpm typecheck                      # TS Project References 增量检查
pnpm format                        # Prettier

# 迁移 localStorage 数据
# 浏览器控制台导出 resumeData → 走 POST /api/v1/resumes/import
```

---

**文档版本**：v2
**上次更新**：2026-10-06
**变更**：技术栈确认为 React 19 + NestJS 11 + PG（Prisma 8 contract）；基础设施改用 nest_monorepo脚手架；新增独立前台 app `apps/web`；新增 `packages/resume-kit` 共享渲染器包；补充 Prisma 8 查询写法与 DTO 强约定。