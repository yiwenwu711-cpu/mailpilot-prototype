# MailPilot · Onboarding 配置流程 · Codex 工程提示词

> 本文档把高保真原型 `index.html` 中的 **4 个配置流程页**（创建账户 / 连接资源 / Agent 试跑 / 启用上线）拆解成可交付的工程提示词。每节都是自包含的：路由、数据模型、组件结构、文案、校验、状态、验收标准齐全，可直接交给 Codex / Claude Code 落地。
>
> 视觉参考：根 `index.html`（界面 6 - 界面 9）
> 在线预览：https://mailpilot-prototype.vercel.app

---

## 0. 共享上下文（所有页面通用）

### 0.1 技术栈假设
- **框架**：Next.js 14（App Router）+ TypeScript
- **样式**：Tailwind CSS + shadcn/ui
- **表单**：React Hook Form + Zod
- **数据**：TanStack Query；后端约定 `/api/v1/*` REST 风格
- **状态**：Zustand 管理 onboarding 跨步流转（持久化到 localStorage + 后端 draft）
- **图标**：lucide-react

### 0.2 设计令牌（Design Tokens）
```ts
// tailwind.config.ts 关键 token
colors: {
  brand:   { 50:"#eef2ff", 500:"#4f46e5", 600:"#4338ca", 900:"#1e1b4b" }, // Indigo
  cyan:    { 500:"#22d3ee" },                                              // 渐变副色
  success: { 50:"#ecfdf5", 500:"#10b981", 700:"#047857" },
  warn:    { 50:"#fffbeb", 500:"#f59e0b", 700:"#b45309" },
  danger:  { 50:"#fef2f2", 500:"#ef4444", 700:"#b91c1c" },
  ink:     { 900:"#0f172a", 700:"#374151", 500:"#6b7280", 300:"#9ca3af" },
  surface: { 0:"#ffffff", 50:"#f7f8fc", 100:"#f3f4f8", border:"#eef0f4" },
}
fontFamily: { sans: ["-apple-system","Inter","PingFang SC","sans-serif"] }
radius: { card:"14px", btn:"8px", chip:"999px" }
```
主品牌渐变：`linear-gradient(135deg, #4f46e5 0%, #22d3ee 100%)`。

### 0.3 共用布局：`<OnboardingShell />`
所有 4 个页面共享同一个壳：
```
┌──────────┬─────────────────────────────────────┐
│ Sidebar  │ Main                                │
│ (220px)  │  ┌ Header ───────────────────────┐  │
│  Logo    │  │ Title + subtitle             │  │
│  Steps   │  └──────────────────────────────┘  │
│  (4 项)  │  ┌ Stepper (1—2—3—4) ────────────┐  │
│          │  └──────────────────────────────┘  │
│  User    │  ┌ Page Slot ───────────────────┐  │
│  (底部)  │  │  各步骤独立内容               │  │
│          │  └──────────────────────────────┘  │
│          │  ┌ Footer ───────────────────────┐  │
│          │  │ ← Prev   hint   Next →        │  │
│          │  └──────────────────────────────┘  │
└──────────┴─────────────────────────────────────┘
```
- 高度：`min-h-screen`，整页背景 `bg-surface-50`
- Sidebar 深色 `bg-slate-900`，4 个步骤项带状态：`done`（绿✓） / `active`（紫高亮） / `pending`（灰○）
- Stepper 用 4 个圆点 + 连线，状态与 Sidebar 同步
- Footer 固定底部，包含「上一步 / 提示文案 / 下一步」三段

