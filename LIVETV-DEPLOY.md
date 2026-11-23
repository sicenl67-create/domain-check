# livetv Stremio 直播插件 · 部署说明

这个分支只用来触发部署，仓库里的其它文件对部署没有影响。

- Worker 名称：`livetv`，线上地址 `https://livetv.sicenl67.workers.dev`
- 代码来源：`jyotiprakash101/horizon-home-kit` 的 `livetv-addon/worker-bundle.js`（在 Actions 运行时下载），
  现在把打包好的版本直接提交为仓库根目录的 `livetv-worker.js`（`wrangler.toml` 的 `main` 指向它），
  部署读的是仓库里这份文件，不再在运行时下载。
- 重新部署/更新：Actions → 「自动部署到 CF Worker」→ Run workflow → 分支选 `livetv-deploy`
- 首次部署：2026-11-23，run 45120755600
- 短链接改造：run 45121944506（提交 `177b794`）
- 11 国/地区 + 按国家筛选：2026-11-23，run 45123427738（提交 `240dfb6`）

## 默认配置与短链接（2026-11-23 加入）

`livetv-worker.js` 的路由里内置了一段默认配置 `DEFAULT_SEG`
（11 个国家/地区、`tz:"Asia/Shanghai"`、`name:"直播电视"`、rows 四行、`clean`、`days:3`、`maxStreams:3`）。
请求路径不带配置段、且首段属于 `BARE_ROUTES`（`manifest.json`、`configure`、`catalog`、
`meta`、`stream`、`playlist.m3u`、`playlist.m3u8`、`epg.xml.gz`、`channels.json`、`now.json`）
时自动回落到这段配置，不再返回 `{"error":"bad configuration"}`。

- Stremio 安装链接：`https://livetv.sicenl67.workers.dev/manifest.json`
- 配置页（已预填 11 国配置）：`https://livetv.sicenl67.workers.dev/configure`
- TiviMate 等 IPTV 播放器：`https://livetv.sicenl67.workers.dev/playlist.m3u`
- 根路径 `/` 仍 302 到 `/configure`
- 带自定义 `<config>` 段的老链接行为不变（`/<config>/manifest.json`、`/<config>/catalog/...` 等）

改默认配置只要改 `DEFAULT_SEG` 那一行常量：它是 `base64url(JSON)`，用
`python3 -c "import base64,json;print(base64.urlsafe_b64encode(json.dumps({...},separators=(',',':')).encode()).decode().rstrip('='))"` 生成。

### 2026-11-23 第二次调整：11 国 + 按国家筛选

`DEFAULT_SEG` 从 `cc:["cn"]` 改成 11 个国家/地区：
`cn` 中国、`hk` 香港、`tw` 台湾、`jp` 日本、`kr` 韩国、`us` 美国、`uk` 英国、
`th` 泰国、`au` 澳大利亚、`fr` 法国、`es` 西班牙（**上游数据里英国的代码是 `uk`，不是 ISO 的 `gb`**，
`/api/countries` 里没有 `gb` 键）。合计约 4,521 个频道。

为了方便“想看哪个国家就选哪个国家”，代码里加了三处改动：

1. `manifest()`：国家在 `genre` 选项里用中文标签（`🇨🇳 中国`、`🇺🇸 美国`…，见 `CC_NAMES_ZH` / `ccLabel()`），
   与节目类型（News/Sports…）并列；`genreFilter()` 同时认中文标签和旧的英文标签。
2. `manifest()`：每个国家再加一行独立 catalog `nltv_cc_<cc>`（`直播电视 · 🇨🇳 中国` 等），
   在 Stremio 的「发现」里一键就能只看某个国家；`catalog()` 遇到这一行只拉该国数据。
3. `allChannels()`：各国频道改成轮流交错（round-robin），一页 100 条里每个国家都能看到，
   不再是中国频道排满前几页。

### Free 计划的 10ms CPU 坑（重要）

频道数从 164 涨到 4,521 后，上游原版代码在 Free 计划上会报 `error code: 1102`（CPU 超时）：
原因是它把**所有**符合条件的节目都渲染成 meta（每条要跑好几次 `Intl.DateTimeFormat`），
`nltv_tonight` 还对每个节目各算一次 `dayKey()`/`localHour()`。改法：

- catalog 一律先筛、排序、`slice(skip, skip+100)`，**只给这一页**的条目生成 meta；
- “今晚”改成每个请求只算一次 19:00–24:00 的数字时间窗（`tonightWindow()`）；
- `nowMap()` 改成 `Map<cc, progs>`，不再给每个频道拼一个 `cc:id` 字符串键；
- `allChannels()` 直接给缓存对象打 `cc` 标记，不再为每个频道 `{...ch}` 复制一次。

改完 `now/soon/tonight/channels/搜索` 都能在 Free 计划上稳定 200（单次 11 个国家数据约 9.8MB JSON）。
验证方式：先用 `livetv-test` Worker（分支 `livetv-test`，`wrangler.toml` 里 `name = "livetv-test"`）
跑一遍，再合到 `livetv-deploy` 部署 `livetv`；`livetv-test` 用完即删。

## 那个部署工作流的副作用

`自动部署到 CF Worker` 会把 domain-check 的 `PASSWORD` secret 一起注入目标 Worker（日志里 `vars: PASSWORD`）。
每次部署后要到 Cloudflare 面板 → livetv → 设置 → 运行时变量和密钥，把 `PASSWORD` 删掉（删除并部署）。
2026-11-23 三次部署后都已清理，现在变量列表为空。

## 网络

注意：`*.workers.dev` 在中国大陆被 DNS 污染 + TCP 阻断（2026-11-23 用 5 个国内探测点实测全部失败）。
国内要直连需给 Worker 绑定自定义域（例如 `tv.lkjhgf.de5.net`），绑定后从国内探测点 3/5 可通。
