# Docker Hub 在中国大陆无法访问：根因分析与解决方案

> 更新时间：2026年6月26日

## 背景

已执行方案1（将 `devlake.docker.scarf.sh/apache/` 替换为 `apache/`），但 `docker compose up -d` 仍然失败。本文分析根因并提供可行解决方案。

---

## 根因分析

### 方案1为何无效？

方案1解决的是 Scarf.sh 遥测代理不稳定的问题。去掉 scarf.sh 前缀后，镜像源变为 Docker Hub 官方地址（`registry-1.docker.io`）。但**真正的根因不在 scarf.sh，而在 Docker Hub 本身在中国大陆已被封锁**。

### Docker Hub 封锁时间线

| 时间 | 事件 |
|------|------|
| 2023年5月 | Docker Hub 在中国大陆访问开始大幅不稳定 |
| 2024年6月 | 国内主要镜像加速服务（含上海交大、网易、阿里云公开加速等）相继因监管要求下线 |
| 2025年至今 | `registry-1.docker.io` 在大陆基本处于封锁状态，直连拉取极不可靠 |

**结论**：当前错误 `EOF` / `failed to do request` 是网络层连接被中断的典型表现，与镜像 URL 前缀无关。`mysql:8` 也可能遇到同样问题（Docker Hub 官方镜像同样走 `registry-1.docker.io`）。

---

## 解决方案（按推荐优先级排序）

### 方案A：配置 registry-mirrors（推荐，成本最低）（亲测有效）

为 Docker 配置国内可用的社区镜像代理，所有 `docker pull` 自动走镜像站，无需修改 `docker-compose.yml`。

**截至2026年6月，经社区实测可用的镜像源：**

```json
{
  "registry-mirrors": [
    "https://docker.1ms.run",
    "https://dockerproxy.net",
    "https://proxy.vvxv.ee",
    "https://dockerproxy.link",
    "https://docker.m.daocloud.io",
    "https://docker.xuanyuan.me"
  ]
}
```

> **注意**：社区镜像站随时可能因监管原因下线，建议配置多个。若某个不可用，Docker 会自动尝试下一个。

**Mac Docker Desktop 配置步骤：**

1. 打开 Docker Desktop
2. 点击右上角齿轮图标 → **Settings**
3. 左侧选择 **Docker Engine**
4. 在 JSON 编辑器中合并以下内容：

```json
{
  "registry-mirrors": [
    "https://docker.1ms.run",
    "https://dockerproxy.net",
    "https://proxy.vvxv.ee",
    "https://dockerproxy.link",
    "https://docker.m.daocloud.io",
    "https://docker.xuanyuan.me"
  ]
}
```

5. 点击 **Apply & Restart**
6. 验证配置生效：

```bash
docker info | grep -A 10 "Registry Mirrors"
```

7. 重新拉取：

```bash
docker compose pull
docker compose up -d
```

---

### 方案B：通过 HTTP/HTTPS 代理（需有可用代理）（亲测有效）

如果你本机或局域网内有 HTTP 代理（如 Clash、V2Ray 等），可配置 Docker 走代理。

**Mac Docker Desktop 配置步骤：**

1. Docker Desktop → Settings → **Resources** → **Proxies**
2. 启用手动代理配置：
   - HTTP Proxy: `http://127.0.0.1:7897`
   - HTTPS Proxy: `http://127.0.0.1:7897`
   - No Proxy: `localhost,127.0.0.1`
3. Apply & Restart

**或在终端临时设置（仅当前 shell 有效）：**

```bash
export HTTP_PROXY=http://127.0.0.1:7897
export HTTPS_PROXY=http://127.0.0.1:7897
docker compose pull
docker compose up -d
```

> 常见代理端口：Clash 默认 `7890`，V2Ray 默认 `10809`。以实际客户端设置为准。

---

### 方案C：在海外云主机预拉取后导入（离线方案，无需代理）

在境外服务器（如新加坡 AWS/阿里云 ECS）拉取镜像，打包后传到本地。

```bash
# 在境外服务器执行
docker pull apache/devlake:v1.0.3-beta13
docker pull apache/devlake-dashboard:v1.0.3-beta13
docker pull apache/devlake-config-ui:v1.0.3-beta13
docker pull mysql:8

docker save apache/devlake:v1.0.3-beta13 | gzip > devlake.tar.gz
docker save apache/devlake-dashboard:v1.0.3-beta13 | gzip > dashboard.tar.gz
docker save apache/devlake-config-ui:v1.0.3-beta13 | gzip > config-ui.tar.gz
docker save mysql:8 | gzip > mysql.tar.gz

# 传回本地后执行
docker load < devlake.tar.gz
docker load < dashboard.tar.gz
docker load < config-ui.tar.gz
docker load < mysql.tar.gz

docker compose up -d
```

---

### 方案D：使用 DaoCloud 托管镜像（替换 image 地址）

DaoCloud 维护了 Docker Hub 官方镜像的同步副本，可直接替换 `docker-compose.yml` 中的 image 地址。

**修改 `docker-compose.yml`：**

```yaml
# 将 mysql:8 改为：
image: docker.m.daocloud.io/library/mysql:8

# 将 apache/devlake:v1.0.3-beta13 改为：
image: docker.m.daocloud.io/apache/devlake:v1.0.3-beta13

# 将 apache/devlake-dashboard:v1.0.3-beta13 改为：
image: docker.m.daocloud.io/apache/devlake-dashboard:v1.0.3-beta13

# 将 apache/devlake-config-ui:v1.0.3-beta13 改为：
image: docker.m.daocloud.io/apache/devlake-config-ui:v1.0.3-beta13
```

> DaoCloud (`docker.m.daocloud.io`) 是目前国内最稳定的免费镜像代理之一，覆盖 Docker Hub、GCR、GHCR 等主流源。

---

## 推荐执行顺序

```
方案A（registry-mirrors）→ 若不行 → 方案D（DaoCloud image 地址）→ 若不行 → 方案B（本地代理）
```

1. **先试方案A**：配置 registry-mirrors，`docker compose pull && docker compose up -d`
2. **若方案A失败**：改用方案D，直接把 image 地址前缀换成 `docker.m.daocloud.io`
3. **若方案D也失败**：说明该镜像站未同步此版本，改用方案B（本地代理）或方案C（离线导入）

---

## 验证是否成功

```bash
# 拉取成功后应看到所有容器 Up 状态
docker compose ps

# 访问 DevLake Config UI
open http://localhost:4000
```

---

## 参考资料

- [dongyubin/DockerHub - 2026年6月国内可用镜像源汇总](https://github.com/dongyubin/DockerHub)
- [2026最新国内Docker镜像源加速列表（6月25日更新）- 知乎](https://zhuanlan.zhihu.com/p/2053271908897010016)
- [DaoCloud 官方镜像加速](https://docker.m.daocloud.io)
- [解决Docker Hub国内无法访问方法汇总（2026最新实测）- 腾讯云](https://cloud.tencent.com/developer/article/2660254)
- [Block of Docker Hub in mainland China - Grokipedia](https://grokipedia.com/page/Block_of_Docker_Hub_in_mainland_China)
- [Best Docker Registry Mirror China 2026 - BetterLink Blog](https://eastondev.com/blog/en/posts/dev/20251217-docker-mirror-guide-2025/)