### 0.4 共用数据模型
```ts
type OnboardingState = {
  step: 1 | 2 | 3 | 4;
  account: {
    method: "google" | "email";
    email: string;
    googleSub?: string;       // Google OAuth sub
    company: string;
    role: "ic" | "lead" | "founder" | "marketing";
    teamSize: "1-5" | "5-20" | "20-100" | "100+";
    leadSources: ("fb"|"google_ads"|"linkedin"|"website"|"offline")[];
  };
  resources: {
    inboxEmail: string;       // Gmail OAuth 后回填
    inboxAuthorized: boolean;
    senderEmail?: string;     // 可选
    sameInboxAsSender: boolean;
    websiteUrl: string;
    learningUrls: string[];   // textarea 一行一个
    playbooks: { id: string; filename: string; scenarioCount: number }[];
    meetingDuration: 15 | 30 | 45;
    escalationKeywords: string[];
    maxAiTurns: 3 | 5 | 8 | -1;
  };
  dryRun: {
    samples: Array<{
      id: string;
      from: { name: string; company: string; avatarHue: number };
      excerpt: string;
      draftHtml: string;
      status: "pending" | "approved" | "edited" | "regenerated";
    }>;
    approvedCount: number;
    styleMatchScore?: number; // 0-100，全确认后计算
  };
  launchRules: {
    workingHours: { days: number[]; start: string; end: string; tz: string };
    dailySendLimit: number;
    afterHours: "queue" | "send_now";
    escalationKeywords: string[]; // 与 resources 同源，可独立修改
  };
};
```

### 0.5 共用 API 约定
| Method | Path | 用途 |
|---|---|---|
| `GET` | `/api/v1/onboarding` | 拉取当前用户的 draft 状态 |
| `PATCH` | `/api/v1/onboarding` | 部分更新（每完成一步即调用） |
| `POST` | `/api/v1/onboarding/complete` | 步骤 4 启用 |
| `POST` | `/api/v1/oauth/google/init` | 返回 Google OAuth 跳转 URL |
| `POST` | `/api/v1/playbooks` | multipart 上传话术文件 |
| `POST` | `/api/v1/dry-run/generate` | 生成 3 封试跑草稿 |
| `POST` | `/api/v1/dry-run/regenerate` | 重新生成单封草稿 |

### 0.6 路由约定
```
/onboarding/signup     → 步骤 1
/onboarding/connect    → 步骤 2
/onboarding/dry-run    → 步骤 3
/onboarding/launch     → 步骤 4
/onboarding            → 自动跳转到当前未完成的步骤
```
未登录用户只能访问 `/onboarding/signup`；其余步骤通过 middleware 校验 `account.email` 存在。

### 0.7 全局验收标准（每个页面都要满足）
- ✅ 桌面 ≥ 1280px 完整布局，760 - 1280px 折叠为单栏
- ✅ < 768px 手机端 sidebar 改为顶部抽屉，Stepper 横向滚动
- ✅ 进入 / 离开页面自动保存 draft（防止刷新丢数据）
- ✅ 所有按钮 keyboard accessible（`Tab` 顺序 + `Enter` 触发）
- ✅ 加载态用 skeleton（不要 spinner 占满屏）
- ✅ 错误态用 inline toast（顶部），不打断填表
- ✅ 国际化：所有文案走 `t('onboarding.step1.title')`，i18n key 见各节末尾

---

## 步骤 1 · 创建账户

### 路由
`/onboarding/signup`

### 用户目标
3 分钟内：①完成身份认证（Google 或 邮箱）②录入团队基础画像（用于初始化 Agent 话术风格）

### 页面状态机
```
initial ─────────────────► filling ─────► submitting ─────► success → 跳 /onboarding/connect
                              │                  │
                              └─ validation_err ◄┘
                              └─ google_oauth ──► google_callback ──► filling (邮箱字段预填、disabled)
```

### Layout 结构
```tsx
<OnboardingShell activeStep={1}>
  <PageHeader
    title="👋 欢迎使用 MailPilot · 先建一个工作账户"
    subtitle="账户用于绑定话术、客户、会议数据，后续可邀请同事加入同一团队"
  />
  <Stepper current={1} />

  {/* 主体 2 列 1.1fr / 1fr */}
  <div className="grid grid-cols-[1.1fr_1fr] gap-5 max-w-[920px] mx-auto">
    <SignupMethodCard />   {/* ① 登录方式 */}
    <TeamProfileCard />    {/* ② 团队信息 */}
  </div>

  <OnboardingFooter
    hideBack
    hint="🔐 工作邮箱仅用于登录，不会用作 Agent 发件箱"
    nextLabel="创建账户并进入第二步 →"
    onNext={submit}
    nextDisabled={!form.formState.isValid}
  />
</OnboardingShell>
```

### 组件细节

