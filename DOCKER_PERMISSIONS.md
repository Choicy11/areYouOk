# Docker权限配置说明

## 概述

本项目已优化Docker权限配置，支持动态用户ID，解决了在非特权容器环境下的权限问题。

## 新增功能

### 动态用户ID支持

现在支持通过环境变量动态设置容器内运行用户的UID和GID，避免主机与容器用户ID不匹配的问题。

### 环境变量

- `PUID`: 设置容器内运行用户的UID（默认：1001）
- `PGID`: 设置容器内运行用户的GID（默认：1001）

## 使用方法

### 1. 使用默认用户（UID=1001）

```bash
docker run -d \
  --name areyouok \
  -p 3000:3000 \
  -v $(pwd)/data:/app/data \
  areyouok:latest
```

### 2. 使用自定义用户ID

```bash
docker run -d \
  --name areyouok \
  -p 3000:3000 \
  -v $(pwd)/data:/app/data \
  -e PUID=1000 \
  -e PGID=1000 \
  areyouok:latest
```

### 3. 匹配主机用户

```bash
# 获取当前用户的UID和GID
HOST_UID=$(id -u)
HOST_GID=$(id -g)

docker run -d \
  --name areyouok \
  -p 3000:3000 \
  -v $(pwd)/data:/app/data \
  -e PUID=$HOST_UID \
  -e PGID=$HOST_GID \
  areyouok:latest
```

### 4. Docker Compose配置

```yaml
version: '3.8'
services:
  areyouok:
    image: areyouok:latest
    ports:
      - "3000:3000"
    volumes:
      - ./data:/app/data
      - ./logs:/app/logs
    environment:
      - PUID=1000  # 根据需要修改
      - PGID=1000  # 根据需要修改
    restart: unless-stopped
```

## 兼容性说明

### 向后兼容

- **完全兼容**：现有的start.sh和stop.sh脚本无需任何修改
- **无影响**：不设置PUID/PGID环境变量时，使用默认UID=1001
- **平滑升级**：现有容器可以无缝升级到新版本

### 原有脚本保持不变

- `docker-start.sh`: 启动脚本功能完全保留
- `stop.sh`: 停止脚本功能完全保留
- 所有原有的功能和配置都得到保留

## 权限处理逻辑

1. **默认情况**：如果未设置PUID/PGID，使用默认的nodejs用户（UID=1001）
2. **自定义情况**：如果设置了PUID/PGID，动态创建对应UID/GID的用户
3. **Root运行**：以root用户启动，entrypoint脚本负责用户切换
4. **权限修复**：自动修复关键目录的用户权限

## 解决的问题

### 1. 主机与容器用户ID不匹配
- **问题**：容器内创建的文件在主机上显示为未知用户
- **解决**：通过PUID/PGID匹配主机用户ID

### 2. 挂载目录权限问题
- **问题**：容器无法写入挂载的主机目录
- **解决**：动态调整用户权限，确保目录可访问

### 3. 数据持久化问题
- **问题**：容器重启后数据文件权限混乱
- **解决**：保持一致的用户权限设置

### 4. 多环境部署困难
- **问题**：不同环境用户ID分配不同
- **解决**：通过环境变量灵活配置

## 构建镜像

```bash
# 构建镜像（使用默认UID=1001）
docker build -t areyouok:latest .

# 构建镜像（指定自定义UID）
docker build --build-arg PUID=1000 --build-arg PGID=1000 -t areyouok:latest .
```

## 故障排除

### 1. 权限被拒绝
```bash
# 检查挂载目录权限
ls -la ./data

# 设置合适的权限
sudo chown -R $USER:$USER ./data
sudo chmod -R 755 ./data
```

### 2. 数据库文件权限问题
```bash
# 修复数据库文件权限
sudo chown $USER:$USER ./data/expense_bills.db
```

### 3. 日志写入失败
```bash
# 确保日志目录有写权限
mkdir -p ./logs
chmod 755 ./logs
```

## 最佳实践

1. **生产环境**：始终明确设置PUID/PGID环境变量
2. **开发环境**：使用当前用户ID匹配主机权限
3. **数据持久化**：确保挂载的目录有正确的权限
4. **监控日志**：检查entrypoint日志确认用户切换成功