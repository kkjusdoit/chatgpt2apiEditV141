# ChatGPT2API GCP 运维交接与 SOP

最后核验：2026-07-27（Asia/Shanghai）

本文是当前 GCloud 部署的事实来源和操作手册，供后续 AI 或人工运维接管。执行任何变更前，先读完“接管红线”和对应 SOP。

## 1. 接管红线

1. 先只读检查，再决定是否变更；不要一上来重启、重建、刷新全部账号或启动注册任务。
2. 云端 `/opt/chatgpt2api` **不是 Git 仓库**。不能在 VM 上执行 `git pull`、`git reset` 或 `git checkout`。
3. 本地 `/Users/linkunkun/chatgpt2apiEditV141` 是当前部署源码的主要来源，但工作树仍有未提交修复。不要清理、覆盖或回退这些改动。
4. VM 安装的是 `docker-compose` v1.29.2。云端命令应使用 `docker-compose`，不能假设 `docker compose` 可用。
5. SSH 优先使用 IAP。公网 22 端口连接曾被主动断开，但 IAP SSH 已验证正常。
6. 不要把管理员密钥、账号 Token、Workers 管理密码或邮箱完整列表打印到聊天、日志或文档。密钥从 VM 上已有配置读取。
7. 不要覆盖或删除 `/opt/chatgpt2api/.env`、`config.json`、`data/`。
8. 注册功能只允许使用右上角“新注册”，对应 `/api/register/new/*`。不要调用旧入口 `/api/register/start`。
9. 当前账号已经超过目标值，注册机关闭是正常状态；没有明确需要时不要再次启动。
10. 不要对整个 GCP 项目的共享防火墙规则做修改。该项目还运行其他服务，变更前必须逐一审计受影响实例。

## 2. 当前资源清单

| 项目 | 当前值 |
| --- | --- |
| GCP Project ID | `project-074aa92f-a753-47bf-971` |
| VM | `chatgpt2api-warp-1` |
| Zone | `us-central1-a` |
| 机器规格 | `e2-medium`，4 GiB 内存 |
| 系统 | Debian GNU/Linux 12 |
| 磁盘 | 30 GiB `pd-balanced` |
| 公网静态 IP | `136.115.80.70`，地址资源名 `chatgpt2api-warp-ip` |
| 公网入口 | `http://136.115.80.70/` |
| 健康接口 | `http://136.115.80.70/health?format=json` |
| 本地源码 | `/Users/linkunkun/chatgpt2apiEditV141` |
| VM 部署目录 | `/opt/chatgpt2api` |
| Compose 文件 | `/opt/chatgpt2api/docker-compose.warp.yml` |
| 当前应用镜像 | `chatgpt2api:warp` |
| 已核验镜像 ID | `sha256:0696994aa135fe50b0ef89674d05de2c701d9d3d34d2957932d082dbdce33da7` |
| 应用版本 | `1.5.0` |
| 存储后端 | JSON，本地持久化到 `/opt/chatgpt2api/data` |

GCE 已启用 `automaticRestart=true`，宿主机维护策略为 `MIGRATE`，实例不是 Spot/抢占式实例。所有长期运行容器均为 `restart=unless-stopped`。

当前已有磁盘快照：

```text
chatgpt2api-warp-1-us-central1-a-20260727091127-5tl7x4en
```

## 3. 架构与容器

请求链路：

```text
公网 :80
  -> chatgpt2api-warp（FastAPI + Web）
       -> chatgpt2api-privoxy
            -> chatgpt2api-warp-proxy（出站网络）
       -> chatgpt2api-flaresolverr（Cloudflare clearance）
```

容器：

| 容器 | 作用 | 端口 |
| --- | --- | --- |
| `chatgpt2api-warp` | 主应用 | 公网 `80 -> 80` |
| `chatgpt2api-warp-proxy` | WARP SOCKS 出站 | 仅宿主机 `127.0.0.1:40000` |
| `chatgpt2api-privoxy` | HTTP 到 SOCKS 转换 | 仅宿主机 `127.0.0.1:40080` |
| `chatgpt2api-flaresolverr` | clearance 获取 | 仅宿主机 `127.0.0.1:8191` |
| `chatgpt2api-warp-init` | 启动时写入代理运行配置 | 一次性任务，正常状态是 `Exit 0` |

