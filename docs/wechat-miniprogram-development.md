# AI 算卦微信小程序开发文档

> 版本：v0.1（草案）  
> 面向角色：前端开发、云函数开发、测试、产品设计  
> 目标：指导团队在 Next.js Web 版本基础上实现并维护微信小程序版本的 AI 算卦产品。

## 1. 项目概述

- **核心功能**：三枚硬币六次卜筮生成卦象、传统周易文本展示、AI 解读、历史记录、主题切换。
- **总体架构**：微信小程序（前端） + 云开发（云函数 + 云数据库 + 云存储） + OpenAI API。
- **设计原则**：
  - 功能与 Web 版保持一致，界面遵循微信小程序规范。
  - 所有敏感逻辑（OpenAI API 调用、敏感数据存储）下沉至云端。
  - 支持暗/亮主题，自适应不同屏幕尺寸。

## 2. 系统架构

```
用户操作 → 小程序前端 → 云函数 API → OpenAI 服务
                                ↓
                       云数据库 / 云存储
```

- **小程序层**：负责 UI 展现、交互逻辑、状态管理、动画效果。
- **云函数层**：封装卦象数据读取、AI 解读、记录存储等后端能力。
- **数据层**：云数据库存储卦象元数据、解读缓存、用户记录；云存储托管卦象原文、插画等静态资源。
- **外部服务**：OpenAI API（模型可配置，默认 `gpt-4o-mini` 或项目约定模型）。

## 3. 目录结构建议

```
project-root/
├── miniprogram/                      # 小程序主体
│   ├── app.ts                        # 全局逻辑（TypeScript 可选）
│   ├── app.json                      # 全局配置（页面、网络、主题等）
│   ├── app.wxss                      # 全局样式
│   ├── sitemap.json                  # 页面索引声明
│   ├── pages/
│   │   ├── index/                    # 主流程页面
│   │   │   ├── index.wxml
│   │   │   ├── index.wxss
│   │   │   ├── index.ts
│   │   │   └── index.json
│   │   ├── history/                  # 历史记录页面
│   │   └── settings/                 # 配置与说明页面
│   ├── components/                   # 自定义组件
│   │   ├── coin-flip/
│   │   ├── hexagram-view/
│   │   ├── ai-result/
│   │   └── theme-toggle/
│   ├── utils/
│   │   ├── hexagram.ts               # 卦象计算逻辑
│   │   ├── animation.ts              # 动画辅助函数
│   │   └── storage.ts                # 本地缓存封装
│   ├── styles/                       # 通用 WXSS 变量、mixin
│   └── images/                       # 静态资源
└── cloudfunctions/
    ├── getAIInterpretation/          # 调用 OpenAI 输出解读
    │   ├── index.js
    │   └── package.json
    ├── getHexagramData/             # 读取卦象原文与元信息
    └── saveDivinationRecord/        # 记录存储 / 查询
```

> 注：目录可按团队习惯微调，但小程序端与云函数需分别明确职责。

## 4. 环境与工具

- **基础要求**：
  - Node.js ≥ 18（本地脚本与云函数开发）
  - pnpm ≥ 8（可选，用于管理云函数依赖）
  - 微信开发者工具最新版（启用“使用 npm 模块”）
  - 微信小程序 AppID（开发环境可使用测试号）
  - OpenAI API Key（存放于云函数环境变量）
- **建议扩展**：
  - TypeScript、ESLint（小程序端代码质量）
  - Prettier（统一格式）
  - 微信云开发 CLI（批量部署）

## 5. 初始化流程

1. **创建云开发环境**  
   - 在微信开发者工具中登录 → 选择“云开发” → 创建环境（记录 `envId`）。
2. **配置小程序项目**  
   - 新建项目并指向 `project-root/miniprogram` 目录，启用 TypeScript/ESLint。
   - 在 `app.ts` 中调用 `wx.cloud.init({ env: 'envId', traceUser: true })`。
3. **初始化云函数**  
   - 在 `cloudfunctions/` 内运行 `pnpm install`（或 `npm install`）安装依赖。
   - 在开发者工具中关联云函数目录，上传并部署。
4. **导入基础数据**  
   - 将 Web 版本 `lib/data/gua-list.json`、`gua-index.json` 导入云数据库集合，如 `gua_list`、`gua_index`。
   - 卦象原文 Markdown 可上传至云存储或转为数据库记录。
5. **配置安全域名**  
   - 仅调用云函数，无需额外 HTTPS 域名；如需访问外部资源，须在小程序后台配置合法域名。

## 6. 功能模块说明

### 6.1 小程序页面
- `pages/index`：主卜筮流程，包含硬币动画、问题输入、卦象展示、AI 解读。
- `pages/history`：用户历史记录列表，可查看详情或删除。
- `pages/settings`：主题切换、免责声明、帮助文档入口。

### 6.2 组件职责
- `coin-flip`：封装三枚硬币的动画，暴露 `start()`、`stop()`、`onComplete` 回调。
- `hexagram-view`：展示本卦与变卦，支持阴阳线动态渲染。
- `ai-result`：呈现 AI 流式/分段结果，包含状态指示、错误提示。
- `theme-toggle`：读取系统主题，支持手动切换并写入本地缓存。

