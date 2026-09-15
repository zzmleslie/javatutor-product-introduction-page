# 落地计划：产品发布页集成上线（`intro.javatutor.cn` 子域名 + 静态托管 + 带宽方案）

> **本仓（`javatutor-product-introduction-page`）即发布页专属仓，本计划随发布页一起版本化。**
> 涉及主站的部分只做说明，不在本仓执行。
>
> 发布页现状：单文件 `index.html`（65KB、11 个 section）+ 17 段 demo 录屏（**183MB**）
> + 4 个自托管字体 + logo。全部资源引用均为**相对路径**。
>
> 主站：`javatutor.cn` / `www.javatutor.cn` 解析到 `112.124.67.74`（阿里云 ECS 杭州，Vue SPA + Spring Boot），
> 已启用 HTTPS（Let's Encrypt）。
>
> **采用的方案**：发布页走独立子域名 `intro.javatutor.cn`，静态托管在 ECS 上**独立于主站目录**的
> `/var/www/javatutor-intro/`；视频**转码后留在 ECS**；ECS 公网计费从「固定带宽 3 Mbps」
> 改为「按使用流量 + 峰值 100 Mbps」。**主站代码零改动。**
> 备选方案（路径分区）见 §7，后续可选的 CDN 迁移见 §6。
>
> **状态：T0 现场取证已于 2026-09-15 完成**（见 §4），仅剩 T0-5（计费切换是否闪断）需控制台确认，
> 不构成阻断。计划中所有涉及线上环境的描述均已按实测结果校正。

---

## 0. 已确认的前提

| # | 事项 | 结论 |
|---|---|---|
| C1 | 发布页团队区含成员真实姓名 | **有意保留**，不要"脱敏" |
| C2 | 发布页资源引用 | 全部相对路径（已核实）→ 可整体搬迁 |
| C3 | 发布页第 420 行已有「官网 ↗ javatutor.cn」 | 与子域名方案语义一致，无需改文案 |
| C4 | ECS 公网带宽 | 固定带宽 **3 Mbps** |
| C5 | 计费模式 | 可改为**按使用流量计费，峰值可调 100 Mbps，0.8 元/GB**（本计划采用，见 §1） |
| C6 | `cdn.javatutor.cn` 备案 | 主域名已备案，子域名继承、**无需额外备案** |
| C7 | 线上证书 | **实测**：Let's Encrypt，`/etc/letsencrypt/live/javatutor.cn/`（**全机只有这一份证书**），SAN 含 `javatutor.cn` / `www` / `miaomiaomiaomiao` / `meowmeowmeowmeow`，**不含 `intro`**；2026-10-21 到期。`certbot.timer` 活跃（每 12h 自检）。需 `--expand` 扩容，**必须列全 5 个域名**（Task 3.2） |
| C8 | 执行方式 | **实测可用**：ECS 侧经 `workbench exec` 执行（**需 v1.0.1+**，见下方环境说明），实例 `i-bp18peop1g8ir00s8phq`（`javatutor`，cn-hangzhou，Running，Ubuntu 22.04，**passwordless sudo 可用**）。控制台侧（DNS / 带宽计费 / 费用预警）需人工或 `aliyun` CLI |
| C9 | 17 段视频 | **全部保留**，不做删减 |
| C10 | **nginx 真实布局** | 全机 **只有 1 个站点文件** `/etc/nginx/sites-enabled/javatutor`，`conf.d/` 为空。**`/var/www/html` 是软链 → `/opt/javatutor/dist`**（inode 已验证相同）。端口 80/443 各只有一个 server 块，**均无 `default_server`** |
| C11 | 后端 | `java` 监听 `127.0.0.1:8080`，与 `location /api/` 的 `proxy_pass` 一致。SSE 已配 `proxy_buffering off` / `proxy_read_timeout 300s` |
| C12 | 环境 | 系统盘 40G，**可用 26G**；服务用户 `javatutor`；`/var/www/` 下另有历史遗留 `html.bak/`（www-data，与本计划无关） |