不要把 40000、40080、8191 暴露到公网。

## 4. 当前已部署的重要修复

当前云端不只是上游仓库原始版本，还包含以下尚未提交的本地修复：

- `services/openai_backend_api.py`：将 FlareSolverr Cookie 与匹配的 User-Agent 注入上游请求，修复账号显示有额度但实时请求全部失败的问题。
- `services/register_service.py`：所有目标模式达到目标后都会正常结束并将 `enabled` 设为 `false`。
- 注册配置保存逻辑：未显式配置时，不再写回随机子域字段。
- Docker/WARP/FlareSolverr 启动和代理配置修复。
- 配套测试覆盖代理 clearance、账号环境和注册并发行为。

本地当前必须保留的未提交文件：

```text
Dockerfile
api/app.py
api/support.py
config.json
docker-compose.warp.yml
services/openai_backend_api.py
services/register_service.py
test/test_account_environment.py
test/test_register_concurrency.py
test/test_proxy_clearance_warmup.py
```

本地 Git 基线：

```text
branch: main
HEAD: 9d7a6863b89102495d4b185ed59023b05c96342b
origin: https://github.com/zhangxunvvv-ux/chatgpt2apiEditV141.git
```

在这些修复被正式提交到独立分支之前，不要直接用远端 GitHub 内容覆盖 VM。

## 5. 配置和敏感数据位置

| 路径 | 内容 | 注意事项 |
| --- | --- | --- |
| `/opt/chatgpt2api/.env` | 监听端口、管理员密钥 | 权限必须保持 `600`；不要输出内容 |
| `/opt/chatgpt2api/config.json` | 主配置与运行设置 | 当前权限 `600`；升级时不要盲目覆盖 |
| `/opt/chatgpt2api/data/accounts.json` | 账号池 | 高敏感；禁止打印或提交 Git |
| `/opt/chatgpt2api/data/register.json` | 注册共享配置 | 含 Workers 凭据 |
| `/opt/chatgpt2api/data/new_register.json` | 新注册任务状态 | 含 Workers 凭据 |
| `/opt/chatgpt2api/data/logs.jsonl` | 应用日志 | 可能含邮箱和错误上下文 |
| `/opt/chatgpt2api/data/images/` | 生成图片 | 可通过后台清理，不要直接批量删除 |

`.env` 当前应只包含以下键，不应在交接文档记录实际值：

```text
CHATGPT2API_PORT
CHATGPT2API_AUTH_KEY
```

管理员密钥的读取原则：优先在 VM 内部加载 `.env` 后调用 `127.0.0.1`，避免把密钥带回本地终端输出。

## 6. 新注册配置边界

已确认配置：

| 配置项 | 值 |
| --- | --- |
| 邮件服务 | `cloudflare_temp_email` |
| Workers API | `https://emailbot.kkjusdoit.workers.dev` |
| 邮箱域名 | `mail.fzd-fans.com` |
| subdomain | 必须仍为 `mail.fzd-fans.com` |
| 模式 | `available` |
| 线程 | `2` |
| 正常账号目标 | `15` |
| 达标行为 | 自动结束，`enabled=false` |

禁止重新加入以下字段：

```text
append_random_suffix
random_subdomain_depth
subdomain_levels
```

邮箱唯一性依靠随机邮箱前缀。不要生成自定义子域。

当前行为是“任务启动后补到目标并自动停止”，不是永远驻留的自动唤醒服务。如果未来需要无人值守自动补号，应单独设计监控和授权机制，不要误以为 `enabled=false` 会自行恢复。

## 7. 接管后的首次只读检查 SOP

先确认本地 gcloud 身份和目标，任何命令都显式带 Project 和 Zone：

```bash
gcloud auth list --filter=status:ACTIVE
gcloud config get-value project

gcloud compute instances describe chatgpt2api-warp-1 \
  --project=project-074aa92f-a753-47bf-971 \
  --zone=us-central1-a \
  --format='value(status,networkInterfaces[0].accessConfigs[0].natIP)'
```

检查公网健康：

```bash
curl -fsS --max-time 20 \
  'http://136.115.80.70/health?format=json' | jq
```

健康基线：

