# MailPilot Onboarding · 4 个前端页面 Codex 提示词

> 技术栈：React + TypeScript + Tailwind CSS + shadcn/ui + React Hook Form + Zod。
> 不涉及后端 API，所有数据用 mock + localStorage 持久化即可。
> 视觉参考：https://mailpilot-prototype.vercel.app 的「界面 6 - 界面 9」

---

## 🧩 共享上下文（先实现一次）

### 设计 Token（Tailwind 扩展）
```ts
colors: {
  brand:   { 50:"#eef2ff", 500:"#4f46e5", 600:"#4338ca", 900:"#1e1b4b" },
  cyan:    { 500:"#22d3ee" },
  success: { 50:"#ecfdf5", 500:"#10b981", 700:"#047857" },
  warn:    { 50:"#fffbeb", 500:"#f59e0b" },
  danger:  { 50:"#fef2f2", 500:"#ef4444" },
  ink:     { 900:"#0f172a", 700:"#374151", 500:"#6b7280", 300:"#9ca3af" },
  surface: { 0:"#fff", 50:"#f7f8fc", border:"#eef0f4" },
}
```
品牌渐变：`bg-gradient-to-br from-brand-500 to-cyan-500`
字体：`-apple-system, Inter, "PingFang SC", sans-serif`

### 共享外壳 `<OnboardingShell />`
```
[左 220px 深色侧栏: Logo / 4 步导航 / 底部用户胶囊]
[右 主区: PageHeader → Stepper → 内容 slot → Footer]
```
- 整页 `min-h-screen bg-surface-50`
- 侧栏 `bg-slate-900`，每步：done=绿✓ / active=紫高亮 / pending=灰○
- Stepper：4 个圆（24px）+ 连线，状态同步侧栏
- Footer：固定底部 flex，「← 上一步 / 提示文案 / 下一步 →」

### 跨步状态（Zustand + localStorage）
```ts
type OnboardingState = {
  step: 1|2|3|4;
  account: { method, email, company, role, teamSize, leadSources[] };
  resources: { inboxEmail, sameInboxAsSender, websiteUrl, learningUrls[],
               playbooks[], meetingDuration, escalationKeywords[], maxAiTurns };
  dryRun: { samples[], approvedCount };
  launchRules: { workingHours, dailySendLimit, afterHours, escalationKeywords[] };
};
```

### 路由
- `/onboarding/signup` `/connect` `/dry-run` `/launch`
- `/onboarding` 跳到当前未完成步骤
- 中间件守卫：未填完前一步禁止跳后一步

---

## 1️⃣ 步骤 1 · 创建账户 `/onboarding/signup`

**目标**：让用户用 Google 或邮箱注册 + 录入团队画像。

**Header**
- 标题：`👋 欢迎使用 MailPilot · 先建一个工作账户`
- 副标：`账户用于绑定话术、客户、会议数据，后续可邀请同事加入同一团队`

**Layout**：`grid-cols-[1.1fr_1fr] gap-5 max-w-[920px] mx-auto`，2 张白色圆角卡片（`rounded-[14px] border border-surface-border p-5`）。

**卡片 ① 登录方式**
- 顶部 Google 按钮：白底 + 1px 边，文字「使用 Google 账户一键注册」，左侧蓝色 G 图标。点击触发 mock OAuth（直接跳本页 + 预填 email + disabled）
- 分割线：「— 或邮箱注册 —」居中文字 + 两侧 1px 灰线
- 工作邮箱 input：
  - Zod：`z.string().email()` + 拒绝 gmail.com / qq.com / 163.com（错误：`请使用公司邮箱注册`）
- 密码 input：`z.string().min(10).regex(/[A-Z]/).regex(/[0-9]/)`，失焦显示强度条（弱/中/强 三色块）
- 同意条款 Checkbox（默认未勾，必勾）：`我已阅读并同意《服务条款》《隐私政策》` 两个引号文字是 link，弹 Modal 显示占位文本
- 底部紫色提示框：`💡 推荐使用 Google 登录，下一步连接 Gmail 时可省去授权`

