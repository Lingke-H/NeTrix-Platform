# NeTrix 独立平台 — 技术栈与开发框架方案

**文档定位：** 面向技术组下次例会，提供一份**可立即执行**的全栈技术方案。涵盖技术选型、数据库 Schema、仓库结构、初始化命令，以及 6月8日路演的**做与不做清单**。

---

## 目录

- [一、技术栈选型（Tech Stack）](#一技术栈选型tech-stack)
- [二、系统架构总览（Architecture Overview）](#二系统架构总览architecture-overview)
- [三、数据库 Schema 设计（Supabase PostgreSQL）](#三数据库-schema-设计supabase-postgresql)
- [四、GitHub 仓库目录结构](#四github-仓库目录结构)
- [五、Study Buddy 匹配服务集成方案](#五study-buddy-匹配服务集成方案)
- [六、初始化命令（Init Commands）](#六初始化命令init-commands)
- [七、开发策略：做 vs 假装做（Build vs Fake）](#七开发策略做-vs-假装做build-vs-fake)
- [八、四周冲刺排期建议](#八四周冲刺排期建议)
- [九、开发人员分工建议](#九开发人员分工建议)

---

## 一、技术栈选型（Tech Stack）

| 层级 | 选型 | 选择理由 |
|------|------|---------|
| **前端框架** | **Next.js 14 (App Router)** + TypeScript | SSR/SSG 开箱即用，Vercel 一键部署，生态最成熟 |
| **UI 组件库** | **shadcn/ui** + Tailwind CSS v4 | 复制粘贴式组件，零运行时开销，深度可定制赛博朋克主题 |
| **节点地图 UI** | **React Flow** | 轻量级节点图库，完美实现 Cyber Neural Network 隐喻，无需 3D |
| **图标** | **Lucide React** | shadcn/ui 默认图标库，一致性好 |
| **BaaS 平台** | **Supabase** | 免费额度极大（50K MAU），PostgreSQL 原生，Auth/Realtime/Edge Functions/Storage 全包 |
| **数据库** | Supabase 托管 **PostgreSQL** + **Row Level Security (RLS)** | 无需自建数据库，RLS 替代后端权限层 |
| **认证** | Supabase Auth（Email + OAuth） | 一行代码接入 GitHub/Google 登录 |
| **AI Oracle** | **Supabase Edge Function** (Deno) 调用 OpenAI/DeepSeek API | 无需额外服务器，按调用计费 |
| **匹配算法微服务** | **FastAPI** (Python) 部署到 **Render** | 直接复用 Hackathon 现有代码，最小改动 |
| **前端部署** | **Vercel** | Next.js 亲儿子，免费额度足够 MVP |
| **版本控制** | GitHub + GitHub Actions (CI) | 标准协作流程 |

### 为什么选 Supabase 而非 Firebase？

1. **PostgreSQL > Firestore**：关系型数据天然适合论坛帖子、评论、投票的关联查询
2. **Edge Functions (Deno)** 比 Cloud Functions 冷启动更快
3. **RLS（行级安全）** 让你不用写后端中间件就能控制数据权限
4. **免费额度**：50K MAU、500MB 数据库、1GB 存储、500K Edge Function 调用/月

---

## 二、系统架构总览（Architecture Overview）

```
┌─────────────────────────────────────────────────────────┐
│                    用户浏览器 / 移动端                      │
│                  Next.js 14 (Vercel)                      │
│         React Flow 节点图 + shadcn/ui + Tailwind          │
└──────────┬───────────────┬──────────────────┬────────────┘
           │               │                  │
           ▼               ▼                  ▼
   ┌───────────┐   ┌──────────────┐   ┌──────────────────┐
   │ Supabase  │   │  Supabase    │   │  FastAPI 微服务   │
   │   Auth    │   │  Edge Func   │   │  (Render.com)    │
   │  认证服务  │   │  AI Oracle   │   │  匹配算法引擎     │
   └───────────┘   └──────┬───────┘   └────────┬─────────┘
                          │                    │
                          ▼                    │
                  ┌──────────────┐             │
                  │ OpenAI /     │             │
                  │ DeepSeek API │             │
                  └──────────────┘             │
                                               │
   ┌───────────────────────────────────────────┘
   │
   ▼
┌────────────────────────────────────────────┐
│          Supabase PostgreSQL               │
│    (数据库 + RLS + Realtime 订阅)           │
│    profiles / posts / comments / votes     │
│    resources / match_requests              │
└────────────────────────────────────────────┘
```

**关键数据流：**

1. **Knowledge Hub（论坛）：** 前端 → Supabase Client SDK → PostgreSQL (RLS 鉴权) → Realtime 推送新回复
2. **Summon Oracle（召唤AI）：** 前端点击按钮 → Supabase Edge Function → 调用 LLM API → 将 AI 回复作为一条普通 comment 写入数据库
3. **Study Buddy（雷达扫描）：** 前端 → Next.js API Route (代理) → FastAPI 微服务 → 读 Supabase DB → 返回 Top 3 匹配结果

---

## 三、数据库 Schema 设计（Supabase PostgreSQL）

> 以下使用 Supabase Migration SQL 格式。所有表均需配置 RLS 策略。

### 3.1 枚举类型

```sql
-- 用户协议（Protocol）状态
CREATE TYPE user_protocol AS ENUM (
  'seeker',      -- 求知者（蓝色节点，寻找答案）
  'oracle',      -- 先知（金色节点，准备答疑）
  'builder',     -- 建造者（绿色节点，寻找组队）
  'stealth'      -- 隐身（灰色节点，潜水浏览）
);

-- 资源类别
CREATE TYPE resource_category AS ENUM (
  'past_paper',      -- 历年真题
  'study_material',  -- 学习资料
  'ai_whisper',      -- AI 对话精选（Prompt 分享）
  'news'             -- 校园资讯
);

-- 投票类型
CREATE TYPE vote_target_type AS ENUM ('post', 'comment');
```

### 3.2 profiles（用户画像 / 身份系统）

```sql
CREATE TABLE profiles (
  id             UUID PRIMARY KEY REFERENCES auth.users(id) ON DELETE CASCADE,
  username       TEXT UNIQUE NOT NULL,
  display_name   TEXT NOT NULL DEFAULT '',
  bio            TEXT DEFAULT '',

  -- NeTrix 身份系统：Protocol & Plugins
  protocol       user_protocol NOT NULL DEFAULT 'stealth',
  plugins        JSONB NOT NULL DEFAULT '[]',
  -- plugins 示例: ["Advanced Calculus Engine v2.0", "Macroeconomics Data-Miner"]

  -- 所属 Hub（学院）
  hub_id         UUID REFERENCES hubs(id),

  -- 匹配算法所需的向量化画像（从 Hackathon 迁移）
  skill_vector   JSONB DEFAULT '{}',
  -- 示例: {"modeling": 7, "coding": 9, "writing": 4}
  personality    JSONB DEFAULT '{}',
  -- 示例: {"leader": 3, "executor": 8, "supporter": 5}

  -- 游戏化积分
  xp_points      INT NOT NULL DEFAULT 0,
  level          INT NOT NULL DEFAULT 1,

  created_at     TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at     TIMESTAMPTZ NOT NULL DEFAULT now()
);

-- 索引
CREATE INDEX idx_profiles_hub ON profiles(hub_id);
CREATE INDEX idx_profiles_protocol ON profiles(protocol);
```

### 3.3 hubs（学院节点 — 一级节点）

```sql
CREATE TABLE hubs (
  id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  name        TEXT NOT NULL,           -- 如 "Business School"
  slug        TEXT UNIQUE NOT NULL,    -- URL 友好标识
  description TEXT DEFAULT '',
  color_hex   TEXT NOT NULL DEFAULT '#00FFFF', -- 节点发光颜色
  position_x  FLOAT NOT NULL DEFAULT 0,        -- React Flow 节点坐标
  position_y  FLOAT NOT NULL DEFAULT 0,
  created_at  TIMESTAMPTZ NOT NULL DEFAULT now()
);
```

### 3.4 sub_nodes（话题节点 — 二级节点）

```sql
CREATE TABLE sub_nodes (
  id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  hub_id      UUID NOT NULL REFERENCES hubs(id) ON DELETE CASCADE,
  name        TEXT NOT NULL,           -- 如 "Finance", "Calculus"
  slug        TEXT NOT NULL,
  description TEXT DEFAULT '',
  position_x  FLOAT NOT NULL DEFAULT 0,
  position_y  FLOAT NOT NULL DEFAULT 0,
  post_count  INT NOT NULL DEFAULT 0,  -- 反范式计数，提升查询性能
  created_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
  UNIQUE(hub_id, slug)
);

CREATE INDEX idx_subnodes_hub ON sub_nodes(hub_id);
```

### 3.5 posts（帖子 / 问题）

```sql
CREATE TABLE posts (
  id            UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  author_id     UUID NOT NULL REFERENCES profiles(id) ON DELETE CASCADE,
  sub_node_id   UUID NOT NULL REFERENCES sub_nodes(id) ON DELETE CASCADE,
  title         TEXT NOT NULL,
  content       TEXT NOT NULL,            -- Markdown 格式
  is_ai_generated BOOLEAN NOT NULL DEFAULT FALSE,
  status        TEXT NOT NULL DEFAULT 'open' CHECK (status IN ('open', 'resolved', 'archived')),
  tags          TEXT[] DEFAULT '{}',
  upvote_count  INT NOT NULL DEFAULT 0,   -- 反范式计数
  downvote_count INT NOT NULL DEFAULT 0,
  comment_count INT NOT NULL DEFAULT 0,
  is_pinned     BOOLEAN NOT NULL DEFAULT FALSE,
  created_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_posts_subnode ON posts(sub_node_id, created_at DESC);
CREATE INDEX idx_posts_author ON posts(author_id);
```

### 3.6 comments（评论 / H2A2H 回复）

```sql
CREATE TABLE comments (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  post_id         UUID NOT NULL REFERENCES posts(id) ON DELETE CASCADE,
  author_id       UUID NOT NULL REFERENCES profiles(id) ON DELETE CASCADE,
  parent_id       UUID REFERENCES comments(id) ON DELETE CASCADE, -- 支持嵌套回复
  content         TEXT NOT NULL,
  is_ai_generated BOOLEAN NOT NULL DEFAULT FALSE,  -- 标记 Oracle AI 回复
  upvote_count    INT NOT NULL DEFAULT 0,
  downvote_count  INT NOT NULL DEFAULT 0,
  created_at      TIMESTAMPTZ NOT NULL DEFAULT now(),
  updated_at      TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_comments_post ON comments(post_id, created_at ASC);
```

### 3.7 votes（投票系统 — AI 与人类平权）

```sql
CREATE TABLE votes (
  id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id     UUID NOT NULL REFERENCES profiles(id) ON DELETE CASCADE,
  target_type vote_target_type NOT NULL,
  target_id   UUID NOT NULL,              -- post 或 comment 的 UUID
  value       SMALLINT NOT NULL CHECK (value IN (-1, 1)), -- -1 downvote, 1 upvote
  created_at  TIMESTAMPTZ NOT NULL DEFAULT now(),
  UNIQUE(user_id, target_type, target_id)  -- 每人每目标只能投一票
);
```

### 3.8 resources（资源中心 / AI Whispers Gallery）

```sql
CREATE TABLE resources (
  id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  author_id   UUID NOT NULL REFERENCES profiles(id) ON DELETE CASCADE,
  sub_node_id UUID REFERENCES sub_nodes(id),
  category    resource_category NOT NULL,
  title       TEXT NOT NULL,
  description TEXT DEFAULT '',
  url         TEXT NOT NULL,              -- 资源链接 / AI 对话分享链接
  tags        TEXT[] DEFAULT '{}',
  upvote_count INT NOT NULL DEFAULT 0,
  created_at  TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_resources_category ON resources(category);
CREATE INDEX idx_resources_subnode ON resources(sub_node_id);
```

### 3.9 match_requests（雷达扫描 / 匹配记录）

```sql
CREATE TABLE match_requests (
  id          UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id     UUID NOT NULL REFERENCES profiles(id) ON DELETE CASCADE,
  status      TEXT NOT NULL DEFAULT 'pending' CHECK (status IN ('pending', 'scanning', 'completed', 'failed')),
  results     JSONB DEFAULT '[]',
  -- results 示例: [{"user_id": "xxx", "rank": 1, "score": 0.87, "reasons": {...}}, ...]
  created_at  TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE INDEX idx_match_user ON match_requests(user_id, created_at DESC);
```

### 3.10 RLS 策略示例（核心安全层）

```sql
-- profiles: 所有人可读，仅本人可改
ALTER TABLE profiles ENABLE ROW LEVEL SECURITY;
CREATE POLICY "profiles_select" ON profiles FOR SELECT USING (true);
CREATE POLICY "profiles_update" ON profiles FOR UPDATE USING (auth.uid() = id);

-- posts: 所有人可读，认证用户可创建，仅作者可改
ALTER TABLE posts ENABLE ROW LEVEL SECURITY;
CREATE POLICY "posts_select" ON posts FOR SELECT USING (true);
CREATE POLICY "posts_insert" ON posts FOR INSERT WITH CHECK (auth.uid() = author_id);
CREATE POLICY "posts_update" ON posts FOR UPDATE USING (auth.uid() = author_id);

-- votes: 仅本人可操作自己的投票
ALTER TABLE votes ENABLE ROW LEVEL SECURITY;
CREATE POLICY "votes_select" ON votes FOR SELECT USING (true);
CREATE POLICY "votes_insert" ON votes FOR INSERT WITH CHECK (auth.uid() = user_id);
CREATE POLICY "votes_delete" ON votes FOR DELETE USING (auth.uid() = user_id);
```

---

## 四、GitHub 仓库目录结构

```
netrix/
├── .github/
│   └── workflows/
│       └── ci.yml                      # GitHub Actions: lint + type-check
│
├── apps/
│   └── web/                            # Next.js 14 前端应用（主战场）
│       ├── public/
│       │   └── fonts/                  # 赛博朋克字体（JetBrains Mono, Orbitron）
│       ├── src/
│       │   ├── app/                    # Next.js App Router
│       │   │   ├── (auth)/             # 路由组: 登录/注册
│       │   │   │   ├── login/page.tsx
│       │   │   │   └── register/page.tsx
│       │   │   ├── (dashboard)/        # 路由组: 主界面（需登录）
│       │   │   │   ├── matrix/page.tsx         # 核心页: Cyber Neural Network 节点地图
│       │   │   │   ├── hub/[slug]/page.tsx      # Hub 详情（某学院的子节点列表）
│       │   │   │   ├── node/[slug]/page.tsx     # Sub-node 详情（帖子列表 = StackOverflow 式）
│       │   │   │   ├── post/[id]/page.tsx       # 帖子详情 + 评论 + Summon Oracle 按钮
│       │   │   │   ├── radar/page.tsx           # Study Buddy 雷达扫描页
│       │   │   │   ├── resources/page.tsx       # 资源中心 + AI Whispers Gallery
│       │   │   │   ├── profile/page.tsx         # 个人资料 / Protocol 切换 / Plugin 装备
│       │   │   │   └── layout.tsx               # Dashboard 通用布局（侧边栏 + Header）
│       │   │   ├── api/                # Next.js API Routes（作代理层）
│       │   │   │   └── match/route.ts          # 代理转发到 FastAPI 匹配微服务
│       │   │   ├── layout.tsx          # 根布局（全局字体、主题、Supabase Provider）
│       │   │   ├── page.tsx            # Landing Page（赛博朋克风着陆页）
│       │   │   └── globals.css         # Tailwind 全局样式 + 赛博朋克 CSS 变量
│       │   │
│       │   ├── components/
│       │   │   ├── ui/                 # shadcn/ui 组件（由 CLI 自动生成）
│       │   │   │   ├── button.tsx
│       │   │   │   ├── card.tsx
│       │   │   │   ├── badge.tsx
│       │   │   │   ├── dialog.tsx
│       │   │   │   ├── input.tsx
│       │   │   │   ├── textarea.tsx
│       │   │   │   ├── avatar.tsx
│       │   │   │   ├── dropdown-menu.tsx
│       │   │   │   └── ...
│       │   │   ├── matrix/             # Cyber Neural Network 专用组件
│       │   │   │   ├── MatrixCanvas.tsx        # React Flow 画布容器
│       │   │   │   ├── HubNode.tsx             # 一级节点（学院）自定义节点
│       │   │   │   ├── SubNode.tsx             # 二级节点（话题）自定义节点
│       │   │   │   ├── UserNode.tsx            # 用户节点（Protocol 决定颜色/动画）
│       │   │   │   └── NeonEdge.tsx            # 霓虹发光连线
│       │   │   ├── forum/              # Knowledge Hub 论坛组件
│       │   │   │   ├── PostCard.tsx
│       │   │   │   ├── PostDetail.tsx
│       │   │   │   ├── CommentThread.tsx       # 嵌套评论（人类 + AI 平权展示）
│       │   │   │   ├── SummonOracleBtn.tsx     # "召唤先知" 按钮
│       │   │   │   ├── VoteButtons.tsx         # 上下投票组件
│       │   │   │   └── PostEditor.tsx          # Markdown 编辑器
│       │   │   ├── radar/              # Study Buddy 雷达扫描组件
│       │   │   │   ├── RadarPulse.tsx          # CSS 雷达脉冲动画
│       │   │   │   ├── MatchResultCard.tsx     # 匹配结果卡片
│       │   │   │   └── RadarScanner.tsx        # 扫描流程控制器
│       │   │   ├── resource/           # 资源中心组件
│       │   │   │   ├── ResourceCard.tsx
│       │   │   │   ├── ResourceGrid.tsx
│       │   │   │   └── AiWhisperCard.tsx       # AI 对话精选卡片
│       │   │   ├── profile/            # 身份系统组件
│       │   │   │   ├── ProtocolSwitcher.tsx     # Protocol 切换器
│       │   │   │   ├── PluginEquip.tsx          # Plugin 装备面板
│       │   │   │   └── UserBadge.tsx            # 用户身份徽章
│       │   │   └── layout/             # 通用布局组件
│       │   │       ├── Sidebar.tsx
│       │   │       ├── Header.tsx
│       │   │       └── CyberBackground.tsx     # 赛博朋克动态背景（纯 CSS 网格线）
│       │   │
│       │   ├── lib/
│       │   │   ├── supabase/
│       │   │   │   ├── client.ts       # Supabase 浏览器端 Client
│       │   │   │   ├── server.ts       # Supabase 服务端 Client (SSR)
│       │   │   │   ├── middleware.ts    # Auth Session 刷新中间件
│       │   │   │   └── types.ts        # Supabase 自动生成的 TypeScript 类型
│       │   │   ├── utils.ts            # 工具函数（cn() 等）
│       │   │   └── constants.ts        # 全局常量（Protocol 映射、颜色等）
│       │   │
│       │   ├── hooks/
│       │   │   ├── useAuth.ts          # 认证状态 Hook
│       │   │   ├── useProfile.ts       # 用户画像 Hook
│       │   │   ├── useVote.ts          # 投票逻辑 Hook
│       │   │   └── useRealtime.ts      # Supabase Realtime 订阅 Hook
│       │   │
│       │   └── styles/
│       │       └── cyber-theme.ts      # 赛博朋克主题色值 & Tailwind 扩展
│       │
│       ├── middleware.ts               # Next.js 中间件（Auth 路由守卫）
│       ├── next.config.ts
│       ├── tailwind.config.ts
│       ├── tsconfig.json
│       ├── components.json             # shadcn/ui 配置
│       └── package.json
│
├── services/
│   └── matching/                       # FastAPI 匹配微服务（从 Hackathon 迁移）
│       ├── app/
│       │   ├── main.py                 # FastAPI 入口
│       │   ├── routers/
│       │   │   └── match_router.py     # /api/v1/match/scan 路由
│       │   ├── services/
│       │   │   ├── match_engine.py     # 从 FindBud 迁移的核心数学函数
│       │   │   └── match_service.py    # 从 FindBud 迁移的匹配逻辑（泛化版）
│       │   ├── schemas/
│       │   │   └── match_schemas.py    # Pydantic 请求/响应模型
│       │   └── core/
│       │       └── config.py           # 环境变量
│       ├── requirements.txt
│       ├── Dockerfile                  # Render 部署用
│       └── render.yaml                 # Render Blueprint
│
├── supabase/
│   ├── migrations/
│   │   └── 00001_init_schema.sql       # 上述所有建表 SQL
│   ├── functions/
│   │   └── oracle-respond/
│   │       └── index.ts                # Edge Function: AI Oracle 回复
│   ├── seed.sql                        # 种子数据（Hub、Sub-node 预设）
│   └── config.toml                     # Supabase 本地开发配置
│
├── .env.example                        # 环境变量模板
├── .gitignore
├── package.json                        # Monorepo 根 (pnpm workspace)
├── pnpm-workspace.yaml
├── turbo.json                          # Turborepo 配置（可选，简化构建）
└── README.md
```

---

## 五、Study Buddy 匹配服务集成方案

### 5.1 架构决策：FastAPI 微服务 on Render（推荐）

| 方案 | 优点 | 缺点 | 推荐度 |
|------|------|------|--------|
| **FastAPI on Render** | 直接复用 Hackathon Python 代码，改动最小 | 额外一个服务要维护 | ⭐⭐⭐⭐⭐ |
| Supabase Edge Function (Deno) | 全部在 Supabase 生态内 | 需要把 Python 算法重写为 TypeScript | ⭐⭐ |
| Vercel Serverless Function | 和前端同一平台 | Python 支持有限，冷启动慢 | ⭐⭐ |

**结论：** 你们已经有一个经过验证的 Python 匹配引擎（`match_engine.py` + `match_service.py`），重写成 TypeScript 是浪费时间。直接包一层轻量 FastAPI 路由，部署到 Render 免费实例（750小时/月）。

### 5.2 API 路由设计

#### `POST /api/v1/match/scan` — 发起雷达扫描

**Request:**

```json
{
  "user_id": "uuid-of-requesting-user",
  "protocols": ["seeker", "builder"],
  "plugins": ["Advanced Calculus Engine v2.0", "Python Data-Miner"],
  "skill_vector": {
    "modeling": 7,
    "coding": 9,
    "writing": 4
  },
  "personality": {
    "leader": 3,
    "executor": 8,
    "supporter": 5
  },
  "preferences": {
    "hub_filter": "engineering",
    "max_results": 3
  }
}
```

**Response (200):**

```json
{
  "scan_id": "uuid-of-this-scan",
  "status": "completed",
  "matches": [
    {
      "user_id": "matched-user-uuid-1",
      "rank": 1,
      "match_score": 0.9234,
      "match_reasons": {
        "summary": "技能正交互补，性格角色互补，协作摩擦小",
        "dimension_breakdown": [
          { "dimension": "技能向量", "score": 0.92, "comment": "技能正交互补" },
          { "dimension": "性格动能因子", "score": 0.85, "comment": "性格角色互补" },
          { "dimension": "综合实力", "score": 0.78, "comment": "实力较为接近" }
        ]
      },
      "profile_snapshot": {
        "username": "cyber_coder_42",
        "display_name": "张三",
        "protocol": "oracle",
        "plugins": ["Linear Algebra Core v3.1"],
        "hub": "Engineering"
      }
    },
    { "user_id": "...", "rank": 2, "..." : "..." },
    { "user_id": "...", "rank": 3, "..." : "..." }
  ],
  "diversity_guaranteed": true,
  "scanned_at": "2025-05-20T14:30:00Z"
}
```

**Error (4xx/5xx):**

```json
{
  "error": "insufficient_candidates",
  "message": "候选池中用户不足，请稍后再试",
  "min_required": 3,
  "available": 1
}
```

### 5.3 前端调用流程

```
用户点击 "Initiate Radar Scan"
        │
        ▼
前端显示 CSS 雷达脉冲动画（RadarPulse.tsx）
        │
        ▼
fetch('/api/match', { method: 'POST', body: userProfile })
        │  （Next.js API Route 代理转发到 FastAPI）
        ▼
FastAPI 从 Supabase DB 读取候选用户 → 运行匹配算法 → 返回 Top 3
        │
        ▼
前端收到结果 → 地图上高亮匹配节点（React Flow 节点动画）
```

### 5.4 从 Hackathon 迁移清单

从 `UNNC 30H Hackathon/FindBud_APP/backend/app/services/` 迁移以下文件：

| 源文件 | 迁移到 | 改动点 |
|--------|--------|--------|
| `match_engine.py` | `services/matching/app/services/match_engine.py` | 无需改动（纯数学函数） |
| `match_service.py` | `services/matching/app/services/match_service.py` | 泛化 `UserProfile` 字段名，匹配 NeTrix 的 `skill_vector`/`personality` Schema |

核心改动仅为：将 `UserProfile` 的硬编码字段（如 `skill_modeling`）改为从 JSONB 动态读取，以适配 NeTrix 的灵活标签系统。

---

## 六、初始化命令（Init Commands）

### 6.0 环境要求

| 工具 | 最低版本 | 安装方式 |
|------|---------|---------|
| **Node.js** | ≥ 18.17 | https://nodejs.org (LTS) |
| **pnpm** | ≥ 9.0 | `npm install -g pnpm` |
| **Python** | ≥ 3.11 | https://python.org |
| **Supabase CLI** | ≥ 1.200 | `brew install supabase/tap/supabase` |
| **Git** | ≥ 2.40 | 系统自带或 brew install git |

### 6.1 创建 GitHub 仓库 & 初始化 Monorepo

```bash
# 1. 创建仓库（在 GitHub 上创建后 clone，或本地 init）
mkdir netrix && cd netrix
git init

# 2. 初始化 pnpm workspace
pnpm init

# 3. 创建 workspace 配置
cat > pnpm-workspace.yaml << 'EOF'
packages:
  - "apps/*"
  - "services/*"
EOF
```

### 6.2 初始化 Next.js 前端

```bash
# 4. 创建 Next.js 应用
pnpm create next-app@latest apps/web \
  --typescript \
  --tailwind \
  --eslint \
  --app \
  --src-dir \
  --import-alias "@/*" \
  --use-pnpm

# 5. 进入前端目录，安装核心依赖
cd apps/web

# shadcn/ui 初始化
pnpm dlx shadcn@latest init

# 安装必要的 shadcn 组件
pnpm dlx shadcn@latest add button card badge input textarea \
  avatar dropdown-menu dialog tabs separator scroll-area \
  toast tooltip popover command

# React Flow（节点地图）
pnpm add @xyflow/react

# Supabase Client
pnpm add @supabase/supabase-js @supabase/ssr

# Markdown 渲染（论坛帖子内容）
pnpm add react-markdown remark-gfm

# 日期处理
pnpm add date-fns

cd ../..
```

### 6.3 初始化 Supabase

```bash
# 6. 初始化 Supabase（在项目根目录）
supabase init

# 7. 链接到远程 Supabase 项目（需要先在 supabase.com 创建项目）
supabase link --project-ref <YOUR_PROJECT_REF>

# 8. 创建第一个 migration
supabase migration new init_schema
# 将上面 "三、数据库 Schema" 的 SQL 粘贴到生成的文件中

# 9. 推送 migration 到远程
supabase db push

# 10. 生成 TypeScript 类型（前端直接用）
supabase gen types typescript --linked > apps/web/src/lib/supabase/types.ts

# 11. 创建 AI Oracle Edge Function
supabase functions new oracle-respond
```

### 6.4 初始化 FastAPI 匹配微服务

```bash
# 12. 创建匹配服务目录
mkdir -p services/matching/app/{routers,services,schemas,core}

# 13. 创建 Python 虚拟环境
cd services/matching
python -m venv .venv
source .venv/bin/activate  # macOS/Linux
# .venv\Scripts\activate   # Windows

# 14. 安装依赖
cat > requirements.txt << 'EOF'
fastapi>=0.115.0
uvicorn[standard]>=0.30.0
pydantic>=2.9.0
pydantic-settings>=2.4.0
httpx>=0.27.0
supabase>=2.0.0
python-dotenv>=1.0.0
EOF

pip install -r requirements.txt

cd ../..
```

### 6.5 环境变量模板

```bash
# 15. 创建 .env.example
cat > .env.example << 'EOF'
# ===== Supabase =====
NEXT_PUBLIC_SUPABASE_URL=https://xxx.supabase.co
NEXT_PUBLIC_SUPABASE_ANON_KEY=eyJxxx...
SUPABASE_SERVICE_ROLE_KEY=eyJxxx...

# ===== AI Oracle (Edge Function) =====
OPENAI_API_KEY=sk-xxx
AI_MODEL_NAME=gpt-4o-mini

# ===== Matching Service =====
MATCHING_SERVICE_URL=https://netrix-matching.onrender.com
MATCHING_API_KEY=your-internal-api-key

# ===== 匹配微服务自身 (services/matching/.env) =====
SUPABASE_URL=https://xxx.supabase.co
SUPABASE_SERVICE_KEY=eyJxxx...
EOF

# 16. 配置 .gitignore
cat > .gitignore << 'EOF'
node_modules/
.next/
.env
.env.local
.venv/
__pycache__/
*.pyc
.DS_Store
.vercel
.supabase
dist/
EOF
```

### 6.6 一键验证

```bash
# 启动前端 dev server
cd apps/web && pnpm dev

# 另一个终端：启动匹配微服务
cd services/matching && source .venv/bin/activate && uvicorn app.main:app --reload --port 8001

# 另一个终端：启动 Supabase 本地开发（可选，也可直接用云端）
supabase start
```

---

## 七、开发策略：做 vs 假装做（Build vs Fake）

> 这是本文档**最关键**的一节。4周时间，你们必须无情地砍掉一切不影响路演效果的功能。

### 必须完整实现（路演核心演示流程）

| 功能 | 原因 | 预估工时 |
|------|------|---------|
| **Supabase Auth 登录/注册** | 必须有真实用户系统，评委会现场试用 | 0.5天 |
| **Cyber Neural Network 节点地图（React Flow）** | 这是你们的**视觉核武器**，第一印象决定生死 | 3-4天 |
| **Protocol 切换 + Plugin 展示** | 身份系统是核心差异化卖点，必须能演示 | 1天 |
| **Knowledge Hub 帖子列表 + 详情 + 评论** | 论坛基本 CRUD 是产品根基 | 3天 |
| **Summon Oracle 按钮 → AI 回复** | H2A2H 是你们的**灵魂叙事**，必须现场演示"召唤AI → AI回复 → 人类纠错" | 2天 |
| **上下投票系统** | 证明 AI 和人类答案"平权"的核心机制 | 1天 |
| **Study Buddy 雷达扫描动画 + 结果展示** | 展示匹配算法的存在和效果 | 2天 |
| **资源中心基础列表** | 路演中一笔带过即可，但需要有页面 | 1天 |

### 可以 Hardcode / Fake 的（路演中看起来正常即可）

| 功能 | 如何 Fake | 省多少时间 |
|------|-----------|-----------|
| **Hub / Sub-node 数据** | 直接在 `seed.sql` 中硬编码 5-8 个学院节点和 15-20 个话题节点，不做后台管理 | 省2天 |
| **React Flow 节点布局** | 手动指定所有节点的 x/y 坐标（在 seed 数据中），**不做自动布局算法** | 省2天 |
| **Plugin 系统** | 准备一个固定的 Plugin 列表（约20个），用户只能从中选择，**不做自由创建** | 省1天 |
| **XP 积分 / 等级** | 数据库字段留着，但 MVP 阶段 XP 永远显示 0，**不实现积分规则** | 省2天 |
| **通知系统** | 完全不做。路演时说"我们的架构已支持 Realtime 通知，下一版迭代" | 省3天 |
| **搜索功能** | 用 Supabase 的 `ilike` 做最简单的标题模糊搜索，**不做全文检索** | 省2天 |
| **用户头像** | 用 `DiceBear` API 根据 username 生成随机赛博朋克风头像，**不做上传** | 省1天 |
| **移动端适配** | 只做桌面端。路演用笔记本电脑投屏展示 | 省3天 |
| **匹配算法候选池** | 如果数据库用户不够，在 `seed.sql` 中预埋 30-50 个虚拟用户画像作为匹配候选池 | 省0.5天 |
| **AI Whispers Gallery** | 只做链接提交 + 展示，**不做链接预览/内容抓取** | 省1天 |
| **Markdown 编辑器** | 用 `<textarea>` + `react-markdown` 渲染即可，**不做富文本 WYSIWYG** | 省2天 |
| **邮箱验证 / 密码重置** | 关闭 Supabase 邮箱确认流程，注册后直接登录 | 省0.5天 |

### 绝对不做（留给暑假）

| 功能 | 为什么不做 |
|------|-----------|
| 实时聊天 / DM 私信 | 工程量巨大，与路演核心叙事无关 |
| AI Agent 自动巡逻发帖 | 复杂的触发逻辑，MVP 只做被动"召唤" |
| RLHF 数据飞轮 / 模型微调 | 路演讲概念即可，实现需要数月 |
| 多语言国际化 (i18n) | MVP 只做英文 UI（UNNC 学生习惯英文界面） |
| 管理后台 / 内容审核 | 用 Supabase Dashboard 直接操作数据库代替 |
| 支付系统 / Freemium | 商业化留给融资后 |
| PWA / 原生 App | 纯 Web，路演够用 |

---

## 八、四周冲刺排期建议

### Week 1: 基建 & 骨架（Infrastructure Sprint）

| 天数 | 任务 | 负责人建议 |
|------|------|-----------|
| Day 1-2 | 初始化仓库、Supabase 项目、数据库 Migration、Vercel 部署流水线 | 全员 |
| Day 3-4 | Supabase Auth 接入 + 路由守卫 + 赛博朋克主题基础样式 | 前端主力 |
| Day 5-6 | React Flow 节点地图 MVP（hardcode 节点数据，实现点击跳转） | 前端主力 |
| Day 7 | 匹配微服务搭建 + Render 部署 + 联调测试 | 后端/算法 |

### Week 2: 核心模块开发（Feature Sprint 1）

| 天数 | 任务 | 负责人建议 |
|------|------|-----------|
| Day 8-10 | Knowledge Hub: 帖子 CRUD + 评论系统 + 投票 | 前端 + 后端 |
| Day 11-12 | Summon Oracle: Edge Function + AI 回复写入数据库 + 前端集成 | 后端/AI |
| Day 13-14 | Protocol 切换 + Plugin 装备 + 用户画像页面 | 前端 |

### Week 3: 完善 & 联调（Feature Sprint 2）

| 天数 | 任务 | 负责人建议 |
|------|------|-----------|
| Day 15-16 | Study Buddy: 雷达脉冲动画 + 匹配 API 联调 + 地图节点高亮 | 前端 + 后端 |
| Day 17-18 | 资源中心 + AI Whispers Gallery 基础版 | 前端 |
| Day 19-21 | 全面联调、Bug修复、种子数据灌入、视觉打磨（动画/渐变/发光效果） | 全员 |

### Week 4: 打磨 & 路演（Polish & Demo Sprint）

| 天数 | 任务 | 负责人建议 |
|------|------|-----------|
| Day 22-23 | 路演演示流程编排：确定点击路径、准备演示账号和数据 | 全员 |
| Day 24-25 | UI 微调、赛博朋克动效打磨、Loading States 完善 | 前端 |
| Day 26-27 | 压力测试、Edge Case 处理、部署最终版到 Vercel | 全员 |
| Day 28 | **路演彩排！录制 Demo Video 作为备份** | 全员 |

---

## 九、开发人员分工建议

基于团队 3 位技术成员（EEE/CS/Math 专业）：

| 角色 | 建议人选 | 职责范围 |
|------|---------|---------|
| **前端主力 (Frontend Lead)** | CS 专业成员 | React Flow 节点地图、shadcn/ui 组件开发、页面路由、赛博朋克视觉效果 |
| **全栈/后端 (Fullstack + BaaS)** | EEE 专业成员 | Supabase 配置（Auth/RLS/Migration）、Edge Function (AI Oracle)、Next.js API Routes |
| **算法工程师 (Algorithm + Integration)** | Math 专业成员 | 匹配算法从 Hackathon 迁移 + 泛化、FastAPI 微服务、数据库种子数据、匹配 API 联调 |

**协作规则：**
- 使用 **GitHub Flow**：`main` 分支保护，所有功能通过 PR 合并
- 每个 PR 至少一人 Code Review（可以快速 review，不要成为瓶颈）
- 每天 15 分钟站会同步进度（微信群即可）

---

## 附录：赛博朋克主题 CSS 变量参考

```css
:root {
  /* 核心色板 — Hacker/Cyberpunk but Clean */
  --neon-cyan:    #00F0FF;
  --neon-magenta: #FF00E5;
  --neon-gold:    #FFD700;
  --neon-green:   #39FF14;

  /* Protocol 对应色 */
  --protocol-seeker:  #00A8FF;  /* 蓝 */
  --protocol-oracle:  #FFD700;  /* 金 */
  --protocol-builder: #39FF14;  /* 绿 */
  --protocol-stealth: #555555;  /* 灰 */

  /* 背景 */
  --bg-primary:   #0A0A0F;
  --bg-secondary: #12121A;
  --bg-card:      #1A1A2E;

  /* 文字 */
  --text-primary:   #E0E0FF;
  --text-secondary: #8888AA;

  /* 发光效果 */
  --glow-cyan:    0 0 10px #00F0FF, 0 0 30px rgba(0, 240, 255, 0.3);
  --glow-magenta: 0 0 10px #FF00E5, 0 0 30px rgba(255, 0, 229, 0.3);
}
```

---

> **文档版本：** v1.0 | **撰写时间：** 2025年4月25日 | **下次更新：** 技术组例会后