#### `<SignupMethodCard />`
- 卡片标题：「① 登录方式」
- **Google 按钮**：白底 + 1px border + 居中 logo + 文字「使用 Google 账户一键注册」
  - 点击触发 `POST /api/v1/oauth/google/init`，跳转到返回的 url
  - OAuth 回调成功后回到本页，预填 `email` 字段为 disabled
- **分割线**：「— 或邮箱注册 —」用居中文字 + 两侧 1px 灰线
- **邮箱字段**：
  - label 「工作邮箱」
  - 占位 `your@company.com`
  - Zod 校验：`z.string().email("请输入有效的邮箱地址")`
  - 不允许 gmail.com / qq.com / 163.com 等公共邮箱后缀，校验失败提示「请使用公司邮箱注册，避免后续团队权限管理混乱」
- **密码字段**：
  - label 「设置密码」
  - 校验：`z.string().min(10).regex(/[A-Z]/).regex(/[0-9]/)`（10位 + 大写 + 数字）
  - 失焦显示强度条（弱/中/强）
- **同意条款 Checkbox**：默认未勾选，必须勾选才能提交。文字「我已阅读并同意《服务条款》《隐私政策》」中的两个引号链接打开 modal
- **底部提示框**：浅紫底，文字「💡 推荐使用 Google 登录，下一步连接 Gmail 时可省去授权」

#### `<TeamProfileCard />`
- 卡片标题：「② 团队信息（用于话术风格初始化）」
- **公司名称**：text input，必填，max 80
- **你的角色**：select，4 个选项
  - `ic` 销售（个人贡献者）
  - `lead` 销售主管 / RevOps
  - `founder` 创始人 / CEO
  - `marketing` 市场负责人
- **团队规模**：select，4 档 `1-5`、`5-20`（默认推荐）、`20-100`、`100+`
- **主要客户来源**：多选 Chip 组件
  - 5 个选项：`fb`(Facebook 表单)、`google_ads`(Google Ads)、`linkedin`(LinkedIn)、`website`(官网表单)、`offline`(展会 / 线下)
  - 选中态：紫底白字 + 前缀 ✓
  - 未选中：白底灰字 1px 边
  - 默认选中 `fb` + `google_ads`（这是产品主要场景）
  - 至少必须选 1 个，否则禁用「下一步」

### 提交流程
```ts
async function submit(values: SignupForm) {
  // 1. 邮箱注册路径
  if (values.method === "email") {
    await api.post("/auth/signup", values);  // 创建用户
  }
  // 2. 更新 onboarding draft
  await api.patch("/onboarding", { step: 1, account: values });
  // 3. 跳转
  router.push("/onboarding/connect");
}
```

### 状态 & 边界
- 已登录用户访问本页：直接跳转下一步
- Google OAuth 失败：顶部红色 toast「Google 授权失败，请重试或使用邮箱注册」
- 邮箱已被注册：邮箱字段下方红字「该邮箱已注册，是否去 [登录页](/login)？」
- 提交中：「创建账户」按钮显示 spinner + 禁用

### i18n keys
```
onboarding.step1.title
onboarding.step1.subtitle
onboarding.step1.method.title
onboarding.step1.method.google
onboarding.step1.method.divider
onboarding.step1.field.email.label / .placeholder / .error.public
onboarding.step1.field.password.label / .error.weak
onboarding.step1.field.terms
onboarding.step1.hint.google
onboarding.step1.profile.title
onboarding.step1.profile.role.{ic,lead,founder,marketing}
onboarding.step1.profile.size.{1-5,5-20,20-100,100+}
onboarding.step1.profile.source.{fb,google_ads,linkedin,website,offline}
onboarding.step1.cta
```

### 验收标准
- [ ] Google 按钮跳转 OAuth 并能正确回填
- [ ] 公共邮箱后缀被拦截
- [ ] 客户来源未选时下一步禁用
- [ ] 离开页面再回来字段已保留（draft）
- [ ] 移动端两个卡片堆叠为单栏，按钮宽度 100%
- [ ] Lighthouse a11y ≥ 95

---

## 步骤 2 · 连接资源

### 路由
`/onboarding/connect`