> **执行环境（2026-09-15）**：`workbench` 已升级至 **v1.0.1**，在 `PATH` 上直接可用，`exec` 验证通过。
> **版本下限是 v1.0.1** —— v1.0.0 的 `exec`/`upload`/`download` 在 Windows 上会报
> `dial tcp: address \\.\pipe\...: missing port in address`（命名管道 bug），而 `list ecs`/`config` 正常
> （走 API 不走本地 daemon），容易误判成"工具没问题"。若发现 `exec` 报此错，先查版本。

---

## 1. 带宽方案：为什么是「转码 + 改流量计费」，而不是 OSS / CDN

### 1.1 实测数据（ffprobe，非估算）

| 文件 | 分辨率 | **码率** | 时长 | 体积 |
|---|---|---|---|---|
| `kb-demo` | 1920×1080 | **5.49 Mbps** | 56.4s | 38.73 MB |
| `tpl-demo` | 1920×1080 | **5.01 Mbps** | 58.8s | 36.85 MB |
| `landing-mascot`（hero） | 1920×1080 | **4.57 Mbps** | 21.2s | 12.16 MB |
| `agent-scene3-step` | 1814×1080 | 3.15 Mbps | 44.0s | 17.34 MB |
| `multifile-uml-nav` | 1816×1080 | 3.07 Mbps | 44.5s | 17.13 MB |
| `test-demo` | 1728×1080 | 2.24 Mbps | 66.2s | 18.60 MB |

**两条硬事实：**

1. **视频流码率（4.57–5.49 Mbps）高于链路带宽（3 Mbps）** ⇒ 边下边播跑不动，会反复缓冲。
   不是「要等 32 秒」，是**放不了**。
2. 这些是**录屏**（静态 UI + 文字为主、运动极少），5 Mbps 对 1080p 录屏是**严重过编码**。
   x264 CRF 26 + 缩宽 1280，录屏通常可达 **0.8–1.2 Mbps 且文字依然清晰** ⇒ **约 5 倍压缩**。

### 1.2 方案对比

| 方案 | 单价 | 工期 | 备案 | 结论 |
|---|---|---|---|---|
| 现状（固定带宽 3 Mbps） | 已付费 | — | — | ❌ 视频放不动 |
| **转码 + 改按流量计费** | **0.8 元/GB** | **半天** | 不需要 | ✅ **采用** |
| OSS 直出（绑自定义域名） | ~0.5 元/GB | 天级 | **需要** | ❌ 省不了事还更贵 |
| OSS + CDN | ~0.2 元/GB | 天级 | 需要 | 后续可选（§6） |

> 说明：OSS 是**存储**、CDN 是**分发**，不是二选一。OSS 直出看似最省事，但绑自定义域名 + HTTPS
> 同样要备案，且每次回源、无边缘缓存，单价还高于 CDN。所以实际只有「留 ECS」和「OSS + CDN」两条路。
> CDN 单价便宜 4 倍，但省下的是几十元，付出的是数天工期 + 备案风险 —— 大赛场景下不划算。

### 1.3 成本测算（0.8 元/GB，转码后按 50MB / 次完整浏览）

| 场景 | 流量 | 费用 |
|---|---|---|
| 1 次完整浏览（17 段全看） | 0.05 GB | **0.04 元** |
| 200 次浏览 | 10 GB | **8 元** |
| 1000 次浏览 | 50 GB | **40 元** |

### 1.4 采用这个方案必须配的两件事

1. **费用预警（强制）**：按流量计费**没有硬性费用上限**。必须在阿里云费用中心设消费预警
   （建议 50 元）。这是本方案唯一的真风险。
2. **视频转码（强制）**：把 183MB 降到 ~40MB，省 4 倍流量；**并且在万一尚未切换计费模式时，
   页面就已经可用**（1 Mbps 的流转 3 Mbps 的链路，流畅不缓冲）。

---

## 2. 目标形态

**现状（已取证）**：

