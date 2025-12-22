# Sky-Take-Out 项目部署文档

本文档旨在指导您如何从一台全新的服务器开始，完整地部署 Sky-Take-Out 全栈应用。

---

## 1. 环境要求 (Prerequisites)

在开始部署之前，请确保您的服务器（推荐使用 CentOS 7+ 或 Ubuntu 20.04+）已安装以下软件：

- **Git**: 用于从代码仓库拉取项目代码。
- **Docker**: 用于运行应用程序的容器化环境。
- **Docker Compose**: 用于编排和管理多个 Docker 容器（应用、数据库、Nginx 等）。
- **Maven**: 用于打包 Java 后端应用程序。

### 安装命令参考

#### 在 CentOS 上安装：
```bash
# 安装 Git
sudo yum install -y git

# 安装 Maven
sudo yum install -y maven

# 安装 Docker
sudo yum install -y yum-utils
sudo yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo
sudo yum install -y docker-ce docker-ce-cli containerd.io

# 启动 Docker
sudo systemctl start docker
sudo systemctl enable docker

# 安装 Docker Compose
sudo curl -L "https://github.com/docker/compose/releases/latest/download/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose
```

#### 在 Ubuntu 上安装：
```bash
# 安装 Git
sudo apt-get update
sudo apt-get install -y git

# 安装 Maven
sudo apt-get install -y maven

# 安装 Docker
sudo apt-get install -y apt-transport-https ca-certificates curl software-properties-common
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable"
sudo apt-get update
sudo apt-get install -y docker-ce docker-ce-cli containerd.io

# 安装 Docker Compose
sudo curl -L "https://github.com/docker/compose/releases/latest/download/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose
```

---

## 2. 部署步骤 (Deployment Steps)

### 步骤 1: 获取项目代码
将本项目克隆或上传到您服务器的任意目录，例如 `/root/sky-take-out`。
```bash
git clone <您的项目git仓库地址>
cd sky-take-out
```

### 步骤 2: 配置项目
所有部署相关的配置都位于 `deployment` 文件夹中。
1.  进入 `deployment` 目录。
    ```bash
    cd deployment
    ```
2.  编辑 `.env` 文件，填入您的真实配置。
    ```bash
    vim .env
    ```
    您需要**务必修改**以下内容：
    - `MYSQL_ROOT_PASSWORD`: 设置一个新的、强壮的数据库 root 用户密码。
    - `REDIS_PASSWORD`: 设置一个新的 Redis 密码。
    - 其他微信支付、对象存储等配置（如果需要）。

### 步骤 3: 打包后端应用
在**项目根目录**下执行 Maven 命令，将 Java 后端代码打包成 JAR 文件。
```bash
# (确保您在项目根目录，而不是 deployment 目录)
cd .. 
mvn clean package -DskipTests
```
该命令会编译项目并在 `sky-server/target/` 目录下生成 `sky-server-1.0-SNAPSHOT.jar`。Docker 在构建镜像时会自动使用它。

### 步骤 4: 启动服务
这是最关键的一步。所有服务都将通过 `docker-compose` 启动。
1.  **确保您位于 `deployment` 文件夹下**。
    ```bash
    cd deployment
    ```
2.  执行以下命令来构建并启动所有服务：
    ```bash
    docker-compose up --build -d
    ```
    - `up`: 创建并启动容器。
    - `--build`: 在启动前，如果镜像不存在或需要更新（如此处的 `app` 服务），则进行构建。
    - `-d`: 在后台（detached mode）运行容器，这样关闭终端后服务依然运行。

### 步骤 5: 验证部署
1.  查看所有容器的运行状态：
    ```bash
    docker-compose ps
    ```
    您应该能看到 `sky-nginx`, `sky-app`, `sky-mysql`, `sky-redis` 四个容器都处于 `Up` 或 `running` 状态。
2.  访问应用：
    打开您的浏览器，直接访问 `http://<您的服务器IP地址>`。如果一切正常，您应该能看到项目的前端登录页面。

---

## 3. 日常维护 (Maintenance)

所有维护命令都应在 `deployment` 文件夹下执行。

- **查看实时日志**:
  ```bash
  # 查看所有服务的日志
  docker-compose logs -f

  # 只看后端应用的日志
  docker-compose logs -f app
  ```

- **停止所有服务**:
  ```bash
  docker-compose down
  ```
  *(此命令会停止并移除容器，但通过 `volumes` 持久化的数据（如数据库文件）会保留下来)*

- **重启服务**:
  ```bash
  # 重启所有服务
  docker-compose restart

  # 只重启后端应用
  docker-compose restart app
  ```

- **更新应用程序版本**:
  当您更新了代码后，部署流程如下：
  1. 在服务器上拉取最新代码: `git pull`
  2. 重新打包后端应用: `mvn clean package -DskipTests` （在项目根目录执行）
  3. 重新构建并启动服务: `docker-compose up --build -d` （在 `deployment` 目录执行）

---

## 4. 关键文件结构概览

- `deployment/docker-compose.yml`: 核心编排文件，定义了 `nginx`, `app`, `mysql`, `redis` 四个服务以及它们之间的关系。
- `deployment/.env`: 环境变量文件，用于存储密码等敏感配置。
- `deployment/nginx.conf`: Nginx 的配置文件，负责处理前端静态文件和后端 API 的反向代理。
- `deployment/数据库/sky.sql`: 数据库初始化脚本，在数据库容器首次启动时自动执行。
- `sky-server/Dockerfile`: 后端 Java 应用的 Docker 镜像定义文件。