### 用户目标
连接 Gmail、提供官网链接、（可选）上传话术手册——这是 Agent 工作的「燃料」。

### Layout 结构
```tsx
<OnboardingShell activeStep={2}>
  <PageHeader
    title="🔌 让我们花 3 分钟连接 Agent 所需的资源"
    subtitle="连接你的 Google 邮箱、官网和话术手册，Agent 就能开始自动跟进 Facebook 表单线索"
  />
  <Stepper current={2} />

  {/* 主体 2 列：邮箱 + 官网；下方一行话术（占满） */}
  <div className="grid grid-cols-2 gap-5 max-w-[920px] mx-auto">
    <GmailConnectCard />
    <WebsiteCard />
    <PlaybookCard className="col-span-2" />
  </div>

  <OnboardingFooter
    backHref="/onboarding/signup"
    hint="🔒 所有数据加密存储 · 你可随时在「Agent 行为」中调整"
    nextLabel="下一步：让 Agent 试跑 3 封邮件 →"
  />
</OnboardingShell>
```

### 组件细节

#### `<GmailConnectCard />`
- 标题「① Google 邮箱」
- **收件邮箱**（必填）：
  - 默认显示「+ 连接收件邮箱（接收 FB 表单提交）」白底按钮
  - 点击 → Gmail OAuth（scope: `gmail.readonly` + `gmail.send`）
  - 授权成功后变为绿底胶囊：`✓ leads@acme.io · 已通过 OAuth 授权`，旁边带「断开」灰链接
- **复选框**：「一收一发使用同一个邮箱」（默认勾选）
- **发件邮箱**（条件显示）：取消上面复选框后才出现，独立 Gmail OAuth
- **检测结果提示框**（绿色）：授权成功后异步调用 `/api/v1/onboarding/scan-inbox` 拉取统计，3 秒后显示「✓ 已检测到 **52 封** 来自 Facebook 表单的未处理邮件，将在启用后开始跟进」
  - 加载中用 skeleton bar
  - 0 封时显示橙色「ℹ 未检测到 FB 表单邮件，Agent 仍可处理后续新邮件」

#### `<WebsiteCard />`
- 标题「② 官网链接」
- **公司官网 URL**：text input，必填，自动补 `https://` 前缀
- **重点学习页面**：textarea，一行一个 URL，可选
  - 占位包含 3 个示例（pricing、integrations、customers）
  - 校验：每行必须是 https URL，否则该行红色高亮
- **提示框**（紫色）：「🧠 抓取后将自动总结出 **产品定位 / 定价 / 集成清单 / 客户案例** 知识库」
- 失焦时触发 `POST /api/v1/knowledge/preview` 在右侧显示抓取预览（可选增强项）

#### `<PlaybookCard />`（占满两列宽）
- 标题「③ 话术手册」+ 灰色副标「（非必备 · 不上传则使用通用销售话术）」
- 内部再分 2 列：
  **左列（上传区）**：
  - 虚线框 Dropzone（react-dropzone），文案「⬆ 上传 PDF / DOCX / Markdown」+ 副标「支持多文件 · 单文件最大 20MB」
  - 上传中显示进度条
  - 上传完成显示文件列表：图标 + 文件名（粗体）+ 右侧绿字「已解析 · N 个场景」
  - 解析在后端做（LLM 提取场景），前端轮询 `/api/v1/playbooks/{id}/status`，状态机：`uploading → parsing → ready` 或 `failed`
  - 失败显示红色「解析失败」+ 重试按钮
  **右列（话术参数）**：
  - **会议时长偏好**：select，3 档 `15 / 30（默认） / 45 分钟`
  - **转人工触发关键词**：text input，逗号分隔，默认 `enterprise SLA, on-prem, legal, refund, complaint`，下方灰字「客户邮件命中任一关键词，Agent 立即停手并通知你」
  - **AI 单线程最大自动往返轮次**：select，4 档 `3 / 5（推荐） / 8 / 不限制`

### 状态 & 边界
- 用户进入本页前必须完成步骤 1
- Gmail OAuth 失败：toast「Gmail 授权失败：{reason}」
- 没上传话术：下一步仍可点击，使用兜底通用话术
- 没填官网：下一步禁用，红字「官网链接必填——Agent 用它了解你的产品」

