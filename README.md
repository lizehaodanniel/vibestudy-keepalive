# VibeStudy keepalive

这个公开仓库只做一件事：每 5 分钟戳一下 VibeStudy 免费后端的 `/api/ping`，
让它别进入 Render 免费档的休眠（15 分钟无请求就睡，醒来约 1 分钟）。

- 公开仓库用标准 GitHub-hosted Linux runner 是免费的，不占私有仓库的 Actions 额度。
- 仓库里**没有任何密钥**，只有上面那个公开接口地址；可以安全地公开。
- 主程序在私有仓库 `lizehaodanniel/VibeStudy`。这里不放产品代码。