```
javatutor.cn 等 4 个域名 → 112.124.67.74 → nginx
    ├─ /var/www/html  ──(软链)──→  /opt/javatutor/dist/   ← Vue SPA 真实文件在这里
    └─ /api/          ──→  127.0.0.1:8080                  ← Spring Boot

/etc/nginx/sites-enabled/ 只有 1 个文件：javatutor
```

**目标**：

```
intro.javatutor.cn       → 112.124.67.74 → nginx → /var/www/javatutor-intro/  （新增，真实目录）
                                                      └─ videos/ 转码后的 mp4

ECS 公网计费：固定带宽 3 Mbps  →  按使用流量 + 峰值 100 Mbps
```

**关键隔离原则**：发布页目录必须是 `/var/www/` 下的**真实目录**，**绝不能放进 `/var/www/html/` 之下** ——
那会穿过软链落进 `/opt/javatutor/dist/`，也就是 `--delete` 的靶心。原因见 §3 R1。

---

## 3. 风险清单（动手前必读）

| 编号 | 风险 | 后果 | 对策 |
|---|---|---|---|
| **R1** | 把发布页放进 `/opt/javatutor/`，**或放进 `/var/www/html/` 之下**（会穿过软链落进 `/opt/javatutor/dist/`） | 主站下次部署时，`deploy.yml` 的 `ARGS: "-avz --delete"` **静默删除发布页**，且部署仍显示绿色成功 | 发布页放 `/var/www/javatutor-intro/`（**真实目录**，软链之外的兄弟节点）。主站仓的 `deploy/README.md` 已加入该红线（2026-09-15） |
| **R2** | 按流量计费**无费用硬上限** | 被恶意刷流量 / 视频被大量传播 → 账单失控 | **强制**设费用预警（Task 1）；`/videos/` 加 Referer 防盗链（Task 3） |
| **R3** | **已确认并已修**：主站仓库 `deploy/nginx/javatutor.conf` 写的是 `root /var/www/javatutor`，而线上是 `root /var/www/html`（软链 → `/opt/javatutor/dist`）；`deploy/scripts/deploy.sh` 的 `REMOTE_WEB_ROOT` 也是 `/var/www/javatutor` —— **7-19 迁移后一直没同步，前端推送静默失效** | 照仓库任一份抄都会配错 | 2026-09-15 已修正三处并加入说明；本计划**不修改任何主站配置**（证书/nginx 均不动） |
| **R4** | GitHub Secrets 是 **per-repo** 的 | 在本仓新建 workflow 时 `SSH_*` 取不到值，部署静默失败 | 在本仓 Settings → Secrets 重新配置这 4 个 |
| **R5** | 新 server 块的 `server_name` 未精确匹配 | **:80 落到 certbot 那个默认块（`return 404`）→ 404；:443 落到主站块 → 返回 Vue 应用**。两种都不是发布页 | Task 3 用 `curl -H "Host: intro.javatutor.cn"` 直连 127.0.0.1 验证命中 |
| **R6** | 发布页仍在拉 `fonts.googleapis.com`（`index.html` 第 7–9 行） | 墙内首屏闪字体 / 空白 | Task 5 自托管 |
| **R7** | 新 server 块进 `sites-enabled/` 会改变**默认 server 的选举** —— 线上**没有任何块写了 `default_server`**，当前 80/443 各只有一个块故无歧义，加完就变成各两个 | **反向影响主站**（未知 Host 的请求被 intro 接走） | `sites-enabled/` 的加载顺序是字母序，`javatutor` 排 `javatutor-intro` 之前 → 主站仍为默认。**文件名必须用 `javatutor-intro`**；加完立即用 `curl -H "Host: 1.2.3.4" http://127.0.0.1/` 复验仍走主站 |

> R7 的备选加固（**仅在复验失败时**用）：给主站 :80 块补 `default_server`。这是一处对主站配置的改动，
> 与本计划「不动主站」的原则冲突，所以不默认执行。

---

## 4. T0：现场取证 —— **已完成（2026-09-15）**

> ECS 侧用 `workbench exec` 执行（实例 `i-bp18peop1g8ir00s8phq`，需 v1.0.1+，见 C8）。