### i18n keys（节选）
```
onboarding.step2.title / .subtitle
onboarding.step2.gmail.title / .receive.label / .connect / .connected
onboarding.step2.gmail.same_check
onboarding.step2.gmail.detected   ("已检测到 {count} 封…")
onboarding.step2.website.title / .url.label / .focus.label
onboarding.step2.website.hint
onboarding.step2.playbook.title / .optional / .upload / .parsing
onboarding.step2.playbook.duration / .escalation / .max_turns
```

### 验收标准
- [ ] Gmail OAuth 闭环，刷新页面授权状态保留
- [ ] 上传 PDF 解析后能展示场景数
- [ ] 关键词输入支持中英文逗号自动归一
- [ ] 邮件扫描结果加载态用 skeleton
- [ ] 移动端 3 张卡片堆叠

---

## 步骤 3 · Agent 试跑

### 路由
`/onboarding/dry-run`

### 用户目标
在 Agent 真正上线前，让用户**亲眼看到** AI 写的草稿水平，建立信任。明确这一步**不会真的发邮件**。

### 页面状态机
```
loading_samples ──► reviewing ──► (all approved) ──► next_enabled
                       │
                       └─ edit_draft / regenerate ──► back to reviewing
```
进入页面立刻调 `POST /api/v1/dry-run/generate`，后端从用户收件箱中真实未处理邮件里挑 3 封（按行业匹配度排序），每封都跑一次 Agent 生成草稿。

### Layout 结构
```tsx
<OnboardingShell activeStep={3}>
  <PageHeader
    title="🧪 Agent 已学会你的产品 · 用 3 封真实邮件验证一下"
    subtitle="这一步只生成草稿，不会真正发出去。你可以确认风格 / 内容 / 长度是否合适"
  />
  <Stepper current={3} />

  <KnowledgeBaseBanner />        {/* 紫青渐变 banner，展示学会了什么 */}

  <div className="max-w-[1080px] mx-auto">
    <ProgressBar approved={2} total={3} />   {/* 顶部进度条 */}

    {samples.map((s, i) => (
      <DryRunCard
        key={s.id}
        sample={s}
        isCurrent={i === firstPendingIdx}
        onApprove={...}
        onEdit={...}
        onRegenerate={...}
      />
    ))}
  </div>

  <OnboardingFooter
    backHref="/onboarding/connect"
    hint="⏱ Agent 平均生成一封草稿耗时 1.8 秒"
    nextLabel="下一步：启用上线 →"
    nextDisabled={approvedCount < 3}
  />
</OnboardingShell>
```

### 组件细节

#### `<KnowledgeBaseBanner />`
- 紫青渐变 `from-brand-50 to-cyan-50`，圆角 12px，左 1px 紫边
- 左侧大 🧠 图标
- 中间内容：
  - 第一行粗体「Agent 已建立知识库」
  - 第二行紫色小字 `acme.io · 4 个核心页面 · 12 项常见问题 · 5 个集成方案 · 30 个话术场景已解析`
  - 数字从 `/api/v1/knowledge/stats` 拉取
- 右侧「查看知识库 →」按钮，点击打开 drawer 显示抓取到的结构化数据

#### `<ProgressBar />`
- 左：「试跑进度」粗体
- 中：6px 高的灰条 + 紫青渐变填充，宽度 `approved/total`
- 右：紫色 `2 / 3 已确认`

#### `<DryRunCard />`（核心组件）
3 种状态：

**A. `approved` 已确认（折叠态）**
```
┌──────────────────────────────────────────────────────┐
│ [头像] Sarah Johnson · TechCorp Inc.       [✓ 已确认] │
│        "Hi, I saw your ad on Facebook…"              │
├──────────────────────────────────────────────────────┤
│ ▷ AI 草稿摘要：感谢询问 + 介绍 3 个核心功能 + 提议会议 │
└──────────────────────────────────────────────────────┘
```
- 整卡 1px 边、圆角 12px
- 头部：头像 + 客户名/公司 + 截断的客户邮件首句 + 右侧绿色 chip
- 底部：浅灰底，AI 草稿摘要（后端预生成的 1 行摘要）
- 整卡可点击展开看完整草稿（可选增强）