- `status` 应为 `ok`
- `healthy` 应为 `true`
- `accounts.active` 应至少为 1；正常目标为 15 或更多
- `proxy_runtime.enabled` 应为 `true`
- `proxy_runtime.has_proxy` 应为 `true`
- `proxy_runtime.clearance_enabled` 应为 `true`
- `proxy_runtime.has_clearance_bundle` 应为 `true`
- `cached_clearance_hosts` 应包含 `chatgpt.com`

使用 IAP 登录：

```bash
gcloud compute ssh chatgpt2api-warp-1 \
  --project=project-074aa92f-a753-47bf-971 \
  --zone=us-central1-a \
  --tunnel-through-iap
```

登录后检查：

```bash
cd /opt/chatgpt2api

sudo docker-compose -f docker-compose.warp.yml ps
sudo docker inspect chatgpt2api-warp \
  --format 'status={{.State.Status}} restart_count={{.RestartCount}} oom={{.State.OOMKilled}} started={{.State.StartedAt}}'

free -h
df -h / /opt
sudo du -sh /opt/chatgpt2api/data
```

正常条件：

- 应用、WARP、Privoxy、FlareSolverr 都是 `Up`
- WARP、Privoxy、FlareSolverr 显示 `healthy`
- `chatgpt2api-warp-init` 显示 `Exit 0`
- `oom=false`
- 磁盘使用率低于 80%
- 可用内存不应长期接近 0

## 8. 不泄漏密钥的 API 验收 SOP

以下命令在 VM 内执行，密钥不会被打印：

```bash
cd /opt/chatgpt2api
set -a
. ./.env
set +a

curl -fsS \
  -H "Authorization: Bearer ${CHATGPT2API_AUTH_KEY}" \
  http://127.0.0.1/v1/models \
  | jq '{count:(.data|length),models:[.data[].id]}'

curl -fsS \
  -H "Authorization: Bearer ${CHATGPT2API_AUTH_KEY}" \
  http://127.0.0.1/api/accounts \
  | jq '{
      count:(.items|length),
      normal:([.items[]|select(.status=="正常")]|length),
      refresh_errors:([.items[]|select(.last_refresh_error != null and .last_refresh_error != "")]|length),
      quota_sum:([.items[].quota // 0]|add)
    }'

curl -fsS \
  -H "Authorization: Bearer ${CHATGPT2API_AUTH_KEY}" \
  http://127.0.0.1/api/register/new \
  | jq '.register | {enabled,mode,threads,target_available,stats}'
```

模型清单必须包含 `gpt-image-2`。不要把 `/api/accounts` 的完整原始响应贴到聊天中。

## 9. 日常巡检 SOP

建议每天或故障时检查一次：

```bash
curl -fsS --max-time 20 \
  'http://136.115.80.70/health?format=json' \
  | jq '{status,healthy,accounts,proxy_runtime}'
```

在 VM 检查容器和近期错误：

```bash
cd /opt/chatgpt2api
sudo docker-compose -f docker-compose.warp.yml ps

sudo docker logs --since 30m chatgpt2api-warp 2>&1 \
  | grep -Ei 'error|exception|traceback|403|502|no available image quota|oom|killed' \
  | tail -n 100
```

不要把一次正常的 `401 Unauthorized` 误判为服务异常；公网扫描器或错误登录会产生 401。重点关注连续 403、502、Traceback、OOM 和所有账号实时刷新失败。

## 10. 安全重启 SOP

只重启主应用：

```bash
cd /opt/chatgpt2api
sudo docker-compose -f docker-compose.warp.yml restart app
```

等待几秒后验收：

```bash
sudo docker-compose -f docker-compose.warp.yml ps
curl -fsS 'http://127.0.0.1/health?format=json' | jq
sudo docker logs --since 5m chatgpt2api-warp 2>&1 | tail -n 100
```

代理链异常时，按顺序重启：

```bash
cd /opt/chatgpt2api
sudo docker-compose -f docker-compose.warp.yml restart warp-proxy
sudo docker-compose -f docker-compose.warp.yml restart privoxy
sudo docker-compose -f docker-compose.warp.yml restart flaresolverr
sudo docker-compose -f docker-compose.warp.yml restart app
```

重启后必须确认三个依赖容器恢复 `healthy`，并重新检查 `has_clearance_bundle`。

