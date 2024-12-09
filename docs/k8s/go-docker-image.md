### GO 项目镜像打包<Badge type="tip" text="docker" />

#### 1. 准备项目

`go.mod go.sum` 列出项目依赖

#### 2. 创建Dockerfile

```bash
# 使用官方 Go 镜像作为构建阶段
FROM golang:1.20 AS builder

# 设置工作目录
WORKDIR /app

# 将 go.mod 和 go.sum 复制到工作目录
COPY go.mod go.sum ./
# 下载依赖
RUN go mod download

# 将其余的源代码复制到工作目录
COPY . .

# 编译应用
RUN go build -o myapp .

# 使用轻量级的基础镜像
FROM alpine:latest

# 将编译好的二进制文件复制到新镜像中
COPY --from=builder /app/myapp /myapp

# 指定容器启动时运行的命令
CMD ["/myapp"]

EXPOSE 8080
```

#### 3. 优化手段

- 多阶段构建

- 去除调试信息

  ```bash
  go build -ldflags="-w -s" -o myapp .
  ```

- 使用给更小的基础镜像

#### 4. 构建镜像

```bash
docker build -t my-go-app .
```

#### 5. 运行镜像

```bash
docker build -t my-go-app .
```

#### 6. 发布镜像

```bash
docker push my-go-app
```



