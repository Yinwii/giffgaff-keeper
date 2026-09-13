# Hermes Agent 通用极简版 · VPS 部署执行手册

> 对应手册章节：⭐ 终极推荐方案（零明文 Key）
> 目标机器：Netcup-2oVPS（2C 2G 60GB · DE-NUE · 私有资产与 AI 智能中枢）

## 一、架构要点（为什么这样设计）

| 设计点 | 说明 |
|---|---|
| 零明文 Key | Compose 内**不出现任何 API_KEY / BASE_URL**，大模型凭据上线后在 9119 面板动态录入，热切换、换 Key 永不停机、备份不泄密 |
| TG 凭据 | Bot Token 放在 `.env`（权限 600），与 compose 分离，比硬编码在 YAML 更干净 |
| 资源硬上限 | cpus 1.5 / memory 1200M，为 2C2G 宿主机上并行的笔记、Filebrowser、Sun-Panel、Dockhand 留足 800MB 底线 |
| 持久化 | `/root/data/docker/hermes/data:/opt/data`，记忆、SQLite、已配置凭据全部落盘 |
| 白名单铁律 | `TELEGRAM_ALLOWED_USERS` 必须填本人数字 ID（@userinfobot 查询），否则公网任何人可盗刷 API 余额 |

## 二、部署步骤（SSH 到 2oVPS 后执行）

### 方式 A：一键脚本（推荐）
```bash
# 把 install.sh 上传或直接在服务器上新建后执行
chmod +x install.sh
sudo ./install.sh
```
脚本会自动：建目录 → 写 compose → 交互式填 Token/ID 生成 .env → 拉镜像启动 → 自检输出日志。

### 方式 B：手动逐步
```bash
sudo mkdir -p /root/data/docker/hermes/data
# 上传 docker-compose.yml 与 .env（由 .env.example 改名并填入真实值）
sudo chmod 600 /root/data/docker/hermes/.env
cd /root/data/docker/hermes
sudo docker compose pull && sudo docker compose up -d
sudo docker logs --tail 20 -f hermes-agent   # 看到面板/长轮询启动日志即成功
```

## 三、上线后配置 AgentRouter（网页热配置，姿势一）

1. 浏览器访问 `http://<2oVPS公网IP>:9119`
2. **Settings → Model Providers → OpenAI Compatible**，填入：
   - Base URL：`https://agentrouter.org/v1`（**必须带 /v1**）
   - API Key：你的 AgentRouter 密钥
   - Model ID：`gpt-5.5/glm-5.2`
3. 保存即热生效，手机 Telegram 直接对话验证。

> 姿势二（挂载 TS 扩展接 Claude Opus 4.6，走 anthropic-messages 协议、baseUrl **严禁加 /v1**）属于进阶玩法，需要时再启用，目录：`/root/data/docker/hermes/data/extensions/agentrouter.ts`。

## 四、安全加固（建议至少做一项）

| 方案 | 做法 | 适用 |
|---|---|---|
| SSH 隧道（最简最安全） | compose 端口改 `127.0.0.1:9119:9119`，本地 `ssh -L 9119:127.0.0.1:9119 root@<IP>` 后访问 localhost:9119 | 只有自己配置时 |
| BitsFlow NPM 反代 | 在 BITSFLOW-DE 的 NPM 上加一条 `hermes.xxx.com → 2oVPS:9119`，开启 Access List 密码/仅 HTTPS | 想随时随地开面板 |
| 防火墙限源 | `ufw allow from <你家出口IP> to any port 9119` | 家宽 IP 相对固定时 |

## 五、日常运维备忘

```bash
docker compose -f /root/data/docker/hermes/docker-compose.yml restart   # 重启
docker compose logs -f hermes-agent                                     # 跟日志
tar czf hermes-backup-$(date +%F).tar.gz -C /root/data/docker/hermes data  # 备份（凭据已入库，注意备份文件本身保管）
```

- 资源红线：容器内存上限 1.2G；宿主机若出现 OOM，优先检查并行服务。
- 换模型/换 Key：一律走 9119 面板热操作，**不改 compose、不重启**。
- 该机是核心续费机，禁止当测试沙盒跑未验证镜像。

## 六、验收清单

- [ ] `docker compose ps` 状态 Up，内存占用 < 1G
- [ ] 9119 面板可登录，Model Providers 已保存 AgentRouter
- [ ] Telegram Bot 对本人 ID 正常回复，对陌生 ID 不响应
- [ ] `/help`、`/model` 内置命令可用
- [ ] 数据目录备份机制就位
