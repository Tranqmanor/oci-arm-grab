# 关机也能抢：GitHub Actions 版轮询器

原理：GitHub 的免费 runner 按 cron（默认 `*/15`）跑这个 workflow，每次 job 内部再用
`oci-arm-catcher.sh` 连续重试约 13 分钟（`RETRY_INTERVAL` 默认 120 秒）。
你的电脑关不关机都不影响，抢到了还能用 Server酱 推微信。

> ⚠️ GitHub 的 cron **不保证精确**，实测 `*/15` 的真实间隔可能是 2~5 小时。
> 想真正提高命中率，请让本机 `.\arm.ps1 grab-bg` 和 GitHub 通道**同时跑**。

## 1. 全自动（推荐）：一条命令建私有仓库 + push + 配 Secrets/Variables

在上层工作区目录跑（脚本会直接读你本机 `~/.oci/config`、`oci-arm-catcher\.env`
和 `~/.ssh/id_ed25519.pub`，不用手抄任何东西）：

```powershell
cd f:\projects\VS_code\Oracle
.\setup-grab-repo.ps1 -DryRun     # 先看要上传哪些值（私钥只显示字节数）
.\setup-grab-repo.ps1             # 交互粘贴 PAT，自动建库/推送/上传/触发首轮
```

需要的 PAT（Personal Access Token）：
- **Fine-grained**（推荐）：Expiration 自己定；Resource owner 选你自己的账号；
  Repository permissions 至少 `Contents: Read and write`、`Actions: Read and write`、
  `Administration: Read and write`；Repository access 选 `All repositories`
  （因为要新建仓库）。
- **Classic**：勾选 `repo` + `workflow` 两个 scope 即可。

> ⚠ **实测坑（2026-10-21）**：fine-grained token 即使把 Actions 设成 Read and write，
> `git push` 里包含 `.github/workflows/*` 时仍可能被 pre-receive hook 拦下：
> `refusing to allow a Personal Access Token to create or update workflow ... without 'workflow' scope`
> 三种解法，任选：
> 1. 换 **Classic token**（勾 `repo` + `workflow`）→ 直接 `.\setup-grab-repo.ps1`，一次到位；
> 2. `.\setup-grab-repo.ps1 -ApiPush` → 其它文件用 Contents API 上传，workflow 文件你在
>    网页 Add file → Create new file 手动粘贴（网页提交不受 token scope 限制）；
> 3. 在网页上把整个仓库手建（含 workflow 文件），然后 `.\setup-grab-repo.ps1 -SkipPush`
>    只上传 Secrets/Variables 并触发运行。
> 另外：给已存在的 token 改权限，GitHub 不一定让旧 token 长出新能力，必要时重新生成一个。

> Token 只出现在命令行（通过 `http.extraheader` 传给 git），**不会**写进
> `.git/config`；Secrets 用 libsodium `crypto_box_seal` 加密后才上传。
> 想省交互可以把 token 先放文件：`.\setup-grab-repo.ps1 -TokenFile $env:TEMP\gh.txt`
> （用完记得删掉那个文件）。

脚本跑完会打印仓库/Actions 页面链接，并自动 `workflow_dispatch` 触发一次验证。
核对页面：`https://github.com/<你>/<仓库名>/settings/secrets/actions`