| # | 待确认 | 结论 | 状态 |
|---|---|---|---|
| T0-0 | `workbench exec` 可用性 | v1.0.1 修复，用户目录安装后 `nginx -v` 正常返回 | ✅ |
| T0-1 | 默认 server / 证书配置 | 全机**无 `default_server`**；:80 与 :443 各只有 1 个 server 块（证书配置为标准 certbot 块，`include options-ssl-nginx.conf`） | ✅ |
| T0-2 | 主站真实 root | **`/var/www/html` →（软链）→ `/opt/javatutor/dist`**，**不是**仓库里写的 `/var/www/javatutor`（R3 确认） | ✅ |
| T0-3 | 后端服务用户 | `javatutor` | ✅ |
| T0-4 | 磁盘余量 | 系统盘 40G，**可用 26G** —— 远超需要 | ✅ |
| T0-5 | 计费切换是否受限 / 是否闪断 | — | ❌ **唯一遗留**，需控制台确认 |
| T0-6 | certbot 自动续期 | `certbot.timer` 活跃（12h 周期，最近一次 4h56m 前）；`/etc/letsencrypt/live/` 下只有 `javatutor.cn` 一份证书 | ✅ |

**T0 推翻/修正了本计划原先的三处臆测**（§2、§3、§5 已同步更正）：

1. **主站 webroot 三处写法互不相同**：仓库 `deploy/nginx/javatutor.conf` 写 `/var/www/javatutor`、
   `deploy/scripts/deploy.sh` 写 `REMOTE_WEB_ROOT="/var/www/javatutor"`、**线上实际是
   `/var/www/html` →（软链）→ `/opt/javatutor/dist`**。以线上为准（R3）。
2. 线上**不存在** `server_name _` 的 catch-all 块 —— R5 的失败形态因此不同：
   `server_name` 没匹配上时，**:80 落到 certbot 的 `return 404` 块、:443 落到主站块**（不是回到某台"默认"）。
3. 线上**不存在**显式 `default_server`，但当前每端口只有一个块所以无歧义 ——
   R7 是**新增块之后才出现**的风险，且依赖 `sites-enabled/` 的字母序，必须复验。

> **T0-5 未完成不构成阻断** —— 它只影响 Task 1 的执行时机（是否避开高峰），不影响其余任务。

---

## 5. 任务分解

### Task 1：ECS 带宽计费切换 + 费用预警

1. 阿里云控制台 → 费用中心 → **设置消费预警（建议 50 元）**。**先做这一步，再切计费模式。**
2. 控制台 → ECS → 实例 → 网络与安全组 → 带宽 →
   计费方式改为 **按使用流量**，峰值带宽设为 **100 Mbps**。
3. 观察是否闪断（按 T0-5 的结论选择执行时段）。

**验收**：控制台显示「按使用流量 / 峰值 100 Mbps」；`curl -o /dev/null -w '%{speed_download}\n' https://javatutor.cn/`
的下载速度**显著高于 3 Mbps 的量级**（约 375 KB/s）。

**回滚**：改回固定带宽 3 Mbps（可逆）。

---

### Task 2：DNS A 记录

阿里云 DNS（云解析）→ 域名 `javatutor.cn` → 新增记录：

| 记录类型 | 主机记录 | 解析线路 | 记录值 | TTL |
|---|---|---|---|---|
| A | `intro` | 默认 | `112.124.67.74` | 600 |

> **只做 DNS。证书扩容必须等到 Task 3 建好 server 块之后** —— 见 Task 3 的说明。

**验收**：`dig +short intro.javatutor.cn` 返回 `112.124.67.74`。

---

### Task 3：服务器目录 + nginx（含证书扩容）

> **顺序很重要**：先建 server 块，再跑 certbot。
> `certbot --nginx` 需要先有一个 `server_name intro.javatutor.cn` 的块去挂载证书；
> 若不存在，它可能把 `intro` 塞进**主站**的 server 块 —— 那就动了主站配置，正是本计划要避免的。

