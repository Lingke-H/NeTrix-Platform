# NeTrix 核心 MVP 技术与开发方案 (正式版)

**文档定位：** 面向技术组的整体 MVP 开发纲领。经过团队一致决定，我们将初始的“全栈大部头”方案**精简为聚焦单点核心体验**的 MVP。本方案彻底砍掉了复杂的匹配微服务、沉重的数据库体系和鉴权，集中咱们三人的全部火力，打造一条极具视觉冲击力且逻辑闭环的“黄金展示路径 (Golden Path)”。

---

## 一、 核心模块交付标准（每个人到底要做到什么程度？）

为了保证 MVP 的高优纯粹，每位成员必须清晰知道自己的职责边界，**绝不多写一行评委看不到的代码**。

### 1. 赛博视觉与前端主 R【CS 同学】
*   **不要做：** 不要写复杂的路由鉴权、不要写注册表单、不要去折腾复杂的后端部署逻辑。
*   **核心攻坚区（基础及格线）：**
    *   画出带有青/洋红发光特效的 React Flow 静态节点图（NeTrix 赛博神经元网络）。
    *   写一个带下拉框的轻量化入口页（选择协议和插件）。
    *   用静态 JSON 或 Mock 数据实现论坛列表和帖子详情页 UI。
    *   做一个带有 Loading 态和文字逐字出现动画的“召唤先知 (Oracle)”聊天模块。
*   **完美展示线：** 页面切换有平滑的过渡动画，鼠标悬停时有赛博朋克风的失真（Glitch）或流光动效，确保大屏路演或录播 Demo 时视觉效果极其惊艳。

### 2. 数据管道修井工【EEE 同学】
*   **不要做：** 不要研究 Supabase 的 RLS（行级安全）、不要折腾 Edge Functions、不要设计复杂的用户外键关联表。
*   **核心攻坚区（基础及格线）：**
    *   在 Supabase 网页后台建好 `posts` 和 `comments` 两张最基础的表。
    *   在 Next.js 的工程中封装好 `getPosts()` 和 `createComment()` 等数据交互函数。
    *   确保前端的交互能成功连通云端数据库进行增删查改。
*   **完美展示线：** 与前端同学无缝丝滑联调，把 UI 里的假数据全部替换为真实的 API 驱动，同时处理好数据加载时的骨架屏（Skeleton），避免白屏。

### 3. AI 守护者与造假大师【Math 同学】
*   **不要做：** 彻底停止搭建复杂的 FastAPI 匹配微服务或外置爬虫系统。
*   **核心攻坚区（基础及格线）：**
    *   **核心内容填充：** 用 Python 脚本在本地批量调大模型，生成 30 篇以上极高专业度的理科/商科学术讨论贴和优质评论，一键导入 Supabase 数据库。这决定了我们 Demo 时的“逼格”。
    *   **AI 接口实现：** 在 Next.js 里面建一个 `app/api/chat/route.ts` 接口。对接大模型（如 OpenAI / DeepSeek），接收前端传来的帖子上下文，返回高质量的 AI 答疑流（Streaming）。无服务、轻量级。

---

## 二、 极限“瘦身”后的技术选型

| 模块 | 核心选型 | 为什么这么选？ |
|------|---------|-------------|
| **前端框架** | Next.js 14 + Tailwind CSS | 单体架构，Server Actions / API Routes 直接搞定全栈需求 |
| **赛博组件** | React Flow + shadcn/ui | 极高颜值的现成组件库，所见即所得 |
| **数据库** | Supabase (裸用 PostgreSQL) | 仅当一个带 API 的云端 JSON 数据池用，舍弃权限体系，零运维 |
| **AI 集成** | Next.js API Routes 直接调 LLM | 砍掉 FastAPI 独立系统，降本增效，前后端一口气联调 |
| **部署** | Vercel | 推送 Github 即上线，无需操心任何公网 IP 或云服务器配置 |

---

## 三、 极简数据库 Schema（Supabase）

只需在 Supabase 后台运行以下极度简化的建表语句。保留最直接的数据交互结构。

```sql
-- 1. 帖子表
CREATE TABLE posts (
  id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  hub_slug      TEXT NOT NULL,        -- 所属节点，例如 'math', 'business'
  author_name   TEXT NOT NULL,        -- 直接写入用户名，如 'cyber_student_01'
  author_plugin TEXT NOT NULL,        -- 装备的插件，如 '高级微积分引擎'
  title         TEXT NOT NULL,
  content       TEXT NOT NULL,
  created_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- 2. 评论表
CREATE TABLE comments (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  post_id         UUID NOT NULL REFERENCES posts(id) ON DELETE CASCADE,
  author_name     TEXT NOT NULL,
  content         TEXT NOT NULL,
  is_ai_generated BOOLEAN NOT NULL DEFAULT FALSE, -- 标明是不是先知 Oracle
  created_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

---

## 四、 仅需的初始化命令

因为砍掉了微服务，现在的 Monorepo 工作区主体就是一个纯粹的 Next.js 项目（在 `apps/web` 下）。

```bash
# 1. 创建带 Tailwind 的 Next.js 应用（根目录下运行）
pnpm create next-app@latest apps/web --typescript --tailwind --eslint --app --src-dir --import-alias "@/*"

# 2. 进入项目并初始化 UI 库
cd apps/web
pnpm dlx shadcn@latest init
pnpm dlx shadcn@latest add button card badge input textarea dialog

# 3. 安装 React Flow, Supabase 和 Markdown 渲染库
pnpm add @xyflow/react @supabase/supabase-js react-markdown
```

---

## 五、 MVP 打磨里程碑（黄金落地节奏）

#### 🚀 阶段 1：前置视觉布景（"Looks Good"）
- 此阶段只允许出现 Mock 数据，绝不允许任何人拖慢前端进度。
- 设计 NeTrix 赛博星图（React Flow 节点图）。
- 完成论坛列表、文章详情及聊天对话框纯静态 UI，落实深色模式和发光动效。

#### 🔗 阶段 2：数据挂载与 API 联通（"Works Well"）
- 启动 Supabase 和云端连接。
- Math 同学脚本打满测试数据，EEE 同学完成对前端组件的数据挂载。
- 后端打通 `app/api/chat/route.ts` 大模型实时流（Streaming）。

#### ⚔️ 阶段 3：体验熔断与路演全剧本死磕（"Ready to Demo"）
- **功能熔断卡点：** 坚决停止一切新功能开发！一切以 UI 别崩、不报错为准。
- 把 UI 里的细节问题吃透，防止真实数据把页面排版撑爆。
- 全员花至少 2 天去死磕产品路演那 3 分钟：鼠标点哪、输入什么字、说什么话、放什么音乐，精确到秒，确保震撼全场。

---
> **文档版本：** v3.0 (单点击穿 MVP 正式版) | **更新核心：** 去掉“期末限定”字眼，彻底确立这就是我们这代 MVP 的正式演进路线。