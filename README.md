# VibeStudy keepalive

这个公开仓库只做一件事：约每 9 分钟戳一下 VibeStudy 免费后端的 `/api/ping`，
让它别进入 Render 免费档的休眠（15 分钟无请求就睡，醒来约 1 分钟）。
用户实报「点登录第一下特别慢」就是这个休眠。

- 公开仓库用标准 GitHub-hosted Linux runner 是免费的，不占私有仓库的 Actions 额度。
- 仓库里**没有任何密钥**，只有下面那个公开接口地址；可以安全地公开。
- 主程序在私有仓库 `lizehaodanniel/VibeStudy`。这里不放产品代码。

## 怎么做到「真的会一直跑」

GitHub 的 `schedule` 是**尽力而为**。实测：新建的公开仓库里写了每 5 分钟一次，
建仓后 1 小时 15 分本该跑 15 次，实际 **0 次**；而 `workflow_dispatch` 手动
触发每次都正常。把速度押在 schedule 上就是自己骗自己。

所以这里改成**自续链**：

1. `keepalive.yml` 每次先 ping，再用 `GITHUB_TOKEN` 触发自己的下一轮
   `workflow_dispatch`（GitHub 文档明确写了 `GITHUB_TOKEN` 触发的事件默认不会
   新建 workflow run，但 `workflow_dispatch` / `repository_dispatch` 是例外，
   配 `actions: write` 即可），然后安静等 9 分钟。下一轮在队列里排着，
   一轮接一轮，不依赖 GitHub 的定时器。
2. `heartbeat.yml` 每小时（低频 schedule 通常能跑）无条件补排一轮：链活着时，
   `concurrency` 组只会把「之前排队的那轮」换掉，不会越滚越多；链万一被整体
   取消，这一轮就是重新点火。它同时每 7 天写一次 `LAST-ACTIVITY.txt`，防
   「公开仓库 60 天无活动就停掉定时任务」这条规则。

两个 workflow 都在默认分支 `main` 上（`workflow_dispatch` 只认默认分支）。
手动启动链：Actions → `Keep VibeStudy backend awake` → Run workflow，
`chain` 保持 `1`。填 `0` 只 ping 一次，用来排查。

## 再加一层（可选，但更稳）

GitHub 这套是免费的，但终究是「用公开 runner 排队」。想彻底不依赖 GitHub，
用外部免费定时器直接打这个地址就行：

```
https://vibestudy-app.onrender.com/api/ping
```

它不查库、不打外网，只回一个很小的 JSON。三种免费做法（选一个就行）：

1. **UptimeRobot**（最省事）：免费版支持 5 分钟间隔。新建一个 HTTP(s) 监控，
   URL 填上面那个地址，间隔选 5 分钟。它既是保活也是可用性告警。
2. **cron-job.org**：免费定时任务，每 5 分钟 GET 一次上面那个地址。
3. **Cloudflare Workers Cron Triggers**（已有 Cloudflare 账号时用）：
   建一个 Worker，代码是

   ```js
   export default {
     async scheduled(_event, _env, ctx) {
       ctx.waitUntil(fetch('https://vibestudy-app.onrender.com/api/ping'));
     },
     async fetch() { return new Response('ok'); },
   };
   ```

   `wrangler.toml` 里加

   ```toml
   [triggers]
   crons = ["*/5 * * * *"]
   ```

   免费额度对每 5 分钟一次（每天 288 次）绰绰有余。主仓库里
   `tools/keepalive/worker.js` + `wrangler.toml` 就是现成的这份文件。

> 实话：链被 GitHub 整体停掉、或者外部定时器没配的情况下，手机 App 关掉超过
> 15 分钟后第一次点登录仍可能等冷启动。App 开着时前端自己每 9 分钟打一次
> `/api/ping`，这段是有保证的；关掉 App 之后就靠上面这些服务端定时器。