```bash
# 1) 建目录（注意：不在 /opt/javatutor 下 —— R1）
sudo mkdir -p /var/www/javatutor-intro

# 2) 属主给部署用户（取 T0-3 的结果；与主站 deploy.yml 的做法一致）
svc_user="$(systemctl show -p User --value javatutor)"
sudo chown -R "${svc_user:-root}:${svc_user:-root}" /var/www/javatutor-intro
sudo chmod 755 /var/www/javatutor-intro

# 3) 占位页，先验证 nginx 路由（此时不要放真页）
echo '<h1>javatutor-intro placeholder</h1>' | sudo tee /var/www/javatutor-intro/index.html
```

新增 `/etc/nginx/sites-available/javatutor-intro`（**独立文件，不改动现有 `javatutor` 配置**）：

```nginx
server {
    listen 80;
    server_name intro.javatutor.cn;

    root /var/www/javatutor-intro;
    index index.html;

    gzip on;
    gzip_types text/plain text/css application/json application/javascript
               image/svg+xml font/woff2;

    # 视频与 API 共用带宽：给视频连接限速，保证 /api/ai/chat 的 SSE 不被挤占。
    # 转码后单段码率约 1 Mbps，20m 的单连接上限足够流畅，且可容纳约 5 路并发。
    location /videos/ {
        limit_rate_after 2m;      # 首 2MB 不限速，保证起播快
        limit_rate 20m;           # 之后限到 20 Mbps
        add_header Cache-Control "public, max-age=604800";
        # R2：防盗链（主站 + 发布页自身）
        valid_referers none blocked javatutor.cn *.javatutor.cn;
        if ($invalid_referer) { return 403; }
    }

    # 静态资源缓存（logo / 字体是固定文件名）
    location ~* \.(png|woff2|jpg|svg)$ {
        add_header Cache-Control "public, max-age=604800";
    }

    location / {
        try_files $uri $uri/ /index.html;
    }
}
```

启用并验证：

```bash
sudo ln -s /etc/nginx/sites-available/javatutor-intro /etc/nginx/sites-enabled/
sudo nginx -t                      # 必须先过，再 reload
sudo systemctl reload nginx

# R5：直连 127.0.0.1 带 Host 头，确认命中新 server 块而不是默认 server
curl -s -H "Host: intro.javatutor.cn" http://127.0.0.1/ | head -1
# 期望：<h1>javatutor-intro placeholder</h1>
# 若返回 Vue 的 index.html ⇒ server_name 没匹配上（R5）或默认 server 选举被改（R7）

# 主站必须仍然正常 —— R7 的反向验证
curl -sI -H "Host: javatutor.cn" http://127.0.0.1/ | head -1
```

**验收**：占位内容命中；主站 `Host: javatutor.cn` 仍返回 Vue 应用；`curl -sI https://intro.javatutor.cn` 返回 `200`。

**3.2 证书扩容**

> **必须列全 5 个域名。** `certbot --expand` 是以 `-d` 的**完整集合**为准的 —— 只写其中几个
> 会生成一份**丢掉其余域名**的新证书，把 `miaomiaomiaomiao` / `meowmeowmeowmeow`
> 两个 demo 子域的 HTTPS 直接搞坏。`--cert-name` 用于钉死同一份 lineage，避免 certbot 另起一份。

```bash
# 先看清楚现有证书叫什么、含哪些域名
sudo certbot certificates

# 扩容：列全 5 个域名
sudo certbot --nginx --cert-name javatutor.cn --expand \
     -d javatutor.cn \
     -d www.javatutor.cn \
     -d miaomiaomiaomiao.javatutor.cn \
     -d meowmeowmeowmeow.javatutor.cn \
     -d intro.javatutor.cn \
     --non-interactive --agree-tos
```

certbot 会自动往新建的 `javatutor-intro` server 块里插入 `listen 443 ssl` 与证书指令
（与主站块同款 `# managed by Certbot` 风格）。`certbot.timer` 已在运行（T0-6），续期无需另行配置。

