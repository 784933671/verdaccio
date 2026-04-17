# Verdaccio Docker 部署指南

本手册介绍如何从零开始构建 Verdaccio Docker 镜像并部署到服务器。

## 1. 本地构建 Docker 镜像

### 前置条件

- 已安装 Docker
- 已克隆 Verdaccio 代码仓库

### 构建命令

在 Verdaccio 项目根目录下执行：

```bash
# 构建镜像（默认镜像名为 verdaccio/verdaccio:local）
docker build -t verdaccio/verdaccio:local .
```

构建完成后，可以通过以下命令验证镜像是否成功创建：

```bash
docker images | grep verdaccio
```

### 导出镜像（可选，用于离线部署）

如果服务器无法访问本地 Docker 环境，可以将镜像导出为文件：

```bash
# 导出镜像为 tar 文件
docker save -o verdaccio.tar verdaccio/verdaccio:local
```

## 2. 配置文件说明

### 需要的配置文件

Verdaccio 运行需要以下配置文件：

| 文件 | 用途 | 来源 |
|------|------|------|
| `config.yaml` | Verdaccio 主配置文件 | 项目内置或使用自定义配置 |

### 配置文件来源

#### 方式一：使用项目内置的 Docker 默认配置

项目已经内置了 Docker 环境使用的默认配置文件，位于：

```
packages/config/src/conf/docker.yaml
```

该文件在 Docker 镜像构建时会被自动复制到容器内的 `/verdaccio/conf/config.yaml`。

**如果您不需要修改配置，可以跳过此步骤，直接使用内置配置。**

#### 方式二：使用自定义配置文件

如果您需要自定义配置（如修改存储路径、添加认证、配置上游仓库等），可以创建自己的 `config.yaml`：

1. 复制项目内置配置作为模板：

```bash
# 在项目根目录执行
cp packages/config/src/conf/docker.yaml ./config.yaml
```

2. 编辑 `config.yaml` 文件，根据需求修改配置项。

常用配置项说明：

```yaml
# 包存储路径（容器内路径）
storage: /verdaccio/storage/data

# 插件路径
plugins: /verdaccio/plugins

# Web UI 设置
web:
  title: Verdaccio

# 认证配置
auth:
  htpasswd:
    file: /verdaccio/storage/htpasswd

# 上游仓库（代理）
uplinks:
  npmjs:
    url: https://registry.npmjs.org/

# 包访问控制
packages:
  '@*/*':
    access: $all
    publish: $authenticated
    proxy: npmjs
  '**':
    access: $all
    publish: $authenticated
    proxy: npmjs
```

## 3. 准备部署文件

### 创建 docker-compose.yaml

在项目根目录创建 `docker-compose.yaml` 文件：

```yaml
services:
  verdaccio:
    image: verdaccio/verdaccio:local
    container_name: verdaccio
    environment:
      - VERDACCIO_PORT=4873
    ports:
      - 4873:4873
    volumes:
      - ./data/storage:/verdaccio/storage
      - ./data/config:/verdaccio/conf
      - ./data/plugins:/verdaccio/plugins
    restart: unless-stopped
```

**注意**：如果您的服务器使用 1Panel 等面板，可能需要添加外部网络配置：

```yaml
services:
  verdaccio:
    image: verdaccio/verdaccio:local
    container_name: verdaccio
    networks:
      - 1panel-network
    environment:
      - VERDACCIO_PORT=4873
    ports:
      - 4873:4873
    volumes:
      - ./data/storage:/verdaccio/storage
      - ./data/config:/verdaccio/conf
      - ./data/plugins:/verdaccio/plugins
    labels:
      createdBy: 'Apps'
    restart: unless-stopped

networks:
  1panel-network:
    external: true
```

### 目录结构

部署时应确保以下目录结构：

```text
verdaccio/                          # 项目根目录
├── Dockerfile                      # Docker 镜像构建文件
├── docker-compose.yaml             # Docker Compose 配置文件
├── config.yaml                     # 自定义配置文件（可选）
├── packages/
│   └── config/
│       └── src/
│           └── conf/
│               └── docker.yaml     # 内置默认配置文件
└── data/                           # 数据持久化目录（运行时自动创建）
    ├── config/                     # 配置文件挂载点
    │   └── config.yaml             # Verdaccio 配置文件
    ├── storage/                    # 数据存储目录
    │   ├── data/                   # npm 包存储
    │   └── htpasswd                # 用户认证文件
    └── plugins/                    # 自定义插件目录
```

## 4. 服务器部署步骤

### 第一步：上传代码到服务器

将整个 Verdaccio 项目目录上传到服务器，或使用 Git 克隆：

```bash
git clone <your-repo-url> /path/to/verdaccio
cd /path/to/verdaccio
```

如果使用离线方式，需要上传：
- 构建好的 Docker 镜像 tar 文件
- docker-compose.yaml 文件
- 自定义 config.yaml 文件（如果有）

### 第二步：加载镜像（仅离线部署需要）

```bash
docker load -i verdaccio.tar
```

### 第三步：修正权限（关键）

Verdaccio 容器内部以非 root 用户（UID `10001`）运行。必须执行以下权限修正，否则会导致 `EACCES: permission denied` 错误：

```bash
# 创建数据目录
mkdir -p ./data/storage ./data/config ./data/plugins

# 修正目录所有者
sudo chown -R 10001:10001 ./data
```

### 第四步：启动服务

```bash
# 如果使用自定义端口，先声明环境变量
export VERDACCIO_PORT=4873

# 启动容器
docker-compose up -d
```

## 5. 验证部署

### 查看容器状态

```bash
docker ps
```

应该能看到 verdaccio 容器正在运行。

### 查看日志

```bash
docker logs -f verdaccio
```

### 访问 Web UI

在浏览器中访问：`http://服务器IP:4873`

## 6. 常用管理命令

| 操作 | 命令 |
| :--- | :--- |
| 查看运行状态 | `docker ps` |
| 查看实时日志 | `docker logs -f verdaccio` |
| 停止服务 | `docker-compose stop` |
| 重启服务 | `docker-compose restart` |
| 停止并删除容器 | `docker-compose down` |
| 更新配置 | 修改 `data/config/config.yaml` 后运行 `docker-compose restart` |

## 7. 客户端配置

在本地电脑使用该私有仓库：

```bash
# 设置 npm 镜像源
npm config set registry http://服务器IP:4873

# 登录/添加用户
npm adduser --registry http://服务器IP:4873

# 发布包
npm publish
```

## 8. 常见问题排查

### 启动报错：EACCES: permission denied

- **现象**：日志显示 `mkdir '/verdaccio/storage/data' failed`。
- **原因**：宿主机 `data` 目录权限不正确。
- **解决**：再次运行 `sudo chown -R 10001:10001 ./data`。

### 启动报错：network "xxx" not found

- **现象**：提示找不到 `1panel-network`。
- **解决**：
  - 手动创建该网络：`docker network create 1panel-network`。
  - 或者修改 `docker-compose.yaml` 移除外部网络依赖。

### 无法访问 Web UI

- **检查端口**：确认 `docker-compose.yaml` 中端口映射正确。
- **检查防火墙**：确保服务器防火墙允许访问对应端口。
- **检查容器状态**：运行 `docker ps` 确认容器正常运行。

### 配置不生效

- **检查挂载路径**：确认 `docker-compose.yaml` 中的 volumes 配置正确。
- **重启容器**：修改配置后需要运行 `docker-compose restart`。
