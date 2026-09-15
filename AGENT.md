# AGENT.md — javatutor-product-introduction-page

> 本文件是本仓（JavaTutor 产品发布页）的规约与文档总索引。新增 **plan** 时必须在本文件登记。

## 项目定位

JavaTutor 的**产品发布页**：单页静态展示，面向评审与访客介绍「执行可视化 + AI 教学智能体」两大板块，
通过 11 个 section + 17 段录屏 demo 讲清产品是什么、解决了什么问题、做到什么程度。
对外发布在 **`intro.javatutor.cn`**，页面内的「官网 ↗」外链到主站应用 `javatutor.cn`。

## 仓库关系

| 仓库 | 角色 |
|---|---|
| `javatutor` | 主站（Vue SPA + Spring Boot），部署在 `javatutor.cn` / `www.javatutor.cn` |
| `javatutor-coze` | Coze 侧智能体（LangGraph），主站通过 `/api/ai/*` 调用 |
| **本仓** | 产品发布页 —— 纯静态，无后端、无构建、无框架依赖 |

三个仓各自独立部署、互不阻塞；本仓的部署**不重启后端、不 reload nginx**。

## 技术形态（当前：零构建）

- **单个 `index.html`**，内联全部 CSS 与 JS。没有打包器、没有 `package.json`、没有依赖。
- **注意：并没有「不准引入构建链」的成文约定。** 零构建是现状，不是团队决议。在没有明确收益前
  保持现状即可 —— 发布页的交付目标是「把一个静态页面送达浏览器」，构建链只会新增
  「构建产物过期」「依赖锁文件漂移」这类与目标无关的故障面。真要引入，先在本文档登记理由。
- 主题靠 CSS 变量（`--sans` / `--mono` / `--ink` / `--bg` / `--line` …）；字体为 DM Sans（400/500/700）
  与 Fragment Mono（400），本地自托管 woff2。

## 本地预览

```bash
# 必须经 HTTP server，不要直接双击 index.html
python -m http.server 8080
# 或
npx serve .
```

> **不要用 `file://` 预览。** 页面依赖 `IntersectionObserver`（滚动显现、视频进出视口自动播放/暂停）
> 与 `<video>` 的自动播放策略，两者在 `file://` 下行为与线上不一致，容易得出错误结论。

## 关键文件地图

| 文件 | 内容 |
|---|---|
| `index.html` | 整个站点。结构 = hero + 11 个 `<section class="sec">`：`pain` / `arch` / `kb` / `tpl` / `viz` / `flow` / `test` / `multifile` / `agent` / `team` / `epilogue` |
| `index.html`（脚本段） | 末尾内联 JS：滚动显现 `io`、视频进出视口 `vio`、logo 注入 `LOGO_CONFIG` |
| `dm-sans-*.woff2` / `fragment-mono-400.woff2` | 本地自托管字体（`@font-face`，第 30–33 行） |
| `logo-light.png` / `logo-dark.png` | Logo（由 `LOGO_CONFIG` 注入，见脚本段） |
| `agent-state-machine.png` | 智能体架构示意图 |
| `videos/` | 17 段录屏 demo，**已入库**。原始素材约 183MB、码率 2.2–5.5 Mbps —— **入库前必须转码**，见下方红线 1 |
| `docs/plan/` | 方案与实施计划 |

## 部署

线上目录 **`/var/www/javatutor-intro/`**，由本仓的 `.github/workflows/deploy.yml` 在 push **`master`** 时
rsync 过去（仓库默认分支是 `master`）。首次搭建与服务器侧配置见
[`docs/plan/2026-09-15-product-intro-page-integration-plan.md`](docs/plan/2026-09-15-product-intro-page-integration-plan.md)。

> 该 workflow 依赖本仓自己的 GitHub Secrets（`SSH_PRIVATE_KEY` / `SSH_HOST` / `SSH_USER` / `SSH_PORT`）。
> **GitHub Secrets 是 per-repo 的，主站的配置不会自动共享。**

## 硬约束（红线）

1. **视频必须转码后入库，单段码率 ≤ 1.5 Mbps。**
   原始 1080p 录屏码率 **2.2–5.5 Mbps**，高于 ECS 链路带宽（3 Mbps）—— 边下边播会反复缓冲，
   不是「慢」是**放不了**；183MB 的总体积还会让流量费翻约 4 倍（0.8 元/GB）。
   转码参数与验收判据见计划 Task 4。

2. **部署目录绝不能落在 `/opt/javatutor/` 之下。**
   主站的 `deploy.yml` 用 `rsync --delete` 镜像到 `/opt/javatutor/`，放在那里的任何文件都会被
   **静默删除**，而且主站部署仍显示绿色成功。站点级静态资源一律放 `/var/www/`。

3. **ECS 公网计费必须保持「按使用流量」模式。**
   改回固定带宽 3 Mbps 会让视频重新放不动。同时**必须保有消费预警**（建议 50 元）——
   按流量计费没有硬性费用上限，预警是唯一的兜底。

4. **资源引用一律保持相对路径。**
   这是发布页能整体搬迁到任意域名/路径的前提。

5. **不得硬编码密钥或 token。** 本仓是静态页，不该出现任何凭据。

## 已知设计取舍（**不要"修"**）

| 现状 | 为什么 | 误改的后果 |
|---|---|---|
| 视频**不用** `autoplay` 属性，改由 `IntersectionObserver` 在进入视口时 `play()` | 13 个 autoplay loop 视频同时解码会占满 GPU/解码资源，滚回顶部时合成层错乱（视频残影盖住 hero logo） | 页面滚动卡顿、hero 被残影覆盖 |
| 滚出视口即 `pause()`；hero 视频重入视口时 `v.load()` 重建纹理 | 清掉离屏期间可能错位的残留纹理 | hero 出现残影 |
| 团队区含成员**真实姓名** | 2026-09-15 团队确认**有意保留** | 不要"脱敏"，那会改掉有意为之的署名 |

## 文档规范（强制）

| 触发点 | 必须动作 |
|---|---|
| 新增 plan | 放 `docs/plan/YYYY-MM-DD-<topic>-plan.md`，并在本文档登记 |
| 提交前 | 本地起 HTTP server 过一遍 11 个 section + 17 个视频；确认无硬编码密钥 |
| 文档变更 | 更新本文档索引；文件名只允许字母/数字/下划线/短横线；统一 UTF-8；**不得出现 `TBD` / `TODO`** |
| git 操作 | 除非用户明确指示，**不主动 commit / push** |

> devlog / review 目录暂不设立；等开发者提出要求再建。

## 当前状态

- 分支：`master`
- 已完成：发布页全部内容（11 个 section + 17 段 demo + 自托管字体 + logo）
- 进行中：**集成上线**（`intro.javatutor.cn` 子域名 + `/var/www/javatutor-intro/` 静态托管
  + 带宽计费切换 + 视频转码），见下方计划
- 未开始：`.github/workflows/deploy.yml` 尚未建立（计划 Task 6）

## 文档索引

| 类别 | 文件 | 说明 |
|---|---|---|
| 计划 | [`docs/plan/2026-09-15-product-intro-page-integration-plan.md`](docs/plan/2026-09-15-product-intro-page-integration-plan.md) | 发布页集成上线计划：`intro.javatutor.cn` 子域名 + 静态托管 + ECS 按流量计费（峰值 100 Mbps）+ 视频转码留 ECS + 部署 workflow。**T0 现场取证已完成（2026-09-15）**，含 C1–C12 前提、R1–R7 风险与 S1–S11 验收 |