**验收（三件事都要过）**：

```bash
# ① 新域名在证书里
echo | openssl s_client -connect intro.javatutor.cn:443 -servername intro.javatutor.cn 2>/dev/null \
  | openssl x509 -noout -ext subjectAltName

# ② 原有 4 个域名一个都没丢（R5 之外的第二个回归点）
for h in javatutor.cn www.javatutor.cn miaomiaomiaomiao.javatutor.cn meowmeowmeowmeow.javatutor.cn; do
  printf '%-38s ' "$h"
  echo | openssl s_client -connect "$h:443" -servername "$h" 2>/dev/null \
    | openssl x509 -noout -subject 2>/dev/null | head -1
done

# ③ 主站仍正常
curl -sI https://javatutor.cn | head -1
```

期望：① 输出含 `DNS:intro.javatutor.cn`；② 4 行都返回成功；③ `200`。

---

### Task 4：视频转码 + hero poster（**强制**）

> 依据 §1.4。**这一步先于真页公开** —— 未转码的 5 Mbps 视频在 3 Mbps 链路上放不了。

**4.1 转码**

```bash
mkdir -p videos-opt
for f in videos/*.mp4; do
  ffmpeg -y -i "$f" -c:v libx264 -crf 26 -preset slow \
         -vf "scale='min(1280,iw)':-2" -an -movflags +faststart \
         "videos-opt/$(basename "$f")"
done

# 对照：体积与码率
du -sh videos videos-opt
for f in videos-opt/*.mp4; do
  ffprobe -v error -select_streams v:0 -show_entries stream=bit_rate \
          -of csv=p=0 "$f" | sed "s|^|$(basename "$f") |"
done
```

**验收（客观判据）**：

- 总体积 **≤ 60MB**（目标 ~40MB）；
- 单段码率 **≤ 1.5 Mbps**；
- **画质判据（由项目负责人判定）**：随机抽 3 段，在 100% 缩放下看视频里代码区的**最小字号**，
  **能一次读清**即通过；糊了就把 CRF 降到 22–24 重跑该段。

**4.2 hero poster**

```bash
ffmpeg -y -i videos-opt/landing-mascot.mp4 -ss 2 -frames:v 1 \
       -vf scale=1280:-2 -q:v 3 poster-hero.jpg
```

在 `index.html` 的 hero `<video>` 上加 `poster="poster-hero.jpg"`。

**4.3 替换**

```bash
# 用转码后的文件替换原文件（原文件先备份到 videos-orig/，作为回滚手段）
mv videos videos-orig && mv videos-opt videos
```

**4.4 更新 `.gitignore` 与仓内说明**

`videos/` 目前**已在 git 中**（183MB）。转码后同名替换，因此：

- 转码后的 `videos/`（~40MB）**照常入库** —— 部署 workflow 需要从仓里 rsync 出去；
- `videos-orig/`（183MB）加入 `.gitignore`（若无 `.gitignore` 则新建）；
- **已知后果**：183MB 原始素材会**永久留在 git 历史里**，克隆会变慢。清理历史需要重写提交
  （`git filter-repo`），**本计划不做** —— 收益小、风险高（会改写所有 commit hash）。

---

### Task 5：字体自托管（修 R6）

`index.html` 第 7–9 行从 `fonts.googleapis.com` 拉 `Source Serif 4`（仅 italic 400/600）：

```bash
# 1) 下载 woff2 到本地（与已有 dm-sans / fragment-mono 并列）
# 2) 删除 index.html 第 7–9 行（两个 preconnect + 一个 stylesheet 链接）
# 3) 在已有 @font-face 段落补 Source Serif 4 italic 声明，指向本地 woff2
```

**验收**：`grep -c 'fonts.googleapis.com' index.html` 为 `0`；屏蔽该域名时衬线字体仍正常渲染。

---

### Task 6：本仓的部署 workflow

在本仓新建 `.github/workflows/deploy.yml`：