**卡片 ② 团队信息**
- 公司名称 input（必填，max 80）
- 你的角色 select：`销售（个人贡献者）/ 销售主管 / RevOps / 创始人 / CEO / 市场负责人`
- 团队规模 select：`1-5 / 5-20（推荐） / 20-100 / 100+`
- 主要客户来源（多选 Chip）：5 个 `Facebook 表单 / Google Ads / LinkedIn / 官网表单 / 展会·线下`
  - 默认选 FB + Google Ads
  - 选中态：`bg-brand-50 text-brand-600` 前缀 `✓`
  - 未选：`bg-white border border-surface-border text-ink-500`
  - 至少 1 个，否则下一步禁用

**Footer**
- 无「上一步」（隐藏按钮但占位）
- 提示：`🔐 工作邮箱仅用于登录，不会用作 Agent 发件箱`
- 下一步按钮（紫底白字）：`创建账户并进入第二步 →`

**交互**
- 提交：写 Zustand → `router.push("/onboarding/connect")`
- 已登录访问本页：直接跳下一步
- 提交中按钮 spinner + 禁用

**验收**
- [ ] 公共邮箱被拦截
- [ ] 客户来源未选时下一步禁用
- [ ] 刷新页面 draft 保留
- [ ] 移动端两卡堆叠为单栏
- [ ] Tab 键能依次聚焦到所有交互元素

---

## 2️⃣ 步骤 2 · 连接资源 `/onboarding/connect`

**目标**：连接 Gmail、官网、（可选）话术手册。

**Header**
- 标题：`🔌 连接 Agent 工作所需的资源`
- 副标：`连接你的 Google 邮箱、官网和话术手册，Agent 就能开始自动跟进 Facebook 表单线索`

**Layout**：`grid-cols-2 gap-5 max-w-[920px] mx-auto`
- 第一行：邮箱卡 + 官网卡
- 第二行：话术卡（`col-span-2`）

**邮箱卡 ① Google 邮箱**
- 收件邮箱按钮三态：
  - 未连接：白底 + 1px 边 + 文字 `+ 连接收件邮箱（接收 FB 表单提交）`
  - 连接中：spinner + `授权中…`
  - 已连接：绿胶囊 `bg-success-50 border-success-500/30 text-success-700`，文字 `✓ leads@acme.io · 已通过 OAuth 授权`，右侧灰链「断开」
- Checkbox：`一收一发使用同一个邮箱`（默认勾）
- 取消勾选 → 显示发件邮箱按钮（同上三态）
- 授权成功后 3s 后异步显示绿色检测框：`✓ 已检测到 52 封 来自 Facebook 表单的未处理邮件，将在启用后开始跟进`（用 setTimeout mock；加载用 skeleton bar）

**官网卡 ② 官网链接**
- 公司官网 URL input：必填，自动补 `https://`
- 重点学习页面 textarea：每行一个 URL，可选
  - 占位 3 行示例：`https://acme.io/pricing` / `/integrations/netsuite` / `/customers/3pl`
  - 校验：每行必须 https URL，非法行左侧加红色竖线
- 紫色提示框：`🧠 抓取后将自动总结出 产品定位 / 定价 / 集成清单 / 客户案例 知识库`

**话术卡 ③ 话术手册**（占满两列）
- 标题旁灰副标：`（非必备 · 不上传则使用通用销售话术）`
- 内部 `grid-cols-2 gap-5`：
  - 左列：
    - react-dropzone 虚线框：`⬆ 上传 PDF / DOCX / Markdown` + 副标 `支持多文件 · 单文件最大 20MB`
    - 上传后文件列表：📄 文件名 + 右侧绿字 `已解析 · N 个场景`（mock：上传后 1.5s 变为已解析）
  - 右列：
    - 会议时长偏好 select：`15 / 30（默认） / 45 分钟`
    - 转人工触发关键词 input：默认 `enterprise SLA, on-prem, legal, refund, complaint`
      - 实现 `<KeywordsInput>` 组件：输入逗号（中英文都支持）自动归一成 chip
    - AI 单线程最大自动往返轮次 select：`3 / 5（推荐） / 8 / 不限制`