## 11. 备份 SOP

### 11.1 应用数据备份

在 VM 上执行：

```bash
backup_stamp=$(date +%Y%m%d-%H%M%S)
sudo mkdir -p /opt/chatgpt2api-backups
sudo tar -czf "/opt/chatgpt2api-backups/chatgpt2api-${backup_stamp}.tgz" \
  -C /opt/chatgpt2api \
  .env config.json data
sudo ls -lh "/opt/chatgpt2api-backups/chatgpt2api-${backup_stamp}.tgz"
```

备份文件包含密钥和账号，权限应收紧：

```bash
sudo chmod 600 "/opt/chatgpt2api-backups/chatgpt2api-${backup_stamp}.tgz"
```

### 11.2 GCE 磁盘快照

重大升级前建议创建快照：

```bash
snapshot_stamp=$(date +%Y%m%d-%H%M%S)
gcloud compute disks snapshot chatgpt2api-warp-1 \
  --project=project-074aa92f-a753-47bf-971 \
  --zone=us-central1-a \
  --snapshot-names="chatgpt2api-warp-1-manual-${snapshot_stamp}"
```

创建完成后必须确认状态为 `READY`。

## 12. 发布升级 SOP

### 12.1 发布前条件

1. 先查看本地 `git status --short` 和 `git diff`，保留用户改动。
2. 不要从 GitHub 覆盖当前热修复。
3. 跑相关测试。
4. 创建数据备份和磁盘快照。
5. 记录当前镜像 ID，给镜像打回滚标签。

当前相关测试命令：

```bash
cd /Users/linkunkun/chatgpt2apiEditV141
uv run python -m unittest \
  test.test_account_environment \
  test.test_register_concurrency \
  test.test_proxy_clearance_warmup
git diff --check
```

已知测试陷阱：不要直接以会让 `test/utils.py` 抢占项目 `utils` 包的方式运行全量 discovery。优先使用点号模块路径或逐模块执行。

### 12.2 给当前镜像打回滚标签

在 VM 上：

```bash
release_stamp=$(date +%Y%m%d-%H%M%S)
sudo docker tag chatgpt2api:warp "chatgpt2api:warp-before-${release_stamp}"
sudo docker image inspect "chatgpt2api:warp-before-${release_stamp}" \
  --format '{{.Id}}'
```

### 12.3 上传代码

因为 VM 不是 Git 仓库，建议从本地打一个**不包含数据和密钥**的代码包：

```bash
cd /Users/linkunkun/chatgpt2apiEditV141
release_stamp=$(date +%Y%m%d-%H%M%S)
release_archive="/tmp/chatgpt2api-code-${release_stamp}.tgz"

tar -czf "${release_archive}" \
  .dockerignore Dockerfile docker-compose.warp.yml \
  main.py pyproject.toml uv.lock VERSION CHANGELOG.md \
  api services utils scripts web

gcloud compute scp "${release_archive}" \
  chatgpt2api-warp-1:/tmp/ \
  --project=project-074aa92f-a753-47bf-971 \
  --zone=us-central1-a \
  --tunnel-through-iap
```

确认压缩包内没有 `.env`、`config.json` 和 `data/`。

### 12.4 云端构建和切换

在 VM 上先备份当前代码，再解压代码包。将时间戳替换为实际值：

```bash
release_stamp=YYYYMMDD-HHMMSS
release_archive="/tmp/chatgpt2api-code-${release_stamp}.tgz"

sudo mkdir -p "/opt/chatgpt2api-backups/source-${release_stamp}"
sudo cp -a \
  /opt/chatgpt2api/Dockerfile \
  /opt/chatgpt2api/docker-compose.warp.yml \
  /opt/chatgpt2api/main.py \
  /opt/chatgpt2api/pyproject.toml \
  /opt/chatgpt2api/uv.lock \
  /opt/chatgpt2api/api \
  /opt/chatgpt2api/services \
  /opt/chatgpt2api/utils \
  /opt/chatgpt2api/scripts \
  /opt/chatgpt2api/web \
  "/opt/chatgpt2api-backups/source-${release_stamp}/"

sudo tar -xzf "${release_archive}" -C /opt/chatgpt2api
sudo chown -R linkunkun:linkunkun \
  /opt/chatgpt2api/api \
  /opt/chatgpt2api/services \
  /opt/chatgpt2api/utils \
  /opt/chatgpt2api/scripts \
  /opt/chatgpt2api/web

cd /opt/chatgpt2api
sudo docker-compose -f docker-compose.warp.yml up -d --build
```