**B. `current` 当前确认中（展开态）**
```
┌──────────────────────────────────────────────────────┐
│ [头像] Marcus Reed · BlueOcean Logistics  [▸ 当前确认] │
│        "...could you share more about pricing…"      │
├──────────────────────────────────────────────────────┤
│  浅紫底完整草稿正文（AI 生成的 HTML）                 │
│  Hi Marcus, ...                                       │
│  ...                                                  │
├──────────────────────────────────────────────────────┤
│ [✓ 看起来不错] [✎ 编辑草稿] [↻ 换个角度重新生成]      │
│                          ⚠ 仅生成草稿，不会真正发送   │
└──────────────────────────────────────────────────────┘
```
- 整卡 2px 紫边 + 阴影
- 编辑草稿打开 Modal（基于 TipTap / Lexical 的富文本编辑器），保存后 `status: 'edited'`
- 重新生成调 `POST /api/v1/dry-run/regenerate` 带可选「换个角度」参数（更短 / 更专业 / 加 ROI 数字…）
- 重新生成期间整段正文区显示 typing skeleton

**C. `pending` 未到（折叠态、置灰）**
- 整卡 opacity 50%，按钮不可点
- 头部右侧灰色 chip 「等待中」

### 状态管理
```ts
type DryRunCardStatus = "pending" | "current" | "approved" | "edited";
// "edited" 是 approved 的细分（用于 telemetry，UI 上和 approved 一致）

// store 维护一个 samples 数组，approvedCount = filter(...).length
// firstPendingIdx 决定哪一张是 current
```

### 接口
```http
POST /api/v1/dry-run/generate
→ 200 {
    samples: [
      { id, from:{name, company, avatarHue}, excerpt, draftHtml, draftSummary }
    ],
    knowledgeStats: { pages, faqs, integrations, scenarios }
  }

POST /api/v1/dry-run/regenerate
  body: { sampleId, hint?: "shorter"|"professional"|"add_roi" }
→ 200 { draftHtml, draftSummary }

POST /api/v1/dry-run/approve
  body: { sampleId, finalDraft, action: "approve"|"edit" }
→ 204
```

### 状态 & 边界
- 用户没有任何未处理邮件：用 3 封内置 mock 邮件代替，banner 顶部加灰提示「你的收件箱没有 FB 表单邮件，这里用 3 封示例代替」
- 生成失败：单卡显示「Agent 生成失败 · [重试]」
- 没全部确认就点下一步：toast「请先确认 3 封草稿后再继续」
- 用户绕路直接访问 step 4：路由 guard 拒绝，跳回本页

### i18n keys
```
onboarding.step3.title / .subtitle
onboarding.step3.kb.title / .stats   (插值 {pages}{faqs}{integrations}{scenarios})
onboarding.step3.progress             (插值 {approved}/{total})
onboarding.step3.card.approved
onboarding.step3.card.current
onboarding.step3.card.pending
onboarding.step3.action.approve
onboarding.step3.action.edit
onboarding.step3.action.regenerate
onboarding.step3.warning              ("⚠ 仅生成草稿，不会真正发送给客户")
onboarding.step3.hint                 ("⏱ Agent 平均生成一封草稿耗时 1.8 秒")
```

### 验收标准
- [ ] 进入页面 3 秒内必定显示 3 张卡（含 mock fallback）
- [ ] 当前卡 2px 紫边 + 阴影，前后卡都置灰
- [ ] 编辑草稿 Modal 保存后回填，状态变 approved
- [ ] 重新生成期间禁用所有按钮 + skeleton
- [ ] 全部确认后下一步按钮自动激活并有微动效（亮一下）
- [ ] 草稿正文渲染必须 sanitize（DOMPurify）防 XSS

---

## 步骤 4 · 启用上线

### 路由
`/onboarding/launch`

### 用户目标
最后确认所有配置 → 一键启用 Agent → 跳转到 Dashboard。