**Footer**
- 上一步 → `/onboarding/signup`
- 提示：`🔒 所有数据加密存储 · 你可随时在「Agent 行为」中调整`
- 下一步：`下一步：让 Agent 试跑 3 封邮件 →`（官网未填则禁用）

**验收**
- [ ] OAuth 三态切换流畅
- [ ] 邮件扫描结果用 skeleton 加载
- [ ] 关键词中英文逗号都能解析
- [ ] 上传文件能从 `parsing` 变 `ready`
- [ ] 移动端 3 卡堆叠

---

## 3️⃣ 步骤 3 · Agent 试跑 `/onboarding/dry-run`

**目标**：让用户在启用前**亲眼看到** AI 草稿水平。明确「不会真的发」。

**Header**
- 标题：`🧪 Agent 已学会你的产品 · 用 3 封真实邮件验证一下`
- 副标：`这一步只生成草稿，不会真正发出去。你可以确认风格 / 内容 / 长度是否合适`

**进入页面初始化（mock）**
```ts
const SAMPLES = [
  { id:"s1", from:{ name:"Sarah Johnson", company:"TechCorp Inc.", initials:"SJ", hue:230 },
    excerpt:'"Hi, I saw your ad on Facebook about the SaaS automation tool…"',
    draftHtml:"<p>Hi Sarah,</p><p>Thanks for reaching out…</p>",
    draftSummary:"感谢询问 + 介绍 3 个核心功能 + 提议 6/9 14:00 通话",
    status:"approved" },
  { id:"s2", from:{ name:"Alex Chen", company:"Pixel Labs", initials:"AC", hue:230 },
    excerpt:'"Sounds interesting. Can you send pricing for a 50-seat team?"',
    draftSummary:"50 席团队定价 ($4.2k/月) + ROI 速算表 + 提议下周三 demo",
    status:"approved" },
  { id:"s3", from:{ name:"Marcus Reed", company:"BlueOcean Logistics", initials:"MR", hue:40 },
    excerpt:'"...could you share more about pricing and integration with NetSuite?"',
    draftHtml:`<p>Hi Marcus,</p><p>Thanks for reaching out!...</p>`,
    status:"current" },
];
```

**Layout（顺序）**
1. **知识库 Banner**：`bg-gradient-to-br from-brand-50 to-cyan-50 border border-brand-500/20 rounded-xl p-4 max-w-[1080px] mx-auto mb-4`
   - 左：🧠 24px
   - 中：粗体 `Agent 已建立知识库` + 紫色 12.5px `acme.io · 4 个核心页面 · 12 项常见问题 · 5 个集成方案 · 30 个话术场景已解析`
   - 右：ghost 按钮 `查看知识库 →`（点击打开 Drawer，内容 mock）
2. **进度条**：`flex gap-3 items-center max-w-[1080px] mx-auto mb-3`
   - 左：粗体 `试跑进度`
   - 中：6px 高灰条 + 紫青渐变填充宽度 = `approved/total`
   - 右：紫字 `2 / 3 已确认`
3. **试跑卡列表**：

**`<DryRunCard>` 三态**

