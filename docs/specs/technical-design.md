# 祝福墙项目 - 技术设计方案

> 基于 LeanCloud 部署，面向开发实施的完整技术方案

## 1. 项目结构

```
blessing-wall/
├── public/
│   └── favicon.ico
├── src/
│   ├── main.tsx                    # 入口
│   ├── App.tsx                     # 路由配置
│   ├── config/
│   │   └── leancloud.ts            # LeanCloud SDK 初始化
│   ├── models/
│   │   ├── wall.ts                 # Wall 类操作封装
│   │   ├── message.ts              # Message 类操作封装
│   │   ├── like.ts                 # Like 类操作封装
│   │   └── admin.ts                # 管理员登录（云函数调用）
│   ├── hooks/
│   │   ├── useAuth.ts              # 管理员登录状态
│   │   ├── useWall.ts              # 墙数据查询
│   │   └── useMessages.ts          # 留言列表 + 发布 + 点赞
│   ├── components/
│   │   ├── layout/
│   │   │   ├── Header.tsx          # 顶部导航
│   │   │   └── Footer.tsx
│   │   ├── wall/
│   │   │   ├── WallCard.tsx        # 墙列表卡片
│   │   │   ├── WallGrid.tsx        # 墙网格布局
│   │   │   └── WallSearch.tsx      # 搜索框（>20个墙时显示）
│   │   ├── message/
│   │   │   ├── MessageCard.tsx     # 留言卡片
│   │   │   ├── MessageGrid.tsx     # 留言网格布局
│   │   │   ├── LikeButton.tsx      # 点赞按钮
│   │   │   └── PublishModal.tsx    # 发布留言弹窗
│   │   ├── download/
│   │   │   ├── DownloadModal.tsx   # 下载弹窗（左右布局）
│   │   │   ├── TemplateSelector.tsx # 模板选择网格
│   │   │   └── CanvasPreview.tsx   # Canvas 实时预览
│   │   ├── admin/
│   │   │   ├── WallForm.tsx        # 创建/编辑墙表单
│   │   │   └── ConfirmDialog.tsx   # 删除二次确认
│   │   └── common/
│   │       ├── Modal.tsx           # 通用弹窗
│   │       ├── Toast.tsx           # 提示信息
│   │       └── Empty.tsx           # 空状态
│   ├── pages/
│   │   ├── HomePage.tsx            # 首页 - 墙列表
│   │   ├── WallPage.tsx            # 墙详情 - 留言展示
│   │   ├── LoginPage.tsx           # 管理员登录
│   │   └── AdminPage.tsx           # 管理面板
│   ├── templates/
│   │   ├── index.ts                # 模板注册
│   │   ├── base.ts                 # 基础模板接口
│   │   ├── warmFloral.ts           # 温馨花卉
│   │   ├── minimalArt.ts           # 极简文艺
│   │   ├── starryNight.ts          # 星空许愿
│   │   ├── moonLoveLetter.ts       # 月夜情书
│   │   ├── ribbonLetter.ts         # 丝带信笺
│   │   ├── birthdayParty.ts        # 生日派对
│   │   ├── cuteFun.ts              # 童趣可爱
│   │   └── vintageBirthday.ts      # 复古生日
│   ├── utils/
│   │   ├── fingerprint.ts          # 访客指纹生成
│   │   ├── slug.ts                 # slug 生成（nanoid/自定义）
│   │   └── time.ts                 # 时间格式化
│   └── styles/
│       └── globals.css             # Tailwind 入口
├── cloud/
│   └── functions/
│       ├── adminLogin.js           # 管理员登录验证
│       ├── createMessage.js        # 创建留言（含限频）
│       ├── toggleLike.js           # 点赞/取消点赞（含去重）
│       └── initAdmin.js            # 初始化管理员账号（部署时运行）
├── .env.example                    # 环境变量模板
├── index.html
├── package.json
├── tsconfig.json
├── tailwind.config.js
└── vite.config.ts
```

## 2. LeanCloud 数据模型设计

### 2.1 Wall（祝福墙）

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| title | String | ✅ | 墙标题 |
| description | String | ✅ | 墙描述/副标题 |
| protagonist | String | ❌ | 祝福送给谁，空则取 title |
| themeColor | String | ✅ | 主题色 hex，默认 `#FF6B6B` |
| slug | String | ✅ | URL 友好标识，唯一索引 |
| status | String | ✅ | `open` / `closed` / `hidden` |
| likeCount | Number | ✅ | 总点赞数（冗余），默认 0 |
| messageCount | Number | ✅ | 留言总数（冗余），默认 0 |

**ACL 策略：**
- 创建时设置 `publicRead: true, publicWrite: false`
- 仅管理员可写（通过 `role:admin`）
- `slug` 字段建立唯一索引

**索引：**
```json
{ "slug": 1 }          // 唯一索引
{ "status": 1, "createdAt": -1 }  // 首页列表查询
```