### Layout 结构
```tsx
<OnboardingShell activeStep={4}>
  <PageHeader
    title="🎉 全部就绪 · 启用后立刻接管 52 封待跟进邮件"
    subtitle="Agent 会按你设定的工作时段自动回复，你可在概览页随时查看 / 暂停 / 转人工"
  />
  <Stepper current={4} />

  <div className="max-w-[920px] mx-auto space-y-5">
    <ConfigConfirmGrid />     {/* 2x2 4 项确认 */}
    <LaunchRulesCard />       {/* 上线规则 */}
    <LaunchCTA />             {/* 大号启用按钮 */}
  </div>

  {/* 这一页没有 footer Next，因为 CTA 就是终点 */}
</OnboardingShell>
```
> 注意：步骤 4 通常不需要 Footer 的下一步按钮（CTA 已经替代）。Footer 只显示「← 上一步」+ 文案。

### 组件细节

#### `<ConfigConfirmGrid />`
2 列 4 项，每项是横向排列的卡片：

| 图标 | 标题 | 副标 | 右侧操作 |
|---|---|---|---|
| ✓ | Google 邮箱已连接 | leads@acme.io · 一收一发 · 52 封待跟进 | 「改」→ step 2 |
| ✓ | 官网知识库已构建 | acme.io · 4 页 · 12 个 FAQ · 5 个集成 | 「改」→ step 2 |
| ✓ | 话术手册已加载 | 2 份文件 · 30 个场景 | 「改」→ step 2 |
| ✓ | 试跑通过 | 3 / 3 草稿已确认 · 风格匹配度 96% | 「重新试跑」→ step 3 |

- 图标：32px 圆 + 绿底白勾
- 副标里的数字从 onboarding draft + dry-run 结果聚合
- 「改」/「重新试跑」是 inline link，紫色 12px

#### `<LaunchRulesCard />`
卡片标题「🛡 上线规则确认」，内部 2x2 表单：
- **Agent 工作时段**：行内编辑器，默认「周一 - 周五 09:00 - 19:00 (UTC-7)」
  - 点击展开 popover：周几多选 chip + 起止时间 picker + 时区下拉
- **每日最多发送**：number input，默认 100，范围 10 - 500
- **非工作时段邮件**：select，2 选项
  - `queue` 排队至次日工作时段发送（推荐）
  - `send_now` 立即发送
- **转人工触发关键词**：text input，与 step 2 关联但允许在此独立编辑

#### `<LaunchCTA />`
- 居中
- 巨型按钮：宽 280px+，高 56px
- 紫青渐变 `from-brand-500 to-cyan-500`，圆角 12px，紫色光晕阴影
- 文字「🚀 启用 Agent · 立即开始处理 52 封邮件」
- 数字 `52` 从后端 inbox-scan 取，加粗
- 按钮下方 12px 灰字：「启用后可在「Agent 行为」随时暂停 / 调整 · 客户回复进展可在「概览」实时查看」

### 启用流程
```ts
async function launch() {
  setLaunching(true);
  try {
    await api.post("/onboarding/complete", { launchRules });
    // 后端会：
    // 1) 把 onboarding state 落库
    // 2) 启动 Agent worker
    // 3) 把 52 封 backlog 加入队列
    await sleep(800);   // 给 confetti 一点时间
    router.push("/dashboard?onboarded=1");
  } catch (e) {
    toast.error("启用失败：" + e.message);
    setLaunching(false);
  }
}
```

### 启用中过场
- 点击 CTA 后整页淡入一个全屏遮罩：
  - 中心：大号 emoji 🚀（带轻微 floating 动画）
  - 标题：「正在启用 Agent…」
  - 副标列表（fade-in 逐条）：
    - 「✓ 保存配置」
    - 「✓ 校准话术参数」
    - 「✓ 注入 52 封 backlog 邮件」
    - 「✓ 启动定时调度」
  - 全部完成后跳 `/dashboard?onboarded=1`，dashboard 顶部弹一个绿色 toast「🎉 MailPilot 已上线，正在处理你的第一封邮件」

### 状态 & 边界
- 任何确认项未通过：不渲染 CTA，改显示「请先完成：[xxx]」红卡
- 启用 API 失败 + 5 秒未返回：toast 错误 + CTA 重置可点击
- 用户已经启用过（重复进入本页）：直接跳 `/dashboard`