```
approved（折叠）:
┌──────────────────────────────────────────┐
│ 1px 边 · 圆角 12 · 无阴影                │
│ ┌头部┐ 头像 / 客户名公司 / 截断邮件 / [✓ 已确认 chip] │
│ ┌底部┐ 浅灰底 12.5px 灰字「▷ AI 草稿摘要：xxx」      │
└──────────────────────────────────────────┘

current（展开）:
┌──────────────────────────────────────────┐
│ 2px 紫边 · 阴影 0 8px 24px -12px brand   │
│ 头部（同上但 chip 紫色「▸ 当前确认」）             │
│ 完整草稿正文 bg-[#fafbff] · prose 渲染 sanitize │
│ 底部白底 + 1px 顶边：                              │
│   [✓ 看起来不错] [✎ 编辑草稿] [↻ 换个角度重新生成] │
│                       ⚠ 仅生成草稿，不会真正发送   │
└──────────────────────────────────────────┘

pending（折叠 + 置灰）:
opacity-50, 按钮全部 disabled, chip 灰「等待中」
```

**头像生成**：用 `hsl(${hue} 70% 90%)` 作背景 + `hsl(${hue} 60% 35%)` 作字色，显示 initials。

**交互**
- `✓ 看起来不错` → status 变 approved，当前卡折叠，下一张未到的变 current
- `✎ 编辑草稿` → 打开 Dialog，里面用 TipTap（或简单的 `contentEditable` div）编辑 `draftHtml`，保存后变 approved
- `↻ 换个角度重新生成` → 显示 dropdown menu 3 选项 `更简短 / 更专业 / 加 ROI 数字`，选中后 1.5s mock 重写 draftHtml（期间整段显示 skeleton）

**安全**：渲染 `draftHtml` 用 DOMPurify sanitize。

**Footer**
- 上一步 → `/onboarding/connect`
- 提示：`⏱ Agent 平均生成一封草稿耗时 1.8 秒`
- 下一步：`下一步：启用上线 →`，`approvedCount < 3` 时禁用；全确认时按钮亮一下（`@keyframes pulse-glow`）

**验收**
- [ ] 当前卡 2px 紫边 + 紫色阴影
- [ ] 三态切换流畅，动画 300ms
- [ ] 编辑 Modal 保存后状态变 approved
- [ ] 重新生成期间禁用所有按钮 + skeleton
- [ ] 草稿 HTML 被 sanitize（XSS 测试通过）

---

## 4️⃣ 步骤 4 · 启用上线 `/onboarding/launch`

**目标**：最后确认 + 一键启用，跳 dashboard。

**Header**
- 标题：`🎉 全部就绪 · 启用后立刻接管 52 封待跟进邮件`（`52` 取自 step 2 mock）
- 副标：`Agent 会按你设定的工作时段自动回复，你可在概览页随时查看 / 暂停 / 转人工`

**Layout**：`max-w-[920px] mx-auto space-y-4`

### A. 配置确认网格 `<ConfigConfirmGrid>`
`grid-cols-2 gap-3` · 4 张白底卡片：

```
┌─────────────────────────────────────────────┐
│ [32px绿圆✓]  Google 邮箱已连接          [改]│
│              leads@acme.io · 一收一发 · 52 封│
└─────────────────────────────────────────────┘
```
4 项内容：
| 标题 | 副标 | 操作 |
|---|---|---|
| Google 邮箱已连接 | leads@acme.io · 一收一发 · 52 封待跟进 | 改 → step 2 |
| 官网知识库已构建 | acme.io · 4 页 · 12 个 FAQ · 5 个集成 | 改 → step 2 |
| 话术手册已加载 | 2 份文件 · 30 个场景 | 改 → step 2 |
| 试跑通过 | 3 / 3 草稿已确认 · 风格匹配度 96% | 重新试跑 → step 3 |

- 「改」是紫色 12px inline link，跳转时带 `?from=launch` query
- 数据来自 Zustand store，不要写死

### B. 上线规则卡 `<LaunchRulesCard>`
`<FormCard>` 标题 `🛡 上线规则确认`，内部 `grid-cols-2 gap-3`：
- **Agent 工作时段**：点击展开 popover：
  - 周几多选 chip（一/二/三/四/五/六/日）
  - 起止时间 picker（两个 time input）
  - 时区下拉（默认 UTC-7）
  - 显示文字回显：`周一 - 周五 09:00 - 19:00 (UTC-7)`