### 2.2 Message（留言）

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| wall | Pointer\<Wall\> | ✅ | 所属墙 |
| nickname | String | ✅ | 昵称，空字符串表示匿名 |
| content | String | ✅ | 祝福内容，最大 1000 字 |
| likeCount | Number | ✅ | 点赞数，默认 0 |

**ACL 策略：**
- `publicRead: true, publicWrite: false`
- 仅通过云函数创建（含限频）
- 管理员可删除

**索引：**
```json
{ "wall": 1, "createdAt": -1 }  // 墙详情页查询
```

### 2.3 Like（点赞）

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| message | Pointer\<Message\> | ✅ | 所属留言 |
| wall | Pointer\<Wall\> | ✅ | 冗余关联墙（方便统计）
| fingerprint | String | ✅ | 访客标识 |
| IP | String | ✅ | 客户端 IP |

**ACL 策略：**
- `publicRead: true, publicWrite: false`
- 仅通过云函数操作（含去重检查）

**索引（联合唯一）：**
```json
{ "message": 1, "fingerprint": 1 }  // 防重复点赞
```

### 2.4 AdminConfig（管理员配置）

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| username | String | ✅ | 管理员用户名 |
| passwordHash | String | ✅ | SHA-256 哈希 |
| configKey | String | ✅ | 固定值 `"main"`，唯一索引 |

**ACL 策略：**
- `publicRead: false, publicWrite: false`
- 仅云函数可访问（使用 masterKey）

## 3. 云函数设计

### 3.1 `adminLogin` - 管理员登录

```
入参：{ username: string, password: string }
出参：{ success: boolean, token?: string, error?: string }
```

- 使用 masterKey 查询 AdminConfig
- SHA-256 哈希比对密码
- 成功返回 LeanCloud sessionToken
- 失败返回错误信息

### 3.2 `createMessage` - 创建留言（含限频）

```
入参：{ wallId: string, nickname: string, content: string, fingerprint: string }
出参：{ success: boolean, message?: object, error?: string }
```

- 校验：内容非空、≤1000 字
- **限频**：查最近 60 秒内同 IP 创建的 Message 数量，≥5 则拒绝
- 创建 Message 对象
- 原子递增 Wall 的 `messageCount`

### 3.3 `toggleLike` - 点赞/取消点赞

```
入参：{ messageId: string, wallId: string, fingerprint: string }
出参：{ success: boolean, liked: boolean, likeCount: number }
```

- 查询 Like 表：同 message + 同 fingerprint
- 存在 → 删除 Like，递减 likeCount
- 不存在 → 创建 Like，递增 likeCount
- 同步更新 Wall.likeCount（可选，仅在墙详情页需要总数时）

### 3.4 `initAdmin` - 初始化管理员（一次性）

```
入参：{ username: string, password: string }
出参：{ success: boolean }
```

- 使用 masterKey
- 检查是否已存在，已存在则跳过
- 创建 AdminConfig 记录

## 4. 前端架构设计

### 4.1 路由规划

```tsx
<Routes>
  <Route path="/" element={<HomePage />} />
  <Route path="/login" element={<LoginPage />} />
  <Route path="/admin" element={<AdminPage />}>
    <Route path="walls/new" element={<WallForm />} />
  </Route>
  <Route path="/w/:slug" element={<WallPage />} />
  <Route path="*" element={<Navigate to="/" />} />
</Routes>
```

### 4.2 LeanCloud SDK 初始化

```ts
// src/config/leancloud.ts
import AV from 'leancloud-storage';

const initLeanCloud = () => {
  AV.init({
    appId: import.meta.env.VITE_LC_APP_ID,
    appKey: import.meta.env.VITE_LC_APP_KEY,
    serverURL: import.meta.env.VITE_LC_SERVER_URL,
  });
};
```

### 4.3 状态管理

**不引入 Redux/Zustand**，使用 React 内置方案：
- **全局状态**：`React.createContext` — 管理员登录态（currentUser + token）
- **页面状态**：`useState` / `useReducer` — 各页面自行管理
- **服务端状态**：LeanCloud 查询直接在 hooks 中调用，用 `useState` + `useEffect` 管理

### 4.4 核心 Hooks

```ts
// useAuth - 管理员状态
const useAuth = () => {
  // 从 localStorage 读取 token
  // 提供 login / logout / isAuthenticated
};

// useWall(slug) - 墙详情
const useWall = (slug: string) => {
  // 查询 Wall，返回 { wall, loading, error }
};

// useMessages(wallId) - 留言列表 + 操作
const useMessages = (wallId: string) => {
  // 查询留言列表（按 createdAt 倒序）
  // 提供 publish / like / refresh
};
```

### 4.5 组件设计要点