```yaml
name: Deploy Intro

on:
  push:
    branches: [master]        # 本仓默认分支是 master，不是 main

permissions:
  contents: read

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: 部署发布页到服务器
        uses: easingthemes/ssh-deploy@v5.1.1
        with:
          SSH_PRIVATE_KEY: ${{ secrets.SSH_PRIVATE_KEY }}
          REMOTE_HOST: ${{ secrets.SSH_HOST }}
          REMOTE_USER: ${{ secrets.SSH_USER }}
          REMOTE_PORT: ${{ secrets.SSH_PORT }}

          # --delete 在这里是**想要的**：发布页目录专属于它自己，可安全镜像。
          # 转码后的 videos/ 在仓内，随 rsync 一起同步；--delete 会正确清理删掉的旧文件。
          ARGS: "-avz --delete --exclude=.git --exclude=.github --exclude=videos-orig"

          SOURCE: "./"
          TARGET: "/var/www/javatutor-intro/"

          SCRIPT_AFTER: |
            sudo chmod -R a+rX /var/www/javatutor-intro
```

**前置**：在本仓 Settings → Secrets and variables → Actions 配好 `SSH_PRIVATE_KEY`
`SSH_HOST` `SSH_USER` `SSH_PORT`（**R4：per-repo，主站的配置不会自动共享**）。

> 转码后 `videos/` 约 40MB，首次 rsync 传一次，之后是增量（rsync 按块比对，未改动的视频不再传输）。
> 本 workflow **不重启后端、不 reload nginx** —— 静态文件就位即生效，与主站部署完全解耦。

**验收**：push 一次，Actions 变绿；`ls /var/www/javatutor-intro/index.html` 存在；
`curl -sI https://intro.javatutor.cn` 返回 `200` 且 `Content-Type: text/html`。

---

### Task 7：冒烟与回归

| # | 检查 | 命令 / 动作 | 期望 |
|---|---|---|---|
| S1 | 发布页可访问 | `curl -sI https://intro.javatutor.cn` | `200` |
| S2 | 主站**未受影响** | `curl -sI https://javatutor.cn`；跑一段代码；发一条 agent 提问 | `200`；代码可跑；SSE 正常流式 |
| S3 | 域名隔离 | `curl -s https://intro.javatutor.cn \| head -3` | 是发布页，不是 Vue 应用 |
| S4 | 子路径回退 | `curl -sI https://intro.javatutor.cn/nonexistent` | `200` |
| S5 | **视频码率** | 对 `/var/www/javatutor-intro/videos/` 逐个 `ffprobe` | 每段 **≤ 1.5 Mbps** |
| S6 | **限速下可播** | DevTools Network → 自定义限速 **3 Mbps**，硬刷新，滚完全页 | 视频**无反复缓冲**；hero 因 poster **立刻有画面** |
| S7 | 防盗链 | `curl -sI -e "https://evil.example/" https://intro.javatutor.cn/videos/kb-demo.mp4` | `403` |
| S8 | 带宽计费已生效 | 大文件下载测速 | 显著高于 375 KB/s |
| S9 | **R1 隔离回归**（最关键） | 在 ECS 上跑：<br/>`readlink -f /var/www/javatutor-intro`（期望原样返回，**不是** `/opt/javatutor/...`）<br/>`readlink -f /var/www/html`（期望 `/opt/javatutor/dist`，作为对照）<br/>`echo /var/www/javatutor-intro \| grep -q '^/opt/javatutor/' && echo 危险 \|\| echo 安全` | 发布页目录**不在** `--delete` 的 DEST 树内；`安全` |
| S10 | 发布页链接 | 点「官网 ↗」 | 跳转 `https://javatutor.cn` 可达 |
| S11 | **主站部署后复验**（真回归） | 主站下次 `deploy.yml` 跑完后，重跑 S1/S3 | 发布页**仍在**、内容未变 |

> **S6 是本次的核心验收**：它直接回答「3 Mbps 下页面能不能用」。
> **S9 用软链/路径判据代替在生产 push** —— 同样的结论（发布页不在删除范围内），代价为零、不碰生产。
> **S11 才是真正的端到端证明**，但它的成立依赖主站下一次部署自然发生，不为此专门触发。