### 6.3 云函数能力
- `getHexagramData`：根据卦象编号返回标题、卦辞、爻辞、象辞等文本。
- `getAIInterpretation`：入参包含用户问题、卦象数据、上下文记录 ID；调用 OpenAI 并返回分段结果。
- `saveDivinationRecord`：写入或更新用户记录，同时提供分页查询。

## 7. 业务流程

1. 用户输入问题并开始卜筮。
2. `coin-flip` 组件执行六次动画，生成六爻结果。
3. `hexagram.ts` 根据六爻计算本卦、变卦、动爻信息，返回结构化数据。
4. 前端并行：
   - 调用 `getHexagramData` 拉取传统文本；
   - 调用 `saveDivinationRecord` 预创建记录（状态为进行中）。
5. 前端调用 `getAIInterpretation`，通过轮询或长轮询获取流式 AI 结果。
6. 结果展示完成后，更新记录状态，供历史页面读取。

## 8. 数据模型

### 8.1 云数据库集合

- `divination_records`
  - `userId`：匿名用户标识（可用 openid）
  - `question`：用户问题
  - `primaryHexagram`：本卦编号
  - `changingHexagram`：变卦编号（无则为空）
  - `changingLines`：数组，标记动爻
  - `aiSummary`：AI 解读摘要
  - `status`：`pending` / `completed` / `error`
  - `createdAt` / `updatedAt`

- `hexagram_texts`
  - `hexagramNo`
  - `name`
  - `judgement`
  - `image`
  - `lines`：六爻文本数组
  - `references`

### 8.2 配置项

使用云函数环境变量或 `config/index.ts` 导出：
- `OPENAI_API_KEY`
- `OPENAI_BASE_URL`（可选）
- `OPENAI_MODEL`
- `RATE_LIMIT_TOKENS`（流控设置）

## 9. 主题与样式

- 使用 `rpx` 单位适配不同屏幕，结合 `safe-area-inset` 处理异形屏。
- 在 `styles/variables.wxss` 定义颜色、字号、间距变量，通过 `page` 的 `data-theme` 切换主题。
- 动画使用 `wx.createAnimation` 或 CSS 过渡，避免高频 `setData`。
- 为暗色模式预留背景、文字、描边三套色值。

## 10. 开发与调试

- **本地运行**：在开发者工具中打开项目，使用模拟器或真机调试。
- **云函数调试**：可直接在工具内调用或通过 `wx.cloud.callFunction` 的 `debug` 模式输出日志。
- **日志记录**：统一使用 `console.log`，配合 `wx.getRealtimeLogManager()` 汇总关键日志。
- **模拟数据**：提供 `mock/` 目录存放模拟云函数响应，支持离线联调。

## 11. 测试策略

- **单元测试**：对脱离 UI 的模块（如 `utils/hexagram.ts`）使用小程序官方单测框架或 Vitest（需额外配置）运行。
- **组件测试**：采用小程序测试框架（如 `miniprogram-automator`）模拟交互。
- **集成测试**：
  - 正常卜筮流程，验证本卦/变卦计算正确。
  - AI 调用失败、网络中断、超时重试等异常路径。
  - 不同主题、机型（全面屏、折叠屏）兼容性。
- **手动验证清单**：
  1. 三次连续卜筮动画不卡顿。
  2. 历史记录列表分页与详情展示。
  3. AI 结果加载失败时出现降级提示。
  4. 暗色模式下文本可读。

## 12. 发布流程

1. **质量检查**：完成测试清单，导出调试基础库版本截图与录屏。
2. **代码托管**：同步至 Git 仓库（建议与 Web 版同 repo 的 `miniprogram/` 子目录）。
3. **体验版**：在小程序后台上传体验版，邀请团队成员测试。
4. **审核材料**：
   - 功能描述：强调“传统文化研究”与“AI 辅助解读”，附免责声明。
   - 截图：主要页面及功能流程。
   - 说明文档：列出使用的第三方服务与用途。
5. **正式发布**：审核通过后发布，并设置运营监控（如 OpenAI 调用统计、云函数日志）。

## 13. 与 Web 版的差异与复用

- **可复用**：
  - 六爻计算逻辑（需改写为适配小程序语法）
  - 卦象数据 JSON / Markdown
  - AI 提示词结构与错误处理思路
- **需重写**：
  - React/Tailwind UI → 小程序 WXML/WXSS
  - Vercel AI SDK → 纯 HTTP 请求
  - 流式响应 → 云函数分段返回或轮询
  - next-themes → 自定义主题切换
- **新增**：
  - 云数据库读写逻辑
  - 微信登录体系（如需绑定用户数据）
  - 审核合规文案与本地隐私声明

## 14. 后续规划

- **性能优化**：引入分包策略、按需加载组件、预加载云函数。
- **功能拓展**：分享海报生成、收藏夹、占筮日历、AI 自定义模型选择。
- **数据洞察**：结合云开发日志分析用户使用路径，优化引导体验。

---

如需补充章节（例如 API 详细契约、设计稿链接），请在文档顶部更新版本号并记录修订人，保持文档与实现同步。
