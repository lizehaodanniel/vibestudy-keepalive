# VibeStudy keepalive

这个公开仓库只做一件事：每 5 分钟戳一下 VibeStudy 免费后端的 `/api/ping`，
让它别进入 Render 免费档的休眠（15 分钟无请求就睡，醒来约 1 分钟）。

- 公开仓库用标准 GitHub-hosted Linux runner 是免费的，不占私有仓库的 Actions 额度。
- 仓库里**没有任何密钥**，只有上面那个公开接口地址；可以安全地公开。
- 主程序在私有仓库 `lizehaodanniel/VibeStudy`。这里不放产品代码。

## ⚠️ 实话：光靠 GitHub 定时任务不够

GitHub 的 `schedule` 是**尽力而为**：官方文档和社区案例都确认，新建仓库里
每 5 分钟一次的高频任务会被大量丢弃，实际可能几小时才跑一次，甚至一整天
不触发；而手动 `workflow_dispatch` 一直正常。也就是说这里的 workflow
只是「能跑就赚到」的一层，不能当成「服务保证不睡」的承诺。

要真正做到每 5 分钟必有一次入站请求，**用外部免费定时器直接打这个地址**：

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

   免费额度对每 5 分钟一次（每天 288 次）绰绰有余。

另外这个仓库里有个每周写一次 `LAST-ACTIVITY.txt` 的 heartbeat：公开仓库
连续 60 天没有任何活动，GitHub 会自动把定时任务停掉，这个心跳就是防那条规则。