---

## 6. 后续可选：什么时候才值得迁 CDN

当**月度流量费持续超过迁移成本**时再做，判据：

- 月流量 > 100GB（≈80 元/月）并持续 2 个月 ⇒ CDN 单价 0.2 元/GB，此时节省已覆盖工期成本；
- 或出现跨国 / 跨运营商访问体验问题。

迁移时的改动面（本计划**不含**）：开 OSS bucket → 上传 `videos/` → 绑 `cdn.javatutor.cn`
（需确认备案，C6）→ 改 `index.html` 的 17 处 `src` → 本仓 workflow 加 `--exclude=videos`。

---

## 7. 备选分支：路径分区（若「发布页必须占根域名」）

仅当交付/评审硬性要求根域名为发布页时启用。**注意：以下改动点须按 §4 取证到的真实布局重新推导** ——
线上 webroot 是软链 `/var/www/html → /opt/javatutor/dist`，而非仓库里写的 `/var/www/javatutor`。
改动点（**均在主站仓**）：

1. `frontend/vite.config.js`：`base: '/app/'`
2. `frontend/src/router/index.js`：`createWebHistory('/app/')`
3. nginx 主站块：`location /app/ { alias /opt/javatutor/dist/; try_files $uri $uri/ /app/index.html; }`，
   发布页占 `location = / { root /var/www/javatutor-intro; ... }`
4. 主站 `deploy.yml` 的 `frontend/dist` 与 nginx 的 `location /app/` 路径需对齐
   （**现状是软链 `/var/www/html`，改之前必须先决定是否拆掉这个软链**）
5. **新增回归**：应用内所有绝对路径跳转、静态资源 URL、`/api/` 相对路径调用需全量复验

> 风险等级显著高于方案 A，返工成本集中在「主站白屏」这一最坏故障上。启用前需单独评审。

---

## 8. 回滚

| 层 | 回滚动作 | 影响面 |
|---|---|---|
| 带宽计费 | 控制台改回固定带宽 3 Mbps | 全站；视频将重新放不动 |
| nginx | `sudo rm /etc/nginx/sites-enabled/javatutor-intro && sudo systemctl reload nginx` | 仅发布页下线；主站不受影响 |
| 证书 | 若扩容后经 Task 3.2 验收 ② 发现有域名丢失：用**列全 5 个域名**的同一命令重跑 certbot（幂等，可反复执行） | 全站 HTTPS |
| DNS | 删除 `intro` A 记录 | 仅发布页 |
| 视频 | `mv videos videos-new && mv videos-orig videos`（原始素材已备份） | 仅发布页 |

**主站（`deploy.yml` / `dist` / 后端）全程未被修改，因此不存在主站回滚需求。**

---

## 9. 交付顺序

```
T0 取证 ✅ 已完成（2026-09-15，仅剩 T0-5 控制台确认）
  └─ Task 1 带宽计费切换 + 费用预警（先设预警，再切换）
       └─ Task 2 DNS A 记录
            └─ Task 3 目录 + nginx 占位页 → 验证路由与主站未受影响 → 证书扩容（含主站回归）
                 └─ Task 4 视频转码 + poster      ← 强制，先于真页公开
                      └─ Task 5 字体自托管
                           └─ Task 6 部署 workflow（真页上线）
                                └─ Task 7 冒烟（S5/S6 视频与限速、S9 隔离回归）
```

> **Task 3 内部有严格顺序**：建 server 块 → 验证命中 → 才能跑 certbot。
> 反过来做，certbot 可能把 `intro` 塞进主站的 server 块。

**M1 结束判据**：`intro.javatutor.cn` 可达且 HTTPS 正常、11 个 section 完整可浏览、
**S5/S6 通过（3 Mbps 限速下视频不缓冲）**、主站无回归（S2）、
**证书仍含全部 4 个原有域名**（Task 3.2 验收 ②）、隔离成立（S9）。