| 组件 | 关键点 |
|------|--------|
| `MessageCard` | 接收 themeColor，卡片边框/装饰跟随主题色；点赞按钮需传 fingerprint |
| `DownloadModal` | 左右布局，左侧 TemplateSelector 3 列网格，右侧 CanvasPreview 实时渲染 |
| `CanvasPreview` | 每次模板切换时重新绘制 Canvas，接收 message + wall 数据 |
| `PublishModal` | 昵称可选（空=匿名），内容 textarea 带字数统计（最大1000） |
| `AdminPage` | 墙列表表格 + 操作按钮（关闭/隐藏/删除），需登录态守卫 |

## 5. 部署方案

### 5.1 LeanCloud 静态托管配置

```
1. LeanCloud 控制台 → 存储 → 静态网站 → 启用
2. 构建产物上传到 ./dist 目录
3. 初始域名：https://stg-xxx.lc-cn-n1-shared.com（LeanCloud 提供）
4. 后续可绑定已备案自定义域名
```

### 5.2 SPA 路由处理

LeanCloud 静态托管默认不支持 SPA history 模式回退，需要：

**方案：将所有路由映射到 index.html**
- 在 LeanCloud 控制台设置「回退文档」为 `index.html`
- 或使用 hash 路由（`HashRouter`）替代 `BrowserRouter`，避免服务端配置问题

> **推荐使用 HashRouter**，零配置兼容 LeanCloud 托管，URL 格式如 `/#/w/my-wall`

### 5.3 环境变量

```bash
# .env.example
VITE_LC_APP_ID=your_app_id
VITE_LC_APP_KEY=your_app_key
VITE_LC_SERVER_URL=https://your-app.lc-cn-n1-shared.com
```

### 5.4 构建与部署流程

```bash
# 1. 安装依赖
npm install

# 2. 构建
npm run build

# 3. 部署（使用 LeanCloud CLI）
npm install -g leancloud-cli
lean switch --region cn-n1
lean deploy --prod

# 或手动上传 dist/ 到 LeanCloud 控制台
```

### 5.5 LeanCloud 控制台配置清单

- [ ] 创建应用（选择国内节点 cn-n1）
- [ ] 启用静态网站托管
- [ ] 创建 Wall、Message、Like、AdminConfig 四个 Class
- [ ] 设置各 Class 的 ACL 默认值
- [ ] 创建 `slug` 唯一索引
- [ ] 创建联合索引 `(message, fingerprint)`
- [ ] 上传云函数（4 个）
- [ ] 运行 `initAdmin` 云函数初始化管理员账号
- [ ] 设置环境变量
- [ ] 设置回退文档为 index.html（如使用 BrowserRouter）

## 6. 关键技术点

### 6.1 Canvas 卡片生成

```ts
// templates/base.ts - 模板接口
interface CardTemplate {
  id: string;
  name: string;
  thumbnail: string;    // 缩略图描述
  draw: (ctx: CanvasRenderingContext2D, data: CardData) => void;
}

interface CardData {
  protagonist: string;  // 收件人
  nickname: string;     // 留言者
  content: string;      // 祝福内容
  date: string;         // 发布日期
  themeColor: string;   // 主题色
}
```

- 每个模板实现 `draw` 方法，纯 Canvas 2D API 绘制
- 预览时使用 400×560 的 Canvas，下载时使用 800×1120 高清输出
- 下载通过 `canvas.toBlob()` → `URL.createObjectURL()` → `<a>` 标签触发

### 6.2 Fingerprint 方案

```ts
// utils/fingerprint.ts
const getFingerprint = (): string => {
  const stored = localStorage.getItem('bw_fp');
  if (stored) return stored;
  
  // 生成基于浏览器特征的指纹
  const fp = crypto.randomUUID(); // 或基于 canvas/audio 指纹
  localStorage.setItem('bw_fp', fp);
  return fp;
};
```

- 使用 `localStorage` 存储唯一标识
- 结合服务端 IP 做双重校验
- 清除浏览器数据可重新生成（可接受的防刷粒度）

### 6.3 SPA 路由在 LeanCloud 托管的处理

**推荐方案：HashRouter**

```tsx
import { HashRouter } from 'react-router-dom';

// App.tsx
<HashRouter>
  <Routes>...</Routes>
</HashRouter>
```

优势：
- 无需服务端配置回退
- LeanCloud 静态托管直接兼容
- URL 格式：`/#/` `/#/w/my-wall` `/#/admin`

### 6.4 LeanCloud SDK 体积优化

```ts
// 按需引入，避免全量加载
import { Query, Object } from 'leancloud-storage';
// 或使用更轻量的 leancloud-realtime-lite
```

Vite 构建时 LeanCloud SDK 会被 tree-shaking，实际打包体积约 40-60KB gzipped。

### 6.5 安全注意事项

| 风险 | 对策 |
|------|------|
| 前端直接操作数据 | ACL 限制写入，核心操作走云函数 |
| 管理员密码泄露 | 密码仅存储 SHA-256 哈希，云函数比对 |
| XSS（用户输入内容） | React 默认转义 + 限制纯文本+emoji |
| 爬虫批量抓取 | LeanCloud 免费层自带速率限制 |
| 恶意刷点赞 | fingerprint + IP 联合去重 |