不要使用 `docker-compose down -v`，它会扩大停机和数据风险。

### 12.5 发布验收

```bash
cd /opt/chatgpt2api
sudo docker-compose -f docker-compose.warp.yml ps
sudo docker inspect chatgpt2api-warp \
  --format 'image={{.Image}} restart_count={{.RestartCount}} oom={{.State.OOMKilled}}'
curl -fsS 'http://127.0.0.1/health?format=json' | jq
sudo docker logs --since 10m chatgpt2api-warp 2>&1 | tail -n 200
```

然后执行第 8 节的模型、账号和注册配置验收。图片编辑属于会消耗额度的写操作，只有在必要时做 1 次最小化测试。

## 13. 回滚 SOP

代码或镜像升级失败时，优先回滚应用镜像，不要碰 `data/`：

```bash
sudo docker image ls 'chatgpt2api' --no-trunc
```

确认目标回滚标签后：

```bash
rollback_tag='chatgpt2api:warp-before-YYYYMMDD-HHMMSS'
sudo docker tag "${rollback_tag}" chatgpt2api:warp

cd /opt/chatgpt2api
sudo docker-compose -f docker-compose.warp.yml up -d --no-build --force-recreate app
```

随后按第 12.5 节验收。

如果还需要恢复源码，先确认备份目录内容，再将明确的文件复制回 `/opt/chatgpt2api`。数据恢复会覆盖账号、日志和配置，属于高风险操作，必须得到用户明确授权后再执行。

## 14. 常见故障 SOP

### 14.1 `no available image quota`

不要只看后台展示的 quota。按以下顺序判断：

1. 查 `/health?format=json` 中 active、total_quota、代理和 clearance 状态。
2. 查鉴权后的 `/v1/models` 是否包含 `gpt-image-2`。
3. 汇总 `/api/accounts` 的 `last_refresh_error`，不要输出完整 Token。
4. 查日志是否有连续 403、连接重置或 clearance 失败。
5. 查 WARP、Privoxy、FlareSolverr 是否 healthy。
6. 只有确认真实正常账号不足时，才考虑补号；不要先启动注册机掩盖代理故障。

已知根因历史：UI 额度正常，但实时账号验证请求未携带匹配的 Cloudflare Cookie/User-Agent，导致所有候选账号被淘汰。当前修复在 `services/openai_backend_api.py`，升级时不能丢失。

### 14.2 模型列表为 0 或一直加载

这通常与实时上游请求失败同源。先检查 clearance 和代理链，不要先改前端。

```bash
sudo docker-compose -f /opt/chatgpt2api/docker-compose.warp.yml ps
sudo docker logs --since 20m chatgpt2api-warp 2>&1 \
  | grep -Ei '403|clearance|proxy|connection reset|model' \
  | tail -n 100
```

### 14.3 公网 502 或页面打不开

1. 用 gcloud 检查实例是否 `RUNNING`。
2. IAP 登录后检查 Docker 服务和容器。
3. 检查内存、Swap、磁盘和 OOM。
4. 先只重启 `app`。
5. 如果刚发布过，按镜像标签回滚。

```bash
systemctl is-active docker
sudo docker ps -a
free -h
df -h / /opt
sudo docker inspect chatgpt2api-warp \
  --format 'status={{.State.Status}} oom={{.State.OOMKilled}} exit={{.State.ExitCode}} error={{.State.Error}}'
```

### 14.4 WARP/Privoxy/FlareSolverr 不健康

先看对应容器日志，再按第 10 节顺序重启代理链。不要删除 Docker 网络、卷或 WARP 状态目录。

### 14.5 磁盘增长

```bash
sudo du -h -d 2 /opt/chatgpt2api/data | sort -h | tail -n 30
sudo docker system df
```

图片和应用日志优先通过后台已有清理功能处理。不要直接运行 `rm -rf data`，也不要未经确认执行 `docker system prune -a`。

### 14.6 公网 SSH 失败

