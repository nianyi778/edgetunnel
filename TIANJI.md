# 天机对这个 fork 的改动

这是 cmliu/edgetunnel 的 fork，只用来部署 `tj-edge` 这一个 worker。
**上游代码不动，我们的改动全在 `tianji` 分支**：`main` 保持与上游一致，同步上游后
`git merge main` 会把冲突正好卡在我们改过的地方，而不是静默覆盖。

看改了什么：`git diff main -- _worker.js`

## 三处改动，各自的理由

### 1. 摘掉第三方面板

原版把 `/login` `/admin` 等四个路径 `fetch` 到 `edt-pages.github.io`。
那段 JS 由别人控制，却跑在我们的域名下、拿得到我们客户的请求。
四处全改成本地 404，`Pages静态页面 = null`。配置一律走环境变量和 KV。

⚠️ **上游一更新就会把它带回来。** 这是这个文件存在的主要原因。

### 2. UUID 白名单（原版只认一个 UUID）

原版 `env.UUID` 是单个值，一个 worker 只能服务一个人。天机每个客户用自己的 uuid
（`accounts.uuid`，见天机 docs/01 §2.1：uuid 是代理凭据）。不改这里，客户端会显示
节点可用（TLS 握手成功）但连上不通 —— 最难查的一类静默失败。

名单从天机主站的 `/api/edge-uuids` 拉（bearer token），存 KV `users.txt`，
内存缓存 60 秒。

⚠️ **不由 cron 独占。** 2026-08-26 部署当天 `*/5 * * * *` 注册成功、线上 `scheduled`
也在，但 20 分钟一次都没触发（疑似新账号首次注册 cron 的延迟）。所以 KV 读到 0 条时
当场直连回源拉一次（60 秒冷却）。cron 停了只是变慢，不会让整个节点对所有客户失效。

⚠️ **拉到 0 条时拒绝写 KV。** 上游出错返回空 body 就清空白名单 = 把所有在线客户踢下线。
真要清空必须人工做。

### 3. 关掉订阅转换

`SUBAPI` / `SUBCONFIG` 置空。原版指向第三方服务器，开了等于把客户的订阅地址
（含 uuid）发给别人。天机的订阅一律由自己的 `render-sub.ts` 生成。

⚠️ `wrangler.toml` 的 `[vars]` **管不住这个** —— KV 里的 `config.json` 会整份替换默认值
（`config_JSON = JSON.parse(configJSON)`），要改得读-改-写 KV。

## 需要的 secret

`UUID` · `PROXYIP` · `UUID_SYNC_URL` · `UUID_SYNC_TOKEN`
（`UUID_SYNC_TOKEN` 与天机主站的 `EDGE_SYNC_TOKEN` 是同一个值）

## 部署

跑在**独立 CF 账号** `d3c4c106…`，与天机主账号完全分开：拿 Workers 做通用代理是 CF
服务条款的灰色地带，一旦被判违规是整个账号一起封，而主账号里有 Pages 主站、D1、KV
和老系统 panel —— 订阅地址是"永不改变"的对外承诺，断了补不回来。
