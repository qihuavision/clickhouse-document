# Docker 镜像加速配置指南（阿里云）

> 适用场景：国内服务器 `docker pull` 拉取 Docker Hub 镜像缓慢或超时。
> 原理：配置账号专属加速器地址，由阿里云代理拉取 Docker Hub 官方镜像。

---

## 1. 获取加速器地址

1. 登录阿里云控制台，进入**容器镜像服务 ACR**：
   - 直达地址：<https://cr.console.aliyun.com>
2. 左侧导航栏选择 **镜像工具 → 镜像加速器**
3. 页面显示本账号专属加速地址，格式：

```
https://<你的专属编码>.mirror.aliyuncs.com
```

> **注意**
> - 加速地址与阿里云账号绑定，每个账号不同，需登录后查看自己的。
> - 地址仅供本账号使用，不要外传，否则可能触发限流。
> - 2025 年 ACR 控制台有改版，若左侧栏找不到"镜像工具"，用上面直达地址进入。
>   参考：[官方镜像加速 - 阿里云帮助文档](https://help.aliyun.com/zh/acr/user-guide/accelerate-the-pulls-of-docker-official-images)

---

## 2. Linux（CentOS / Ubuntu 等）配置

```bash
# 1. 创建配置目录
sudo mkdir -p /etc/docker

# 2. 写入加速器地址（替换为你的专属地址）
sudo tee /etc/docker/daemon.json <<-'EOF'
{
  "registry-mirrors": ["https://<你的专属编码>.mirror.aliyuncs.com"]
}
EOF

# 3. 重载配置并重启 Docker
sudo systemctl daemon-reload
sudo systemctl restart docker
```

### 验证生效

```bash
docker info | grep -A 2 "Registry Mirrors"
```

预期输出包含你的加速地址：

```
 Registry Mirrors:
  https://<你的专属编码>.mirror.aliyuncs.com/
```

---

## 3. Windows / macOS（Docker Desktop）配置

1. 打开 Docker Desktop → **Settings → Docker Engine**
2. 在 JSON 配置中加入 `registry-mirrors` 字段：

```json
{
  "registry-mirrors": ["https://<你的专属编码>.mirror.aliyuncs.com"]
}
```

3. 点击 **Apply & Restart**

---

## 4. 配置多镜像源（可选，提高可用性）

单个加速源失效时自动切换。写入 `/etc/docker/daemon.json`：

```json
{
  "registry-mirrors": [
    "https://<你的专属编码>.mirror.aliyuncs.com"
  ]
}
```

> 建议：阿里云加速地址账号专属、稳定性最好，放第一位即可。
> 其余公共加速源（如各云厂商、高校源）时效性变化较快，用前先验证可用性，不要盲目堆砌。

---

## 5. 配置生效后仍拉取失败的排查

| 现象 | 排查方向 |
| --- | --- |
| `docker info` 看不到加速地址 | `daemon.json` JSON 格式错误（多余逗号/引号），`journalctl -u docker -e` 看启动报错 |
| 拉取报 `timeout` / `EOF` | 加速源限流，稍后重试或更换镜像源 |
| 拉取报 `not found` | 仅 Docker Hub 官方镜像走加速；第三方仓库（如 `ghcr.io`、`registry.k8s.io`）不走，需完整域名前缀 |
| 服务器在内网/防火墙后 | 检查出口能否解析访问 `<编码>.mirror.aliyuncs.com`：`curl -v https://<编码>.mirror.aliyuncs.com/v2/` |

手动验证加速源可用性：

```bash
curl -I https://<你的专属编码>.mirror.aliyuncs.com/v2/
# 返回 200 或 401 均属正常（401 是未认证，说明服务在响应）
```

---

## 6. Docker Compose / Kubernetes 注意事项

- **Docker Compose**：无需额外配置，走 Docker daemon 的镜像配置。
- **Kubernetes（containerd）**：不走 Docker daemon.json，需配置 containerd：

```bash
# 编辑 /etc/containerd/config.toml，在 registry.mirrors 下追加：
[plugins."io.containerd.grpc.v1.cri".registry.mirrors."docker.io"]
  endpoint = ["https://<你的专属编码>.mirror.aliyuncs.com"]
```

修改后重启：`systemctl restart containerd`

---

## 参考链接

- [官方镜像加速 - 阿里云帮助文档](https://help.aliyun.com/zh/acr/user-guide/accelerate-the-pulls-of-docker-official-images)
- [容器镜像服务控制台](https://cr.console.aliyun.com)
- [ACR 控制台改版公告](https://help.aliyun.com/zh/acr/product-overview/changes-to-the-container-registry-console)
- [Docker 官方文档 - registry mirrors 配置](https://docs.docker.com/engine/daemon/#configure-registry-mirrors)
