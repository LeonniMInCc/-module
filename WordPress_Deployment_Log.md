# 阿里云 Ubuntu 服务器 WordPress 部署工作日志

## 1. 环境准备
- **服务器**: 阿里云 ECS (Ubuntu 操作系统)
- **连接工具**: OrcaTerm
- **操作账号**: root
- **目标**: 部署 WordPress 个人网站

## 2. 部署流程
### 2.1 确认当前环境与需求
- 登录后根据 `bt default` 命令输出以及相关沟通，确认服务器暂未安装宝塔面板。
- 沟通后确定采用更现代、轻量、无残留的 **Docker Compose** 容器化部署方案。

### 2.2 安装 Docker 环境
- 执行基础安装：
  ```bash
  apt update
  apt install docker.io docker-compose -y
  systemctl enable --now docker
  ```
- **解决旧版 Docker Compose (Python v1) 兼容性问题**：在执行 `docker-compose up -d` 时出现 `urllib3 chunked` 异常。系统原有的基于 Python 的 `docker-compose v1` 库与较新的系统网络组件不兼容。
- 采取应对措施：卸载 v1，换用基于 Go 语言重写的 `docker-compose-v2`。
  ```bash
  apt remove docker-compose -y
  apt install docker-compose-v2 -y
  ```

### 2.3 解决网络拉取超时问题
- 在尝试通过新命令拉取 WordPress 和 MySQL 镜像时，遇到了访问 Docker 官方镜像库网络超时 (i/o timeout) 的问题（国内云服务器典型特征）。
- 采取应对措施：配置了国内的镜像加速节点（DaoCloud、DockerPull 等）。
  ```bash
  cat > /etc/docker/daemon.json << 'EOF'
  {
    "registry-mirrors": [
      "https://docker.m.daocloud.io",
      "https://dockerpull.com",
      "https://docker.1panel.live"
    ]
  }
  EOF
  systemctl restart docker
  ```

### 2.4 配置与启动 WordPress
- 编写容器编排配置 `docker-compose.yml`，成功将 MySQL 8.0 数据库服务与 WordPress 应用服务连接。
- 执行 `docker compose up -d` 完成拉取与启动。

### 2.5 后续网络环境配置指导
- 已提示在阿里云控制台 **安全组** 添加入方向规则，放行 `80` 端口。
- 最终成功通过公网 IP 在浏览器访问到 WordPress 安装界面。

## 3. 总结
顺利完成目标。全程未使用繁重的管理面板环境，确保了云服务器环境的纯净与高效。遇到并妥善排除了由“软件源老旧 Bug”以及“国内网络环境受限”带来的两项阻碍。