这是已知现象，直接改用：

```bash
gcloud compute ssh chatgpt2api-warp-1 \
  --project=project-074aa92f-a753-47bf-971 \
  --zone=us-central1-a \
  --tunnel-through-iap
```

不要为了修复单台 VM 的 SSH，直接删除或改写整个项目的共享防火墙规则。

## 15. 注册任务 SOP

启动前必须满足：用户明确要求、正常账号数低于目标、邮件配置核验无误、没有正在运行的注册任务。

只读检查：

```bash
cd /opt/chatgpt2api
set -a
. ./.env
set +a

curl -fsS \
  -H "Authorization: Bearer ${CHATGPT2API_AUTH_KEY}" \
  http://127.0.0.1/api/register/new \
  | jq '.register | {enabled,mode,threads,target_available,stats}'
```

经用户授权后，只能启动新注册：

```bash
curl -fsS -X POST \
  -H "Authorization: Bearer ${CHATGPT2API_AUTH_KEY}" \
  http://127.0.0.1/api/register/new/start \
  | jq '.register | {enabled,mode,threads,target_available,stats}'
```

停止：

```bash
curl -fsS -X POST \
  -H "Authorization: Bearer ${CHATGPT2API_AUTH_KEY}" \
  http://127.0.0.1/api/register/new/stop \
  | jq '.register | {enabled,stats}'
```

严禁误调用：

```text
POST /api/register/start
```

## 16. 安全改进待办

以下不是当前故障，但建议后续安排：

1. 给公网入口绑定域名并启用 HTTPS。当前是明文 HTTP，不适合在不可信网络上传输管理员密钥。
2. 将管理员密钥和 Workers 凭据迁移到 GCP Secret Manager，减少明文配置散落。
3. 为 `/health?format=json` 配置 GCP Uptime Check 和告警。
4. 建立定期快照计划并配置保留周期；不要无限积累快照。
5. 将当前本地热修复提交到独立分支并推送到受控远端，消除“云端已部署但 Git 未记录”的风险。
6. 审计项目级 `default-allow-ssh` 等宽泛防火墙规则；确认不会影响其他实例后再收紧。

## 17. 当前已知良好基线

2026-07-27 最终验收结果：

- 公网首页：HTTP 200
- 健康状态：`ok`
- 正常账号：16/16
- 图片额度合计：216
- 账号刷新错误：0
- 模型数：11，包含 `gpt-image-2`
- clearance：已启用且缓存 `chatgpt.com`
- 应用容器：运行中，`restart_count=0`，`oom=false`
- WARP、Privoxy、FlareSolverr：运行且 healthy
- 内存可用约 2.8 GiB
- Swap：2 GiB
- 磁盘使用约 38%，剩余约 18 GiB
- 本地相关测试：24 个通过
- 已完成一次“延朋小店”图片编辑实测，未再出现 `no available image quota`

账号数和额度会随使用变化，不应把 16 和 216 当成永久常量；判断健康时看状态、错误率和是否高于运行阈值。

## 18. 给下一位 AI 的复制提示词

```text
你正在接管 chatgpt2api 的 GCP 运维。

先完整阅读：
/Users/linkunkun/chatgpt2apiEditV141/docs/GCP_OPERATIONS_RUNBOOK.md

目标资源：
- Project: project-074aa92f-a753-47bf-971
- VM: chatgpt2api-warp-1
- Zone: us-central1-a
- VM 目录: /opt/chatgpt2api
- 本地源码: /Users/linkunkun/chatgpt2apiEditV141

规则：
1. 先做只读健康检查并汇报证据，再执行变更。
2. SSH 使用 --tunnel-through-iap。
3. 云端使用 docker-compose，不是 docker compose。
4. VM 目录不是 Git 仓库；本地有未提交且已部署的关键修复，禁止 reset、checkout 或用 GitHub 覆盖。
5. 保护 .env、config.json、data/，不要输出任何密钥、Token 或完整账号列表。
6. 注册任务只允许使用 /api/register/new/*；除非我明确要求，否则不要启动注册。
7. 任何发布前先备份数据、记录镜像 ID并准备回滚；发布后验证 health、models、accounts、clearance 和容器状态。

如果现状与文档不一致，先说明差异，不要自行假设或扩大操作范围。
```
