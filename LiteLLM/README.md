# LiteLLM Docker Compose

使用 Docker Compose 部署 [LiteLLM](https://github.com/BerriAI/litellm) 代理服务，包含 PostgreSQL 数据库和 Cloudflare Tunnel。

## 服务组成

| 服务 | 镜像 | 说明 |
|------|------|------|
| `postgres` | `postgres:16-alpine` | LiteLLM 元数据存储数据库 |
| `litellm` | `ghcr.io/berriai/litellm:main-latest` | LiteLLM 代理服务 |
| `tunnel` | `cloudflare/cloudflared:latest` | Cloudflare Tunnel，对外暴露服务 |

## 前置要求

- Docker & Docker Compose
- Cloudflare Tunnel Token（在 Cloudflare Zero Trust 控制台创建）

## 配置

在 `LiteLLM/` 目录下创建 `.env` 文件，填入以下环境变量：

```env
# PostgreSQL（POSTGRES_DB 和 POSTGRES_USER 默认为 litellm，可省略）
POSTGRES_PASSWORD=your_strong_password

# LiteLLM 主密钥，用于 API 鉴权
LITELLM_MASTER_KEY=sk-your-master-key

# Cloudflare Tunnel Token
TUNNEL_TOKEN=your_cloudflare_tunnel_token

# 可选：自定义数据库连接串（默认自动拼接）
# DATABASE_URL=postgresql://litellm:your_password@postgres:5432/litellm
```

> **注意**：`POSTGRES_PASSWORD` 和 `LITELLM_MASTER_KEY` 为必填项。



## 启动

```bash
cd LiteLLM

# 启动所有服务（后台运行）
docker compose up -d

# 查看日志
docker compose logs -f litellm

# 停止服务
docker compose down
```

## 访问

- LiteLLM 默认监听容器内部端口 `4000`，通过 Cloudflare Tunnel 对外暴露，无需在宿主机映射端口。
- 如需本地直接访问，可在 `litellm` 服务中添加端口映射：

  ```yaml
  ports:
    - "4000:4000"
  ```

## 数据持久化

PostgreSQL 数据存储在名为 `postgres_data` 的 Docker volume 中，`docker compose down` 不会删除该 volume。若需彻底清除数据：

```bash
docker compose down -v
```
