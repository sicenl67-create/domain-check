# livetv Stremio 直播插件 · 部署说明

这个分支只用来触发部署，仓库里的其它文件对部署没有影响。

- Worker 名称：`livetv`，线上地址 `https://livetv.sicenl67.workers.dev`
- 代码来源：`jyotiprakash101/horizon-home-kit` 的 `livetv-addon/worker-bundle.js`（在 Actions 运行时下载）
- 重新部署/更新：Actions → 「自动部署到 CF Worker」→ Run workflow → 分支选 `livetv-deploy`
- 首次部署：2026-11-23，run 45120755600

注意：`*.workers.dev` 在中国大陆被 DNS 污染 + TCP 阻断（2026-11-23 用 5 个国内探测点实测全部失败）。
国内要直连需给 Worker 绑定自定义域（例如 `tv.lkjhgf.de5.net`），绑定后从国内探测点 3/5 可通。