> **本次实际产出的仓库是 `Tranqmanor/oci-arm-grab`（公开）**，用命令：
> `.\setup-grab-repo.ps1 -Repo oci-arm-grab -Public -VarsToSecrets -DeleteVars`
> 本地目录名 `oci-grab-cloud\` 只是脚本的「载荷目录」，与线上仓库名无关。

## 1.5 公开仓库模式（推荐：不占 Actions 分钟数）

私有仓库的 GitHub-hosted runner 每月只有 2,000 分钟，`*/15` + 每轮 ~11 分钟
两天就会烧光；**公开仓库的 Linux runner 不计分钟数**，可以真正全天候轮询。

```powershell
cd f:\projects\VS_code\Oracle
.\setup-grab-repo.ps1 -Repo oci-arm-grab -Public -VarsToSecrets -DeleteVars
```
这条会：把仓库改成 public → 把 11 个 Variables 的值写进 Secrets → 删掉 Variables
→ push → 触发一轮。workflow 里取的是 `${{ vars.X || secrets.X }}`，所以两种放法都兼容。

> 刚建仓就改可见性，GitHub 常返回
> `422 Failed to update visibility. A previous repository operation is still in progress.`
> 脚本已内置退避重试（最多约 100 秒）；仍失败就等十几秒重跑
> `.\setup-grab-repo.ps1 -Repo oci-arm-grab -Public -VarsToSecrets`，
> 或在网页 Settings → Danger Zone → Change visibility 手动改公开。
> 这一步失败**不会**中断后面的 push / Secrets 上传。

公开前必须确认的两件事：
1. **仓库文件里不要写死 OCID**（本仓库的 README 已改成占位符；如果你自己加过
   真实 OCID，记得清掉 —— git 历史里也留着，必要时重建仓库）；
2. API 私钥只存在于 Secrets 里。公开仓库对 fork 的 PR 默认**不会**暴露 Secrets，
   但请别在仓库里放任何 `.pem`（`.gitignore` 已经拦了）。

> 用完之后想再关回去：`Settings → Danger Zone → Change visibility`。

## 2. 手动路线（不想用脚本时）

建一个**私有**仓库（不要把仓库公开，OCIDs 虽不算秘密但没必要公开）：

```powershell
cd f:\projects\VS_code\Oracle\oci-grab-cloud
git init
git add .
git commit -m "oci arm grabber on GH Actions"
# 在 GitHub 上建 private repo 后：
git branch -M main
git remote add origin https://github.com/<你的用户名>/oci-grab-cloud.git
git push -u origin main
```

> cron 只在**默认分支**上生效；仓库如果 60 天没活动，GitHub 会自动停用定时任务
> （Settings → Actions 里可以重新打开）。

## 3. 配置 Secrets（Settings → Secrets and variables → Actions → New repository secret）

> 用 `setup-grab-repo.ps1` 的话这一节和第 4 节会自动完成，下面只作为核对清单。

| Secret | 内容 | 从哪拿 |
|---|---|---|
| `OCI_TENANCY_OCID` | `ocid1.tenancy.oc1..aaaa...` | 控制台 Profile → Tenancy → Tenancy OCID |
| `OCI_USER_OCID` | `ocid1.user.oc1..aaaa...` | 控制台 Profile → My Profile → OCI 用户 OCID |
| `OCI_KEY_FINGERPRINT` | 形如 `xx:xx:xx:...` | 本机 `Select-String fingerprint $HOME\.oci\config`（和上传到控制台的那把公钥配对） |
| `OCI_API_KEY_PEM` | **私钥全文**（`-----BEGIN PRIVATE KEY-----` 到 `-----END...`） | 本机 `~/.oci/config` 里 `key_file=` 指向的那个 `.pem` 文件 |
| `ARM_SSH_PUB_KEY` | 要注入实例的 **公钥** 一行 | 本机 `Get-Content $HOME\.ssh\id_ed25519.pub` |
| `SERVERCHAN_KEY`（可选） | Server酱 的 SendKey | https://sct.ftqq.com/ ，抢到后推微信 |

## 4. 配置 Variables（同一页面 → Actions variables → New repository variable）

> ⚠ **仓库如果是公开的，别把 OCID 放在 Variables 里**（Variables 在仓库设置和 Actions
> 日志里可见）。公开仓库请一律放 Secrets：
> `..\setup-grab-repo.ps1 -Public -VarsToSecrets -DeleteVars`
> 会把下面这些值搬进 Secrets 并删掉 Variables。workflow 里取的是
> `${{ vars.X || secrets.X }}`，两种放法都能读到，不用改代码。

| Variable | Value（换成你自己的） |
|---|---|
| `OCI_REGION` | home region，如 `ap-singapore-1` |
| `COMPARTMENT_ID` | `ocid1.tenancy.oc1..` 你的 tenancy OCID |
| `AVAILABILITY_DOMAIN` | 如 `XXXX:AP-SINGAPORE-1-AD-1` |
| `SUBNET_ID` | `ocid1.subnet.oc1.<region>....`（能分公网 IP 的子网） |
| `IMAGE_ID` | `ocid1.image.oc1.<region>....`（**aarch64/ARM** 镜像） |
| `OCPUS` | `2` |
| `MEMORY_GB` | `12` |
| `DISPLAY_NAME` | `arm-free-1` |
| `BOOT_VOLUME_SIZE_GB` | `150` |
| `BOOT_VOLUME_VPUS_PER_GB` | `10`（Balanced；>10 是付费档） |
| `RETRY_INTERVAL` | `120`（别设 60，容易撞 429 TooManyRequests） |

变量名可以从本机一键导出对照（不打印私钥）：

```powershell
cd f:\projects\VS_code\Oracle
Get-Content .\oci-arm-catcher\.env -Encoding UTF8 | Select-String -Pattern '^(COMPARTMENT_ID|AVAILABILITY_DOMAIN|SUBNET_ID|IMAGE_ID|OCPUS|MEMORY_GB|DISPLAY_NAME|BOOT_VOLUME|RETRY_INTERVAL)='
Select-String -Path $HOME\.oci\config -Pattern 'region|tenancy|user|fingerprint'
```

## 5. 先手动跑一次验证

Actions → *oci-arm-grab* → **Run workflow**。看日志：

- `写 ~/.oci/config 和公钥` 步骤应打出可用域 JSON（证明认证 OK）；
- `抢 ARM` 步骤出现 `InternalError: Out of host capacity.` = 一切正常，只是还没容量；
- 真的抢到时 job 会成功结束，并自动把引导卷扩到 150 GB、设成 10 VPUs/GB，
  再推一条 Server酱 微信通知。

## 6. 注意事项 / 限制

- **认证私钥放 GitHub Secrets**：等于把钥匙交给 GitHub（公开仓库也一样，fork 的 PR
  默认拿不到 Secrets）。抢完可以把这个 API key 在控制台 API keys 里 **删除/换新**（一分钟的事），
  或者用一条只授予 `use-compute-instance` 相关权限的独立 IAM 用户来生成 key。
- cron 实际触发可能延迟几分钟到**几小时**（GitHub 调度器高峰期会合并/延迟 schedule，
  官方不保证精确性）。本仓库实测 `*/15` 的真实间隔在 **2~5 小时**，
  所以**不能只依赖 GitHub 一条通道**——务必让本机 `.\arm.ps1 grab-bg` 也一起跑。
- 想更密：把 `cron` 改成 `'*/5 * * * *'`（5 分钟一轮），job 内部连续跑 13 分钟，
  并发组 `arm-grab` 会防止重叠。**注意**：调密 cron 并不能让 GitHub 更快触发，
  真正的提升来自本地轮询 + 拉长单轮窗口。
- **单轮窗口**：`timeout 780s` ÷ `RETRY_INTERVAL 120s` ≈ 每轮 6~7 次 launch 尝试
  （`timeout-minutes: 20` 留够了扩卷/通知的余量）。
- **分钟数**：私有仓库每月 2000 分钟，15 分钟一轮 ≈ 每月 ~1000 分钟，够用但占一半；
  **公开仓库的 GitHub-hosted Linux runner 不计分钟数**（本仓库当前就是公开模式），
  所以拉长 `timeout` 不增加费用。想省：把单次 `timeout 780s` 调小（比如 240s）。
- GitHub runner 出口 IP 是微软 Azure 段的（美国），API 调用没问题；真正建实例也是
  在你自己 tenancy 里，不受影响。
- **本机也在跑同一条轮询**（`.\arm.ps1 grab-bg`）时，两个轮询器会一起打
  `LaunchInstance` 并撞 `TooManyRequests`。**不是「偶尔、不影响」**——实测有过一整轮
  7 次尝试全被限流废掉（见下面 #45/#46 对照数据）。429 属于 retryable，轮询不会断，
  但被限的那条通道当轮等于白跑。
- **自检小坑**：`$GITHUB_PATH` 只对后续步骤生效，当前步骤里 `oci` 不在 PATH，
  必须用绝对路径 `"$HOME/oci-venv/bin/oci"`；`bash -e` 下 `timeout` 返回 143
  会被当成失败，脚本里已用 `set +e` + 自己判退出码解决。
- **两条通道并发时会被 Oracle 限流（已知行为，不是 bug）**。两个轮询器同时打
  `LaunchInstance` 会触发 `TooManyRequests: Too many requests for the user`，
  实测代价是其中一条通道的尝试整段作废：

  | 轮次 | 配置 | GitHub 有效探测 | 本地有效探测 |
  |---|---|---|---|
  | #45 | 无错峰偏移 | **0/7 全被 429** | 6/6 正常 |
  | #46 | 随机偏移 31s | **7/7 正常** | 1/6（5 次 429） |

  429 与两通道请求落在同一分钟内严格相关，#33 恢复正常、#34 又撞，说明这靠相位
  避不干净。现在保留错峰、两条都维持 120s，**约一半尝试被 429 吃掉是预期内的**，
  抢到实例靠的是总尝试次数够多，不是单次成功率。429 在脚本里属于 retryable，
  会照常等 `RETRY_INTERVAL` 再打，不会中断轮询。
  想彻底消除竞争只能降频或只留一条通道（见上一条 2026-10-28 的权衡记录）。
- **抢机脚本已内联进本仓库**（`oci-arm-catcher.sh`，别再改成临时 clone 上游）。
  2026-10-28 踩过坑：上游 `alexpua/oci-arm-catcher` 改名为 `deeplabua/oci-arm-catcher`
  并用 Rust 重写，根目录的 `oci-arm-catcher.sh` 被挪进 `legacy/`，于是 clone 后
  `chmod +x oci-arm-catcher.sh` 直接 `No such file or directory`，整轮 job 变红。
  内联版本来自上游 `legacy/oci-arm-catcher.sh`（blob sha `3df6956`，14223 字节，LF），
  退出码语义不变（0=抢到 / 124·143=超时继续等 / 其它=真错误），必填键集合不变，
  所以 `.env` 无需改动。仓库里的 `.gitattributes` 强制 `*.sh` 以 LF 入库——
  Windows 上 `core.autocrlf` 会把 `.sh` 转成 CRLF，那样 bash 会因行尾 `\r` 报错。

## 7. 不想用 GitHub 的另外两条路

1. **另一台常开的机器**（家里的 NAS / 树莓派 / 安卓 Termux / 别人的 VPS）跑 Bash 版：
   ```bash
   git clone https://github.com/alexpua/oci-arm-catcher.git && cd oci-arm-catcher
   pip install oci-cli && oci setup config
   cp .env.example .env && vi .env      # 值照抄本机 oci-arm-catcher\.env
   nohup ./oci-arm-catcher.sh > catcher.log 2>&1 &
   ```
   这条是 24/7 秒级轮询，比 cron 更密，且不依赖第三方。
2. **先建一台免费的 AMD 小机器**（`VM.Standard.E2.1.Micro`，ARM 额度之外的另一份免费
   额度），在上面挂抢机脚本。控制台直接创建，如果也提示 out of capacity，就把这台
   小机器的创建也交给上面这个 workflow（把 `--shape` 换成 micro 即可）。
