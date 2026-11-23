# livetv Stremio 直播插件 · 部署说明

这个分支只用来触发部署，仓库里的其它文件对部署没有影响。

- Worker 名称：`livetv`，线上地址 `https://livetv.sicenl67.workers.dev`
- 代码来源：`jyotiprakash101/horizon-home-kit` 的 `livetv-addon/worker-bundle.js`（在 Actions 运行时下载），
  现在把打包好的版本直接提交为仓库根目录的 `livetv-worker.js`（`wrangler.toml` 的 `main` 指向它），
  部署读的是仓库里这份文件，不再在运行时下载。
- 重新部署/更新：Actions → 「自动部署到 CF Worker」→ Run workflow → 分支选 `livetv-deploy`
- 首次部署：2026-11-23，run 45120755600
- 短链接改造：run 45121944506（提交 `177b794`）

## 默认配置与短链接（2026-11-23 加入）

`livetv-worker.js` 的路由里内置了一段默认配置 `DEFAULT_SEG`
（`cc:["cn"]`、`tz:"Asia/Shanghai"`、`name:"直播电视"`、rows 四行、`clean`、`days:3`、`maxStreams:3`）。
请求路径不带配置段、且首段属于 `BARE_ROUTES`（`manifest.json`、`configure`、`catalog`、
`meta`、`stream`、`playlist.m3u`、`playlist.m3u8`、`epg.xml.gz`、`channels.json`、`now.json`）
时自动回落到这段配置，不再返回 `{"error":"bad configuration"}`。

- Stremio 安装链接：`https://livetv.sicenl67.workers.dev/manifest.json`
- 配置页（已预填中国配置）：`https://livetv.sicenl67.workers.dev/configure`
- TiviMate 等 IPTV 播放器：`https://livetv.sicenl67.workers.dev/playlist.m3u`
- 根路径 `/` 仍 302 到 `/configure`
- 带自定义 `<config>` 段的老链接行为不变（`/<config>/manifest.json`、`/<config>/catalog/...` 等）

改默认配置只要改 `DEFAULT_SEG` 那一行常量：它是 `base64url(JSON)`，用
`python3 -c "import base64,json;print(base64.urlsafe_b64encode(json.dumps({...},separators=(',',':')).encode()).decode().rstrip('='))"` 生成。

## 那个部署工作流的副作用

`自动部署到 CF Worker` 会把 domain-check 的 `PASSWORD` secret 一起注入目标 Worker（日志里 `vars: PASSWORD`）。
每次部署后要到 Cloudflare 面板 → livetv → 设置 → 运行时变量和密钥，把 `PASSWORD` 删掉（删除并部署）。
2026-11-23 两次部署后都已清理，现在变量列表为空。

## 网络

注意：`*.workers.dev` 在中国大陆被 DNS 污染 + TCP 阻断（2026-11-23 用 5 个国内探测点实测全部失败）。
国内要直连需给 Worker 绑定自定义域（例如 `tv.lkjhgf.de5.net`），绑定后从国内探测点 3/5 可通。
