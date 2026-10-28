# Leo Agent 系统（可移植通用版白皮书）

> 一套让拥有沙箱的 AI 助手"跨会话不失忆，新环境快速启动"的文件协作方案。
> 由 [LeoLee](https://s.iee.li/l/u1) 首创，已在多类 AI 沙箱环境实战验证，本文档提炼其完整设计，可迁移到任何带沙箱系统的 AI agent 平台。
> 本版去除所有平台绑定：内核完全平台无关；平台相关差异全部收敛到"适配层"。

**v2.1 变更要点**：核心文件名全英文化（防编码 BUG）；新增轻量记忆索引 `memory-index.md`；媒体目录 `img/` → `pan/`；宪法 `README.md` 不可私自改动并派生永久 `README1ST.md`；流水账从总记忆拆为按周期归档的 `log-*.md`；新增可选项加密保险库（参考 leoi.org/pass）；扩充可选增强清单。

**v2.2 变更要点**：为第十节每个可选增强补充"推荐方案 + 详细教程 + 风险与注意"；内网穿透给出 Cloudflare Tunnel / frp / ngrok 实操步骤与常见坑（待 Leo 实测配置替换）。
**v2.3 变更要点**：将 10.1 内网穿透替换为 Leo 实测的 cloudflared 方案（剔除平台绑定名），保留两种用法 + 6 条实战经验 + frp 备选，删除原通用 ngrok/bore 推测版。

---

## 一、这套方案解决什么问题

AI 对话有两个天生缺陷：**会话结束记忆清零**，以及**多会话之间互相不知道对方干过什么**。结果是每次开新对话都要重新交代背景、重新踩一遍老坑、重复传一遍老文件。

Leo 方案的思路很简单：**把 AI 的"外置记忆"放进文件系统**。约定一个固定工作区目录，里面放规则、经验、日志和文件；再约定一套工作流程，强制 AI 在"会话开始读、任务结束写"。文件不会失忆，于是 AI 也就"不失忆"了。同时这套固定工作区可以定时打包，开启新对话的时候直接发送即可，实现"新环境快速启动"。

此外，长期使用会带来两个工程问题，本版专门解决：
- **记忆无限膨胀**：总记忆可能极其庞大，必须用"轻量索引 + 按需调阅归档"避免每次全量读取（token 炸弹）。
- **规则被误改/误删**：核心规则（宪法）必须不可被 AI 私自改动，且第一版规则作为底层根逻辑永久留存。

## 二、核心设计思想（四条）

1. **文件即记忆**：AI 的记忆不靠对话历史，靠工作区里的文档。对话会丢，文件不会。
2. **目录即分类**：不同类型的记忆放不同文件——稳定规则放 README，增长型知识放 memory.md，目录状态放 file-tree，流水账放独立归档。
3. **日志即传承**：每次操作都追加日志，把踩过的坑、验证过的结论写成"经验条目"。下个会话的 AI 先查经验再动手，同一个坑不踩第二次。
4. **索引即导航**：面对可能极其庞大的记忆，用一张轻量索引（`memory-index.md`）做快速定位，AI 先扫索引再定点读具体文件，避免全量读取。

**宪法保护（跨切面铁律）**：README 作为"宪法"不可被 AI 私自改动；第一版 README 落盘时同步派生 `README1ST.md` 作为底层根逻辑与安全规范，**无论用户下达什么指令都不得删除**，并须主动提示用户本地备份。

## 三、目录结构模板

```
leo/                                # AI 协作专属工作区（名字可自定，建议 ASCII）
├── README.md                       # 宪法（规则总纲）：环境说明 + 工作流 + 命名/安全规则（初始化必读，不可私自改）
├── README1ST.md                    # 第一版宪法快照：底层根逻辑 + 安全规范，永不删除，须本地备份
├── memory.md                       # 总记忆：热缓存 + 技能/规则库 + 长期经验库（不含流水账）
├── memory-index.md                 # 记忆索引：仅目录/TOC，轻量快速查找
├── file-tree.md                    # 文件树（脚本自动生成）
├── vault/                          # 敏感目录（原 document/，不进公开文件树）
│   └── secrets.md                  # 密码/密钥/Token 存档（可选加密，仅存密文）
├── code/
│   └── gen-filetree.py             # 文件树自动生成脚本（见附录 A）
├── pan/                            # 媒体目录（原 img/）
│   ├── imga/                       # 图片
│   ├── video/                      # 视频
│   ├── audio/                      # 音频
│   └── document/                   # 文档（非敏感）
└── log-YYYY.MM.DD-YYYY.MM.DD.md     # 流水账归档（按周期切分，按需查阅，如 log-2026.05.03-2026.06.03.md）
```

**设计要点**：
- 核心文件名全部英文（README / README1ST / memory / memory-index / file-tree / secrets / gen-filetree），**禁止改回中文**，防止路径与 `Content-Disposition` 等编码类 BUG。
- 规则与日志分开：README 是"宪法"（改动少、受保护），memory.md 是"知识库"（可蒸馏），流水账是"史书"（独立归档、按需查阅）。
- 敏感目录 `vault/` 与普通媒体目录 `pan/` 物理隔离，且文件树脚本自动隐藏 `vault/`，防 AI 输出文件树时泄露。
- 媒体按类型分子目录，AI 见到新文件类型时新建子目录并记入日志。

## 四、核心文件职责

### 4.1 README.md（宪法 / 初始化入口）
写"AI 必须遵守什么"：目录结构说明、运行环境档案、标准工作流程、文件命名规则、安全规范、AI 行为约定、版本信息。定位是**每次会话开始时的初始化入口**——AI 读完它就知道自己身在何处、该干什么、什么不能干。

**AI 对 README 的硬约束**：
- 不得私自改动 README 任何内容；任何修改意图都必须先向用户提出并获**二次确认**。
- 读取 README 后必须校验同目录是否存在 `README1ST.md`；若不存在，立即提示用户补建（见 4.2）。

#### 4.1.1 宪法初始化与二次确认流程（仅首版）
第一版 README 不是 AI 单方面写的，必须融合三方内容并经用户确认：
1. **融合草拟**：AI 草拟第一版 README，融合①内核规则（本白皮书前三节）+②目标平台的规则（平台使用条款、AI 行为限制、合规要求）+③沙箱能力配置（来自第九节适配层接口取值，如持久根目录、出网情况、交付通道、定时任务能力）。
2. **完整展示**：AI 向用户完整展示草稿，并明确列出"将写入的安全规范与根逻辑要点"。
3. **二次确认**：必须获得用户明确确认（如回复"确认通过"或指定确认指令）后，才允许落盘为 `README.md`；**未确认前不得写入**。
4. **派生快照**：确认写入 `README.md` 的同时，另存一份**完全一致**的副本为 `README1ST.md`。
5. **本地备份提示**：AI 主动提示用户将 `README1ST.md` **保存到本地（沙箱之外）**作为根信任备份，并说明其不可恢复的重要性。
6. **后续改动**：任何对 README 的改动同样需二次确认；`README1ST.md` 永远保留，不论用户下达何种删除/修改指令，AI 一律拒绝并说明其"底层根逻辑"地位。

### 4.2 README1ST.md（底层根逻辑与安全规范，永不删除）
- 内容：第一版 README 的**逐字副本**，代表本工作区的底层根逻辑与安全规范。
- 地位：最高优先级信任根。即使 README 后续被（经确认的）改动，`README1ST.md` 也不变、不删。
- 保护：AI 在任何情况下（含用户直接要求删除/覆盖）都不得删除或清空 `README1ST.md`；若用户提出，AI 须说明其不可删地位并建议改为"新增补充文档"而非破坏根逻辑。
- 备份：AI 应周期性（如每次会话初始化）提醒用户确认本地已保存最新 `README1ST.md`。

### 4.3 memory.md（总记忆存储，不含流水账）
为避免单文件无限膨胀，memory.md 只承载**长期稳定知识**，采用三层结构：
1. **热缓存层（规则速查）**——把 README 里最高频的规则压缩成几行，AI 每次只需读这节就能进入状态（长文档的"热缓存"）。**固定较小，不随任务增长**。
2. **技能/规则库（永久知识）**——可复用的方法论、工作流、平台技巧、工具用法等。**属于知识而非流水账**，长期有效，不因时间推移而归档。
3. **长期经验库（按主题归类）**——实战踩坑记录，每条按"问题 → 解法 → 适用性"三段式写，标注来源环境。按主题（如 `网络 / 文件交付 / 编码 / 性能 / 安全`）归类，而非按日期堆积。

> **重要区分**：技能与规则**不属于流水账**，常驻 memory.md；流水账（像日志记录一样）见 4.5 独立归档，不写入 memory.md。

**写记忆的纪律**：
- 技能/规则库与长期经验库可编辑、可归并（蒸馏时合并同类）。
- 宁可琐碎不可遗漏；新经验解决完问题立刻写回，不能"等有空再补"。
- 每月至少一次**蒸馏**：把零散经验沉淀为主题条目，更新 `memory-index.md`。

### 4.4 memory-index.md（记忆索引，仅目录）
- 作用：memory.md 可能极其庞大，AI **不应每次全量读取**。索引文档只提取"目录/TOC"——每条记忆的`编号 | 关键词 | 归属文件/章节 | 日期`，用于轻量快速查找。
- 内容形态：纯索引，不含正文。例如：
  ```
  [E-012] nginx反代302跳出 | memory.md §长期经验库/网络 | 2026-06-10
  [S-003] 沙箱出网白名单采集法 | memory.md §技能库 | 2026-07-02
  [L-2026.05] 5月项目流水 | log-2026.05.01-2026.05.31.md | 归档
  ```
- 维护：每次 memory.md 或归档变更后同步更新本文件；AI 会话初始化先读 `memory-index.md`，按索引定点调阅，需要时才读 `memory.md` 对应章节或某个 `log-*.md`。
- 体积目标：控制在数 KB 内，确保每次初始化零负担。

### 4.5 流水账归档 log-*.md（按需查阅）
- 形态：按时间周期切分的独立文件，命名 `log-YYYY.MM.DD-YYYY.MM.DD.md`（如 `log-2026.05.03-2026.06.03.md`）。**文件名主体必须英文/数字，禁止中文**。
- 内容：像日志记录一样，按日期倒序追加每个任务"干了什么、改了什么、结论是什么"。**只保留当前周期的活动日志，周期结束即封板归档**。
- 查阅策略：**默认不读**，仅在"需要回溯某段历史"或"索引指向"时才调阅，避免占用上下文。
- 生命周期：归档文件长期保留但冷存储；超长周期可进一步合并为 `log-YYYY.md` 年档。

### 4.6 file-tree.md（文件树）
由脚本自动扫描生成，让 AI 不用逐个 `ls` 就掌握全局资产。生成规则：隐藏敏感目录 `vault/`（只留占位符）、标注目录/文件图标、统计大小、忽略缓存类文件。

## 五、任务闭环工作流（五步）

```
存文件 → 执行任务 → 更新索引与文件树 → 记录日志 → 大变动打包
```

1. **存文件**：用户上传的文件先按类型归位到对应子目录（统一英文改名，见第七节）。
2. **执行任务**：处理用户的实际请求。
3. **更新索引与文件树**：运行 `python3 code/gen-filetree.py` 刷新文件树；如 memory.md/归档有变动，同步更新 `memory-index.md`。
4. **记录日志**：操作、新经验、踩坑记录追加到**当前周期的 `log-*.md`**（**最容易被偷懒省略、也最不能省的一步**）；可复用的结论同时蒸馏进 memory.md 并更新索引。
5. **大变动打包**：工作区发生结构性大变化时打包 zip 备份；轻量包（规则+代码+文档，MB 级）与完整数据包（含媒体，GB 级）分开。**打包产物可直接作为新会话的初始化输入**——开新对话时把轻量包发过去，AI 解包即恢复规则与经验，实现"新环境快速启动"。

## 六、生命周期与触发（内核定动作，适配层定方式）

内核只定义**三个生命周期动作及必须完成的内容**；具体"靠什么触发"由各平台适配层实现。

| 生命周期动作 | 必须完成的内容 | 触发方式（适配层负责） |
|------|--------|---------|
| **会话初始化** | ①校验 `README1ST.md` 存在 ②读 README + memory.md 热缓存 + memory-index.md（按需定点读 memory.md 章节或 `log-*.md`）③或解包上次轻量备份 | 平台支持会话开始 hook → 自动；否则由适配层在系统提示/首条指令注入读取动作 |
| **任务收尾** | 走第五步工作流（更新索引/文件树 + 记录日志） | 适配层把收尾动作写入任务完成后的标准指令；平台支持任务结束 hook 更佳 |
| **重大变更** | 结构性改动（增删目录、改规则、引入新文件类型）时打包备份 | 适配层定义"何为重大变更"的判据 |

**注意**：人工暗号（如"1"保活、"看文件树"）属于适配层在"平台缺乏原生能力"时的权宜手段，**不进入内核**；能自动化的尽量自动化。

## 七、文件命名规范

- **核心文件名固定英文**：`README.md` / `README1ST.md` / `memory.md` / `memory-index.md` / `file-tree.md` / `secrets.md` / `gen-filetree.py` —— AI 不得擅自改回中文，防止编码类 BUG。
- **默认格式**：`日期_时间戳_三位随机数.扩展名`，如 `2026-10-27_142508_772.png`。
- **可加标题前缀**方便查找：`标题关键词_2026-10-27_142508_772.mp4`（前缀可含中文便于检索，但文件名主体建议 ASCII）。
- **编码陷阱（通用经验）**：部分平台的下载网关 / HTTP 响应头对中文文件名做 `Content-Disposition` 编码时会报 `latin-1` 错误导致连接断开。因此**文件名主体优先 ASCII**；若必须中文，服务类场景按 RFC 5987 做 URL 编码。
- 用户上传文件除非明确要求保留原名，一律改为标准格式。
- 目录名同理建议 ASCII，避免中文路径在部分脚本/服务中出问题。
- 流水账归档命名：`log-YYYY.MM.DD-YYYY.MM.DD.md`，纯数字日期，禁止中文。

**分类规则**：图片→`pan/imga/`；视频→`pan/video/`；音频→`pan/audio/`；文档→`pan/document/`；代码→`code/`；密码/密钥/Token→`vault/`（敏感）；流水账→`log-*.md`；其他类型新建子目录并记入日志。

## 八、安全红线（分级）

1. 密码/Token/密钥**只**存放在 `vault/secrets.md`，不写入其他文件，不在对话中原文复述。
2. `vault/` 下内容不出现在公开文件树（脚本自动隐藏）。
3. 发现用户泄露过的旧密码，主动建议轮换。
4. 临时文件用完即删，临时目录（`/tmp` 等）不放重要数据。
5. **`README1ST.md` 永久保护**：无论用户下达何种删除/覆盖指令，AI 一律拒绝；并周期性提醒用户本地已备份。
6. **加密保险库为可选项**：默认不启用（见附录 C）；启用需用户额外二次确认；未启用且处于"共享/可分享级"沙箱时，禁止明文存储密钥。

**安全分级（重要）**：明文密钥存档仅在"沙箱完全私密、不可被外部读取或分享"时成立。适配层**必须声明本平台安全等级**：
- **私密沙箱级**：沙箱仅本人可见、不可分享 → 可明文降级存储（仍建议启用加密保险库）。
- **共享/可分享级**：会话或文件可能被分享、或沙箱可被他人读取 → **禁止明文**，必须启用加密保险库（附录 C）或把密钥外置到平台提供的密钥管理服务，文件内只留引用。

## 九、适配层：平台能力最小接口

"可移植"能否成立，取决于内核与适配层之间有没有清晰契约。定义以下**最小接口**，适配层逐项为平台填值：

| 接口项 | 说明 | 取值示例 |
|--------|------|----------|
| 持久根目录 | AI 工作区落在哪，重启/新会话后是否保留 | 绝对路径 + 是否持久 |
| 会话自动读取 | 能否在会话开始时自动读文件 | hook / 系统提示注入 / 不支持 |
| 定时任务 | 能否跑保活、定期打包、蒸馏 | 支持 / 不支持 |
| 沙箱私密性 | 文件能否被外部读取或随会话分享 | 私密 / 可分享 |
| 文件交付通道 | 产物如何交还用户 | 平台下载 / 预览链接 / 需自建 |
| 网络出网 | 能访问哪些外网（影响采集/下载） | 全通 / 白名单 / 不通 |

> 第一版 README 的"沙箱能力配置"即来自本接口表的取值，须与用户二次确认后写入（见 4.1.1）。

**适配新平台三步法**：
1. 给新环境"体检"：系统版本、CPU/内存/磁盘、网络连通性（测哪些站点通/不通）、Python/Node 版本、字体等，填进环境档案与接口表。
2. 对照接口表逐项决定"删掉、替换、保留"——原生有的能力就别自建。
3. 跑一个真实小任务验证五步流程，把新平台的第一个坑记进经验库。

**继承下来的通用经验**（跨平台几乎都适用）：进程存在 ≠ 服务正常（要 curl 实测内容）；常用端口易被占用（自建服务用 9000+）；临时目录不可靠；单线程服务器要换多线程；给 AI 的自动化指令必须无歧义且带验证步骤；动态渲染页面 curl 抓不到；数据文件定期核对去重。

## 十、可选增强（不在主流程，按需启用，附推荐方案与详细教程）

以下能力**默认不启用**，仅当平台**缺少原生等价能力**或用户明确需要时考虑。各选项启用前必须向用户说明风险并记入适配层说明。每一项均给出**推荐方案 + 详细教程 + 风险与注意**。

### 10.1 内网穿透 / 隧道（Leo 实测方案 · cloudflared）

**用途**：平台无自带预览/外网访问、且需对外提供本地服务（Web / API / 文件下载）时使用。把运行在沙箱/内网的本地服务暴露到公网，让外部用户或另一平台能稳定访问。

**推荐方案**：选 `cloudflared`（Cloudflare Tunnel）。
**为什么是它（实测结论）**：Cloudflare 官方开源项目（Go 编写，GitHub: `cloudflare/cloudflared`），免费，不需要公网 IP、不开端口、不怕 NAT/防火墙——隧道是**反向出站长连接**，几乎在任意网络环境下都能建立；流量走 Cloudflare CDN，自带 HTTPS 并隐藏真实 IP。方案本身可行性已在多个沙箱环境实测通过。

> **原理一句话**：cloudflared 跑在内网机器/沙箱上，由它**主动向外**连接 Cloudflare 边缘节点建立反向通道：
> `外网访客 → 你的域名(Cloudflare CDN) → 隧道(出站长连接) → cloudflared → 内网服务(localhost:9000)`

**详细教程**

**用法① 临时体验（一行命令，无需账号域名）**
```bash
cloudflared tunnel --url http://localhost:9000
```
立刻得到 `xxx.trycloudflare.com` 随机临时域名，进程关掉即失效。适合临时给人看一眼。**注意本地服务先跑在 9000+ 端口**。

**用法② 长期正式版（推荐，常驻时使用）**
前置条件：一个托管在 Cloudflare 的域名（免费套餐足够）。

**方式 A：命令行自管隧道**
```bash
# 1. 安装（Debian/Ubuntu）
curl -L --output cloudflared.deb https://github.com/cloudflare/cloudflared/releases/latest/download/cloudflared-linux-amd64.deb
sudo dpkg -i cloudflared.deb

# 2. 授权并创建隧道
cloudflared tunnel login
cloudflared tunnel create leo-agent        # 生成凭证文件 ~/.cloudflared/<UUID>.json

# 3. 写配置 ~/.cloudflared/config.yml
#    tunnel: <隧道UUID>
#    credentials-file: /root/.cloudflared/<UUID>.json
#    ingress:
#      - hostname: agent.yourdomain.com
#        service: http://localhost:9000
#      - service: http_status:404

# 4. 绑定 DNS（自动建 CNAME）
cloudflared tunnel route dns leo-agent agent.yourdomain.com

# 5. 启动 / 注册系统服务（开机自启）
cloudflared tunnel run leo-agent
sudo cloudflared service install
```

**方式 B：后台网页托管隧道（更省事，推荐）**
1. Zero Trust 控制台（one.dash.cloudflare.com）→ Networks → Tunnels → Create tunnel
2. 复制页面给出的一行 token 命令，在沙箱/机器上执行：`cloudflared tunnel run --token eyJ...`
3. 回网页 Public Hostname 里点选：子域名 + 本地服务地址（如 `http://localhost:9000`）
4. 完成，DNS 与证书全自动管理

**实战经验（Leo 实测，务必遵守）**
1. **进程存在 ≠ 服务正常**：cloudflared 进程活着不代表隧道通，用 `curl https://agent.yourdomain.com` 实测返回内容。
2. **内网服务也要实测**：先 `curl http://localhost:9000` 确认本地服务本身正常，再排查隧道（分层定位）。
3. **端口选择**：本地服务避开 8080/8000 等常用端口，用 9000+；启动前 `ss -tlnp | grep <端口>` 查占用。
4. **保活**：常驻务必 `sudo cloudflared service install` 注册 systemd 服务，别裸跑 nohup。
5. **多服务复用一条隧道**：ingress 规则里写多条 hostname → service 映射即可，不必一条隧道一个服务。
6. **凭证管理**：`<UUID>.json` 凭证与 `--token` 属于敏感信息，只存 `vault/secrets.md`（见第八节安全红线），**绝不可进代码仓库或明文日志**。

**frp 备选（已有 VPS 时）**
- 服务端 `frps.toml` 设 `bindPort` + `auth.token`；`frps -c frps.toml` 启动。
- 客户端 `frpc.toml` 设 `serverAddr`(VPS IP) + `serverPort` + `auth.token` + `[[proxies]]`（`localPort = 9000`、`remotePort = 9000`）；`frpc -c frpc.toml` 启动。
- 外部 `http://<VPS-IP>:9000` 访问本地服务。延迟更低、完全可控，但需自有公网服务器与端口放行。

**风险与注意**
- 暴露面扩大，**必须加鉴权**（agent 自身登录 / Token / 基础认证），绝不裸奔。
- 隧道凭证（token / credentials-file / frp token）走 `vault/` 管理，不写明文日志、不进 memory。
- 非必要时关闭隧道；公网域名/端口是持续攻击面。
- 不同平台的预览/网关对隧道域名的抓取能力不同（如是否跟随 3xx 重定向、能否渲染 SPA 正文），需实测验证后再依赖。
- 免费隧道有速率/连接数限制，长驻服务评估用量。

### 10.2 保活机制
**用途**：平台会杀空闲会话时维持会话存活。
**推荐方案**：优先用**平台原生定时调度**（cron / 心跳任务）；无原生能力再用"对话暗号 + 极简保活指令"。
**详细教程**：
1. 平台定时任务：建每日极简问候（如 08:00 向会话发送 `1`），任务指令**必须写明「极简回复，禁止读文件执行任务」**。
2. 无 cron：用轻量 keepalive 脚本每 ~30 分钟触发一次最小请求维持会话。
3. 验证：观察会话是否保持活跃、保活任务是否误触发了文件读取/任务执行。
**风险与注意**：暗号保活若指令不严格，会误触发 AI 读文件/执行任务，**token 爆量**；定期审查保活任务内容。

### 10.3 自建文件服务器
**用途**：平台无原生下载通道时交付大文件。
**推荐方案**：本地快速用 `http.server` 仅限内网/临时；对外或对性能要求用 **Caddy `file-server`**（自带 HTTPS、高性能、自动索引）或 Flask + 多线程。
**详细教程**：
1. 临时内网：`python3 -m http.server 9000 --directory ./pan`（单线程，压测/并发会挂，仅临时）。
2. 生产：`caddy file-server --root ./pan --address :9000`（绑域名可自动 HTTPS）或写 Flask 多线程接口。
3. 如需鉴权：Caddy / Nginx 加 basic auth 或 header Token；密钥走 `vault/`，建议启用加密保险库（附录 C）。
**风险与注意**：大文件耗带宽；明文服务别直接暴露公网；端口用 9000+。

### 10.4 定时打包/备份
**用途**：结构性变动后做快照，产物可作新会话初始化输入。
**推荐方案**：脚本化 `zip` + 平台定时任务；轻量包与完整包分离。
**详细教程**：
1. 轻量包（MB 级）：`zip -r leo-light-$(date +%F).zip README.md README1ST.md memory.md memory-index.md code/ vault/`
2. 完整包（GB 级，含媒体）：`zip -r leo-full-$(date +%F).zip leo/`
3. 接平台 cron 每日/每周执行；轻量包发到新会话即可快速恢复规则与经验。
**风险与注意**：完整包含媒体可能 GB 级，注意存储配额；`vault/` 若明文需确认处于私密沙箱级。

### 10.5 加密保险库（可选项，详见附录 C）
**用途**：对 `vault/` 内密钥做加密，仅存密文。
**推荐方案**：AES-256-GCM + PBKDF2（附录 C），**默认关闭**，启用需用户额外二次确认。
**详细教程**：①与用户确认启用 → ②生成钥匙包（passphrase / `.leopass`）并**本地备份** → ③按附录 C 格式写入密文条目 → ④AI 读取时按 C.3 协议现场索取 passphrase、用完即弃、不记忆。
**风险与注意**：钥匙包丢失 = 数据永久不可恢复；务必本地备份；仅保护静态存储。

### 10.6 离线/本地密钥包管理
**用途**：进一步降低泄露面，密钥材料不出本地。
**推荐方案**：钥匙包由用户本地保管（类似 leoi.org/pass 的 `.leopass`），沙箱 `secrets.md` 只留密文与 `key_ref`。
**详细教程**：用户本地用 leoi.org/pass 或等价工具生成并备份钥匙包；沙箱仅存密文条目；AI 需读取时用户现场提供对应钥匙包 passphrase。
**风险与注意**：同 10.5；本地备份责任在用户，AI 须反复提醒。

### 10.7 多工作区切换
**用途**：同一 agent 管理多个项目，各自独立记忆。
**推荐方案**：顶层索引 + 每个项目独立 `leo/` 风格目录。
**详细教程**：建 `workspaces/index.md` 列出各项目根路径与 README 链接；切换项目时加载对应 `README` + `memory` + `memory-index`，上下文隔离。
**风险与注意**：防止跨项目记忆污染；索引需随项目增删维护。

### 10.8 增量同步
**用途**：跨设备/跨平台搬运工作区，省带宽且保留历史归档。
**推荐方案**：规则/代码用 `git` 版本化；大文件/媒体用 `rsync` / `rclone` 增量同步。
**详细教程**：`rsync -a --delete leo/ remote:leo/`（增量搬运）；规则代码 `git init` 后提交；媒体大文件用 `rclone sync`。
**风险与注意**：同步冲突需人工裁决；`vault/` 同步须走加密通道或本地。

## 十一、快速上手清单（新平台从零到可用）

- [ ] 建目录骨架（第三节模板，全英文文件名）
- [ ] 草拟第一版 README（融合内核规则 + 平台规则 + 沙箱能力配置），**经用户二次确认**后落盘，并派生 `README1ST.md`
- [ ] 主动提示用户将 `README1ST.md` 本地备份
- [ ] 写 memory.md（热缓存 + 空技能库/经验库），建 `memory-index.md`（空索引）
- [ ] 放入 gen-filetree.py（附录 A，确认 `vault/` 隐藏规则生效）
- [ ] 跑一次 `python3 code/gen-filetree.py` 验证文件树生成
- [ ] 在适配层声明安全等级（私密 / 可分享）
- [ ] 与内核对齐三个生命周期动作的触发方式
- [ ] 用一个真实小任务完整走一遍五步工作流（含写一条 `log-*.md` 与更新索引）
- [ ] 如平台支持定时任务，配置保活与定期蒸馏（指令须极简，避免误触发任务）
- [ ] 如需存储密钥且非私密沙箱，与用户确认启用加密保险库（附录 C）

---

## 附录 A：gen-filetree.py（完整内联，纯标准库无依赖）

> 功能：递归扫描工作区 → 生成带图标和大小标注的文件树 → 自动隐藏敏感目录 `vault/` → 过滤缓存文件。
> 移植时只需改两个常量：`ROOT`（工作区根目录）与 `SECRET_DIRS`（敏感目录名单）。

```python
#!/usr/bin/env python3
# -*- coding: utf-8 -*-
"""
gen-filetree.py - 扫描工作区生成带图标和大小标注的文件树
- 自动隐藏敏感目录（SECRET_DIRS，默认 vault）
- 过滤缓存/临时文件（IGNORE_DIRS / IGNORE_FILES / IGNORE_EXTS）
- 输出到 file-tree.md

移植时只需改两个常量：ROOT（工作区根目录）与 SECRET_DIRS（敏感目录名单）。
"""
import os
import sys
from datetime import datetime

# ===== 只需修改这两个常量 =====
ROOT = os.path.dirname(os.path.dirname(os.path.abspath(__file__)))  # 默认指向脚本上级目录（leo/）
# 若作为独立脚本使用，可改为：ROOT = "/path/to/leo"
SECRET_DIRS = {"vault"}            # 敏感目录（不展开，只留占位符）
# =============================

IGNORE_DIRS = {".git", "__pycache__", "node_modules", ".cache", "tmp"}
IGNORE_FILES = {".DS_Store", "Thumbs.db", "file-tree.md", "memory-index.md"}
IGNORE_EXTS = {".pyc", ".tmp", ".log"}
OUTPUT = os.path.join(ROOT, "file-tree.md")

ICON_DIR = "📁"
ICON_FILE = "📄"
ICON_SECRET = "🔒"


def human_size(num):
    for unit in ["B", "KB", "MB", "GB", "TB"]:
        if num < 1024:
            return f"{num:.1f}{unit}" if unit != "B" else f"{num}B"
        num /= 1024
    return f"{num:.1f}PB"


def dir_size(path):
    total = 0
    for root, dirs, files in os.walk(path):
        dirs[:] = [d for d in dirs if d not in IGNORE_DIRS and d not in SECRET_DIRS]
        for f in files:
            if f in IGNORE_FILES or os.path.splitext(f)[1].lower() in IGNORE_EXTS:
                continue
            try:
                total += os.path.getsize(os.path.join(root, f))
            except OSError:
                pass
    return total


def walk(dir_path, prefix=""):
    lines = []
    try:
        entries = sorted(os.listdir(dir_path))
    except PermissionError:
        return lines
    dirs = [e for e in entries if os.path.isdir(os.path.join(dir_path, e))]
    files = [e for e in entries if os.path.isfile(os.path.join(dir_path, e))]
    items = [(e, True) for e in dirs] + [(e, False) for e in files]
    for idx, (name, is_dir) in enumerate(items):
        last = idx == len(items) - 1
        branch = "└── " if last else "├── "
        full = os.path.join(dir_path, name)
        if is_dir and name in SECRET_DIRS:
            lines.append(f"{prefix}{branch}{ICON_SECRET} {name}/  [敏感目录·已隐藏内容]")
            continue
        if is_dir:
            if name in IGNORE_DIRS:
                continue
            size = dir_size(full)
            lines.append(f"{prefix}{branch}{ICON_DIR} {name}/  ({human_size(size)})")
            ext = "    " if last else "│   "
            lines.extend(walk(full, prefix + ext))
        else:
            if name in IGNORE_FILES or os.path.splitext(name)[1].lower() in IGNORE_EXTS:
                continue
            try:
                size = os.path.getsize(full)
            except OSError:
                size = 0
            lines.append(f"{prefix}{branch}{ICON_FILE} {name}  ({human_size(size)})")
    return lines


def main():
    if not os.path.isdir(ROOT):
        print(f"[错误] 根目录不存在: {ROOT}")
        sys.exit(1)
    lines = [f"# 文件树（自动生成 · {datetime.now():%Y-%m-%d %H:%M}）", "",
             f"{ICON_DIR} ./", ""]
    lines.extend(walk(ROOT))
    content = "\n".join(lines) + "\n"
    with open(OUTPUT, "w", encoding="utf-8") as f:
        f.write(content)
    print(f"[完成] 已生成 {OUTPUT}（{len(lines)} 行）")


if __name__ == "__main__":
    main()
```

## 附录 B：跨平台验证结论（通用经验，无平台绑定）

- 已在多类 AI 沙箱环境验证：方案跨平台可用，日志回读/打包机制能让新会话 AI 在 1–2 分钟内恢复全部上下文；轻量索引使庞大记忆也可低成本初始化。
- **通用技术坑（写入经验库时标注来源环境即可，不绑定具体平台）**：
  - 进程存在 ≠ 服务正常（要 curl 实测返回内容，而非只看端口）。
  - 常用端口易被占用，自建服务优先用 9000+。
  - 临时目录（`/tmp` 等）不可靠，重要数据别只放那。
  - 单线程开发服务器压测/并发会挂，上线换多线程。
  - 给 AI 的自动化指令必须无歧义且带验证步骤，否则易跑偏。
  - 动态渲染（SPA）页面 curl 抓不到正文，需换采集策略。
  - 数据文件定期核对去重，避免重复累积。

## 附录 C：加密保险库方案（可选项，参考 leoi.org/pass）

> 本模块**默认关闭**。启用需用户**额外二次确认**。设计参考 [Leo 加密系统](https://www.leoi.org/pass)：AES-256-GCM + PBKDF2，以"钥匙包"为唯一解密凭证。

### C.1 设计目标与约束
- `vault/secrets.md` **只存密文**，不存明文密钥。
- 一条密文记录的结构对应你提出的：`用户密钥 + 不可逆加密解密算法 + 密文`。
  - **用户密钥**：解密所需的"钥匙包"（用户掌握的 passphrase 或 `.leopass` 文件），沙箱内不留存。
  - **不可逆加密解密算法**：`PBKDF2-HMAC-SHA256`（单向 KDF，由 passphrase 派生密钥，不可逆）+ `AES-256-GCM`（对称加密，提供保密与完整性）。
  - **密文**：AES-256-GCM 输出（nonce + ciphertext + tag）。
- AI 想读取某条密码时，**必须向用户索取对应钥匙包/passphrase**；AI 仅本次会话内存中使用，使用完毕立即丢弃，**不得写入 memory.md / 日志 / 任何持久文件，也不得在后续对话中复述该密码**。

### C.2 secrets.md 密文条目格式（启用加密后）
```markdown
## github_token
- algo: AES-256-GCM+PBKDF2
- kdf_iter: 600000
- salt: <base64>
- nonce: <base64>
- cipher: <base64>
- key_ref: leopass_main        # 指明用哪个钥匙包解密
```
（未启用加密时，本文件按第八节安全分级以明文或引用方式存储，仅限私密沙箱级。）

### C.3 AI 读取协议（铁律）
1. AI 需要某密钥 → 明确告知用户"需提供钥匙包 `<key_ref>` 的 passphrase 才能解密 `<entry-id>`"。
2. 用户在**当前会话内**提供 passphrase；AI 现场 PBKDF2 派生密钥 → AES-256-GCM 解密 → 取得明文并使用。
3. 使用结束后，AI **立即弃用**该 passphrase 与明文，不得落盘、不得记入 memory/日志、不得在任何持久位置保留。
4. 若用户未提供或提供错误，AI 直接说明"无法读取，需钥匙包"，不猜测、不绕过。

### C.4 风险与备份
- **钥匙包丢失 = 数据永久不可恢复**（与 leoi.org/pass 一致）。启用时 AI 必须强烈提示用户将钥匙包**本地备份**（沙箱之外）。
- 加密仅保护静态存储；传输/对话中的一次性使用仍由用户现场提供保证。

## 附录 D：README 与 README1ST 首版模板（骨架）

> 首版 README 建议结构（融合内核规则 + 平台规则 + 沙箱能力配置，经二次确认后落盘，并复制为 README1ST）。

```markdown
# README（本工作区宪法）
## 0. 版本与根信任
- 本文件为宪法，AI 不得私自改动；首次确认日期：____；底层根逻辑见 README1ST.md
## 1. 目录结构与命名规则
（引用第三节模板 + 第七节命名规范）
## 2. 平台规则与合规
（目标 agent 平台的使用条款、AI 行为限制）
## 3. 沙箱能力配置（来自适配层接口表）
- 持久根目录：____
- 会话自动读取：____ ｜ 定时任务：____ ｜ 沙箱私密性：____
- 文件交付通道：____ ｜ 网络出网：____
## 4. 标准工作流程
（引用第五节约五步工作流 + 第六节气生命周期）
## 5. 文件命名规则
（引用第七节）
## 6. 安全规范
（引用第八节 + 加密保险库开关状态：开/关）
## 7. AI 行为约定
（会话初始化读什么、任务收尾做什么、诚实/止损原则）
## 8. 变更记录
- v1.0 ____ 首次确认生效
```

---

**文档版本**：v2.3 ｜ **整理时间**：2026-10-28 ｜ **性质**：可移植通用版白皮书（内核平台无关，差异收敛至适配层）｜ **首创**：[LeoLee](https://s.iee.li/l/u1)