### i18n keys
```
onboarding.step4.title  (插值 {count})
onboarding.step4.subtitle
onboarding.step4.confirm.gmail / .website / .playbook / .dryrun
onboarding.step4.rules.title
onboarding.step4.rules.hours / .limit / .afterhours / .escalation
onboarding.step4.cta    (插值 {count})
onboarding.step4.cta_hint
onboarding.step4.launching.title
onboarding.step4.launching.step1..step4
onboarding.step4.dashboard_toast
```

### 验收标准
- [ ] 4 项确认卡数据全部来自真实 draft + dry-run 结果，不写死
- [ ] 「改」按钮跳回对应 step 且保留 query `?from=launch` 用于改完后直接跳回
- [ ] CTA 按钮 hover 时阴影加深 + 微微上浮（-2px）
- [ ] 启用流程整体不超过 3 秒
- [ ] 重复进入自动跳 dashboard

---

## 附录 A · 通用 UI 组件清单（建议先建）

| 组件 | 说明 | 复用范围 |
|---|---|---|
| `<OnboardingShell />` | 整体外壳（侧栏 + 主区） | 4 个页面 |
| `<Stepper />` | 顶部 1—2—3—4 步骤指示器 | 4 个页面 |
| `<PageHeader />` | 标题 + 副标 | 4 个页面 |
| `<OnboardingFooter />` | 底部上一步 / 提示 / 下一步 | step 1-3 |
| `<FormCard />` | 白底圆角卡片 + 标题 + 内容 slot | 各步骤 |
| `<ChipSelect />` | 多选 / 单选 chip 组 | step 1 客户来源 |
| `<GmailConnectButton />` | Gmail OAuth 按钮，3 态（idle/loading/connected） | step 2 |
| `<UrlListTextarea />` | textarea 自动校验每行 https URL | step 2 |
| `<FileDropzone />` | 拖拽上传 + 解析进度 | step 2 |
| `<KeywordsInput />` | 标签型关键词输入（中英文逗号自动归一） | step 2、4 |
| `<DryRunCard />` | 试跑卡（3 态） | step 3 |
| `<ProgressBar />` | 通用进度条 | step 3 |
| `<RichEditor />` | TipTap 富文本编辑器 | step 3 编辑草稿 |
| `<ConfirmRow />` | step 4 横向确认行 | step 4 |
| `<TimeRangePopover />` | 工作时段编辑 | step 4 |
| `<LaunchOverlay />` | 启用过场全屏遮罩 | step 4 |

## 附录 B · 全局动效 / 微交互

- 步骤切换：主区 fade + slide（300ms）
- 进度条变化：宽度 transition 400ms ease-out
- CTA hover：transform translateY(-2px) + shadow 加深
- 启用过场：✓ 列表 stagger 100ms 逐条 fade-in
- 错误 toast：从顶部 slide-down，4 秒后自动消失

## 附录 C · 测试要点

每个步骤需覆盖：
- 单元测试：表单校验、Zod schema、状态机转换
- 集成测试：API 模拟（MSW）+ 用户点击流
- E2E（Playwright）：完整跑通 4 步 + Gmail OAuth mock + 上传 mock + 启用成功
- 视觉回归（可选）：每页 1 张 Chromatic 快照

## 附录 D · 给 Codex 的工作指引

1. **先实现外壳和路由**：`<OnboardingShell />`、`<Stepper />`、`/onboarding/*` 路由 + 中间件守卫
2. **建数据契约**：`OnboardingState` 类型 + Zustand store + draft API
3. **按 step1 → step2 → step3 → step4 顺序实现**，每个步骤独立 PR，单独可测
4. **不要把所有 step 写在一个文件**：每页一个 `page.tsx` + 多个 component 文件
5. **样式**：能用 Tailwind class 解决就不写自定义 CSS；颜色统一走 token
6. **文案**：所有用户可见文字必须走 i18n，不许 hard-code 中文
7. **验收前自查**：打开根 `index.html` 的 `界面 6-9` 对比视觉细节

---

**文档版本**：v1.0 · 2026-06-07
**视觉参考**：`index.html`（界面 6 - 界面 9）
**在线预览**：https://mailpilot-prototype.vercel.app
