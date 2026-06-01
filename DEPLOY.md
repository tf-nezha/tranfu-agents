# 部署文档(简洁版)

> 管理员看这份(部署服务端)。团队成员怎么接入自己的 agent → 见 `USAGE.md`。
> 最快路径:`cp deploy/.env.example deploy/.env` 填好密钥,然后 `docker compose up -d`。

只有**一个**需要常开的进程:

- **server** —— 收事件 + 算活跃时长 + 提供看板

看板是纯静态页,由 server 直接提供,不用单独部署。
> 部署就是单容器(不依赖任何外部服务)。

---

## 1. 本地试跑(1 分钟)

```bash
pip install -r server/requirements.txt
TF_KEY=改成你的密钥 \
  python -m uvicorn server.app:app --host 0.0.0.0 --port 8787
# 浏览器打开 http://localhost:8787
```

冒烟测试(另开一个终端):

```bash
export TF_SERVER=http://localhost:8787 TF_KEY=改成你的密钥 TF_OPERATOR=test
source shims/tf_client.sh
tf_emit running --task "联通测试" --step "hello board"
```

---

## 2. Docker 一键(推荐)

```bash
cd deploy
cp .env.example .env        # 填 TF_KEY
docker compose up -d
```

- `8787` = 看板 + 事件接口(单容器)
- 数据落在 docker volume `tf-data`,重启不丢

---

## 3. 部署到线上(任选其一)

> 目标:一个 24h 常开、有公网地址的小服务。规格很低,512MB 内存足够。

**Fly.io**
```bash
fly launch --dockerfile server/Dockerfile --now
fly secrets set TF_KEY=改成你的密钥
fly volumes create tf_data --size 1     # 持久化 SQLite
```

**Railway / Render**
- 新建服务,指向本仓库,Dockerfile 路径填 `server/Dockerfile`
- 环境变量:`TF_KEY`;挂一个 volume 到 `/data`
- 暴露端口 `8787`

**自己的 VPS(systemd 常驻)**
```bash
pip install -r server/requirements.txt
sudo tee /etc/systemd/system/tranfu.service >/dev/null <<EOF
[Service]
Environment=TF_KEY=改成你的密钥
Environment=TF_DB=/var/lib/tranfu/tf.db
WorkingDirectory=$(pwd)
ExecStart=$(which python) -m uvicorn server.app:app --host 0.0.0.0 --port 8787
Restart=always
[Install]
WantedBy=multi-user.target
EOF
sudo mkdir -p /var/lib/tranfu
sudo systemctl enable --now tranfu
```
前面建议挂一层 Caddy/Nginx 上 HTTPS。

---

## 4. 接入各类 agent

团队成员怎么接(本地 CLI 用 `tf-run`、Claude Code 用钩子、云端用 `--coarse`)
见 `USAGE.md`。

---

## 5. 安全清单(务必看)

- **`TF_KEY`(接入密钥)一定要设**,否则任何人都能往你的板子塞数据。
- **打开内容采集(`TF_CAPTURE_CONTENT=1`)后**,prompt 和代码会进库并显示在看板上——
  此时务必把看板放到 **VPN / 内网 / SSO** 后面,不要裸奔公网。
- 活跃时长按 **UTC 日/周** 统计;跨天的会话会按当天边界自动拆分。