- **每日最多发送**：number input，默认 100，范围 10-500
- **非工作时段邮件**：select：`排队至次日工作时段发送（推荐） / 立即发送`
- **转人工触发关键词**：`<KeywordsInput>`，与 step 2 关联可独立改

### C. 启用 CTA `<LaunchCTA>`
居中：
```tsx
<button className="bg-gradient-to-br from-brand-500 to-cyan-500
                   text-white text-[15px] font-medium
                   px-10 py-3.5 rounded-xl
                   shadow-[0_10px_30px_-10px_rgba(79,70,229,0.5)]
                   hover:-translate-y-0.5 hover:shadow-[0_14px_32px_-10px_rgba(79,70,229,0.6)]
                   transition-all">
  🚀 启用 Agent · 立即开始处理 <b>52</b> 封邮件
</button>
<p className="mt-3 text-xs text-ink-500">
  启用后可在「Agent 行为」随时暂停 / 调整 · 客户回复进展可在「概览」实时查看
</p>
```

### D. 启用过场 `<LaunchOverlay>`
点击 CTA 后整页全屏 fixed 遮罩：
```
[半透明黑底 backdrop-blur]
   🚀 (大号 emoji, animate-bounce-slow)
   标题：「正在启用 Agent…」
   步骤列表（依次 fade-in，每条间隔 600ms）：
     ✓ 保存配置
     ✓ 校准话术参数
     ✓ 注入 52 封 backlog 邮件
     ✓ 启动定时调度
   全部完成（~3s 总时长）→ router.push("/dashboard?onboarded=1")
```
dashboard 顶部弹绿色 toast：`🎉 MailPilot 已上线，正在处理你的第一封邮件`

### Footer
- 上一步 → `/onboarding/dry-run`
- 无下一步（CTA 替代）

### 边界
- 任何确认项 store 数据缺失：替换 CTA 区为红卡 `请先完成：[xxx]`，提供按钮跳回
- 已启用过的用户访问本页：直接跳 dashboard

**验收**
- [ ] 4 项确认卡数据全部从 store 取
- [ ] CTA hover 上浮 + 阴影加深
- [ ] 启用过场 4 步 stagger 动画流畅
- [ ] 3 秒后必定跳 dashboard
- [ ] 重复进入自动跳 dashboard

---

## 🧰 建议先建的通用前端组件

| 组件 | 用在哪 |
|---|---|
| `<OnboardingShell>` | 全部 |
| `<Stepper>` | 全部 |
| `<PageHeader>` | 全部 |
| `<OnboardingFooter>` | step 1-3 |
| `<FormCard>` | 全部 |
| `<ChipSelect multi>` | step 1 客户来源 |
| `<OAuthButton>`（三态） | step 2 |
| `<UrlListTextarea>` | step 2 |
| `<FileDropzone>` | step 2 |
| `<KeywordsInput>` | step 2 / 4 |
| `<DryRunCard>` | step 3 |
| `<ProgressBar>` | step 3 |
| `<ConfirmRow>` | step 4 |
| `<TimeRangePopover>` | step 4 |
| `<LaunchOverlay>` | step 4 |

---

## 🎯 给 Codex 的总工作指引

1. 先实现 **共享上下文**（外壳、Stepper、Footer、Zustand store、路由守卫、Tailwind token）
2. 按 **step 1 → 2 → 3 → 4** 顺序实现，每步独立 PR
3. 每页一个 `page.tsx` + 多个 component 文件，不要堆一起
4. 样式优先用 Tailwind class，能不写自定义 CSS 就不写
5. 所有数据用 mock 即可，预留接口型签名给后端对接
6. 视觉细节对照 https://mailpilot-prototype.vercel.app 的「界面 6 - 界面 9」

---

**文档版本**：v1.0 (Frontend-only) · 2026-06-07
**视觉参考**：`index.html` 界面 6 - 界面 9
**在线预览**：https://mailpilot-prototype.vercel.app
