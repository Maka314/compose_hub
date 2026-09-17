# LiteLLM Docker Compose

使用 Docker Compose 部署 LiteLLM Gateway、PostgreSQL 和 Cloudflare Tunnel。LiteLLM 镜像固定为稳定版 `v1.101.0`，避免滚动标签在重建容器时引入未经验证的升级。

## 服务

| 服务 | 镜像 | 用途 |
| --- | --- | --- |
| `litellm` | `ghcr.io/berriai/litellm:v1.101.0` | API Gateway 和 Admin UI |
| `postgres` | `postgres:16-alpine` | 模型、虚拟密钥、用户和消费记录 |
| `tunnel` | `cloudflare/cloudflared:latest` | 通过 Cloudflare Tunnel 对外提供访问 |

LiteLLM 只在 Compose 内部网络暴露 `4000` 端口。Cloudflare Tunnel 的 Public Hostname 应指向 `http://litellm:4000`。

## 配置文件

- `docker-compose.yaml`：服务、健康检查和持久化卷。
- `litellm_config.yaml`：仓库安全的代理默认设置。模型列表为空，模型与供应商凭据通过 Admin UI 管理并存入 PostgreSQL。
- `.env.example`：所需环境变量的模板；真实 `.env` 已被仓库根目录的 `.gitignore` 排除。

## 初始化密钥

复制环境变量模板：

```bash
cp .env.example .env
```

生成三个互不相同的随机值：

```bash
echo "sk-$(openssl rand -hex 32)"  # LITELLM_MASTER_KEY
openssl rand -hex 32               # LITELLM_SALT_KEY
openssl rand -hex 32               # POSTGRES_PASSWORD
```

将生成值和 Cloudflare Tunnel Token 写入 `.env`。

`LITELLM_MASTER_KEY` 是网关管理员凭据，必须以 `sk-` 开头。`LITELLM_SALT_KEY` 用于加密存入数据库的供应商密钥；添加模型后不要更换它，否则已有凭据将无法解密。这里生成的 PostgreSQL 密码仅包含十六进制字符，可以安全用于 `DATABASE_URL`。

## 启动

```bash
docker compose config --quiet
docker compose up -d
docker compose ps
```

第一次启动可能需要等待数据库初始化和 LiteLLM schema migration。`tunnel` 会等到 LiteLLM 的 `/health/liveliness` 检查通过后启动。

查看状态和日志：

```bash
docker compose ps
docker compose logs --tail=100 litellm
docker inspect --format '{{json .State.Health}}' litellm
```

通过 Cloudflare 配置的域名访问 `/ui`，用户名为 `admin`，密码为 `LITELLM_MASTER_KEY`。在 **Models + Endpoints** 中添加模型和供应商凭据。

## `litellm_config.yaml`

默认配置如下：

```yaml
model_list: []

litellm_settings:
  drop_params: true
```

`STORE_MODEL_IN_DB=True` 已在 Compose 中启用，因此通过 Admin UI 添加的模型会保存在 PostgreSQL。若希望由 Git 管理静态模型，可以向 `model_list` 添加条目，并通过 `os.environ/VARIABLE_NAME` 引用 `.env` 中的供应商密钥；不要把真实密钥写入 YAML。

## 升级

升级时将 Compose 中的 LiteLLM 镜像改为经过确认的稳定版本，再执行：

```bash
docker compose pull
docker compose up -d
docker compose ps
```

升级前备份 PostgreSQL 数据。`LITELLM_SALT_KEY` 必须保持不变。

## 停止或删除

停止并删除容器和项目网络，同时保留 PostgreSQL 数据卷：

```bash
docker compose down
```

永久删除 PostgreSQL 数据卷：

```bash
docker compose down --volumes
```

第二条命令会永久删除 LiteLLM 中的模型、虚拟密钥、用户和消费记录。

## 官方参考

- [Docker Quickstart](https://docs.litellm.ai/docs/proxy/docker_quick_start)
- [Production Deployment](https://docs.litellm.ai/docs/proxy/deploy)
- [Configuration reference](https://docs.litellm.ai/docs/proxy/configs)
