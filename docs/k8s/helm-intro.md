### Helm 使用指南<Badge type="tip" text="k8s" />

#### 常用命令

##### 添加仓库

```bash
helm repo add [REPO_NAME] [REPO_URL]
```

##### 搜索 `chart`

```bash
helm search repo [KEYWORD]
```

##### 安装

```bash
helm install [RELEASE_NAME] [CHART]
```

##### 卸载

```bash
helm uninstall [RELEASE_NAME]
```

##### 列出发布

```bash
helm list
```

升级发布

```bash
helm upgrade [RELEASE_NAME] [CHART]
```

##### 回滚发布

```bash
helm history [RELEASE_NAME]

helm rollback [RELEASE_NAME] [REVISION]
```

#### 编写 `Chart` 

##### chart 目录结构

```bash
mychart/
├── .helmignore           # 忽略文件列表
├── Chart.yaml            # Chart 的描述文件
├── values.yaml           # 默认值配置文件
├── charts/               # 依赖的 Charts 存放目录
├── templates/            # Kubernetes 资源模板目录
│   ├── deployment.yaml   # 部署模板
│   ├── service.yaml      # 服务模板
│   └── _helpers.tpl      # 帮助模板
└── README.md             # Chart 的说明文档
```

##### 创建chart模板

```bash
helm create mychart

# 验证语法
helm lint

#打包
helm package mychart
```

##### 最佳实践

1. 使用GIT进行版本管理

2. 合理使用 `values.yml` 避免硬编码

3. 使用模板函数减少重复代码

   ```bash
   {{ if .Values.service.enabled }}
   apiVersion: v1
   kind: Service
   metadata:
     name: {{ .Release.Name }}-service
   spec:
     ...
   {{ end }}
   ```

   

















