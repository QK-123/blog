#### Python 项目镜像打包<Badge type="tip" text="docker" />

##### 1. 准备项目

- `requirements.txt` 列出项目依赖

##### 2. 创建Dockerfile

```bash
# 第一阶段：构建
FROM python:3.9-slim AS builder
WORKDIR /app
COPY . .
RUN pip install --no-cache-dir -r requirements.txt

# 第二阶段：运行
FROM python:3.9-slim
WORKDIR /app
COPY --from=builder /app /app
CMD ["python", "your_script.py"]
```

###### 优化手段

- 轻量级基础镜像
- 多阶段构建
- 精简依赖
- 使用 `.dockerignore` 忽略不需要的文件

##### 3. 构建镜像

```bash
docker build -t your-image-name:tag .
```

##### 4. 运行镜像

```bash
docker run --name your-container-name -d your-image-name:tag
```

##### 5. 发布镜像

```bash
docker push your-image-name:tag
```

##### 生产案例-FASTAPI 镜像打包

```bash

FROM python:3.9

WORKDIR /code

COPY ./requirements.txt /code/requirements.txt

RUN pip install --no-cache-dir --upgrade -r /code/requirements.txt

COPY ./app /code/app

CMD ["fastapi", "run", "app/main.py", "--port", "80"]
```

