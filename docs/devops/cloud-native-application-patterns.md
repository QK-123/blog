### 云原生应用的模式和技术

![image-20241203181419435](/image-20241203181419435.png)

#### 一， 一个仓库一个应用

每个库或服务都可以独立部署，都有独立的代码管理。

#### 二，API设计优先

- 帮助将应用程序融入分布式系统，便于不同团队协作分工。
- 通过提前设计 API，其他团队可以根据该 API 开发其解决方案。
- 提前定义契约，使与其他系统的集成更加稳健，并可在部署流水线中进行测试。
- API 的内部实现可以灵活更改，而不会影响依赖它的其他应用程序或团队。

#### 三，依赖管理

- 所有应用程序的依赖项必须在清单中明确列出。
- 应用依赖项需通过依赖管理工具从中央仓库下载。
- Java 应用通常使用 Maven 或 Gradle 等工具轻松实现上述原则。
- 应用程序对外部环境的唯一隐式依赖是语言运行时和依赖管理工具。
- 私有依赖项也必须通过依赖管理工具进行解析。

#### 四，设计 构建 发布 运行

严格分离设计、构建、发布和运行阶段；禁止运行时更改代码，以避免与构建阶段不一致；构建和发布产物必须不可变，并使用唯一标识以确保可重现性。

- **设计阶段**：决定技术、依赖项和工具，规划应用功能的实现。
- **构建阶段**：将代码和依赖项编译、打包为不可变的构建产物（Build），并确保其唯一标识
- **发布阶段**：构建产物与特定部署配置结合形成发布版本（Release）。每个版本必须唯一标识（如语义版本号 3.9.4 或时间戳 2022-07-07_17:21），并存储在中央仓库以便访问和回滚。
- **运行阶段**：应用基于特定的发布版本在执行环境中运行。

#### 五，配置 密钥和代码

- 配置是指在不同部署之间可能发生变化的所有内容，例如：
  - 支持服务的资源句柄（如数据库或消息系统）。
  - 访问第三方 API 的凭据。
  - 功能开关（Feature Flags）。

- 配置变更要求
  - 配置变更应无需修改代码或重新构建应用。
  - 配置必须外部化，不能直接包含在代码中或与代码库一起追踪。
  - 默认配置可随代码库打包，但其他配置需存储在独立的存储库中。

- 配置安全性检查
  - 检查代码库公开后是否会泄露凭据或特定环境信息，以判断配置是否正确外部化。

- 推荐做法
  - 将配置存储为环境变量，以便相同应用在不同环境中通过配置实现不同行为。

#### 六，日志

- 日志职责分离
  - 原生应用不负责日志的路由和存储。

- 日志处理要求
  - 应用程序将日志输出到标准输出（Standard Output）。
  - 应用程序将日志输出到标准输出（Standard Output）。

- 外部工具管理
  - 日志的存储、轮转等任务由外部工具（如日志聚合器）负责，这些工具负责获取、收集并提供日志供检查。

#### 七，应用可失败

1. 传统环境 vs 云环境
   - **传统环境**：注重维护应用程序持续运行，尽量避免终止。
   - **云环境**：应用程序是短暂的，可随时终止并启动新实例。

2. 弹性扩展能力
   - 失败时可终止无响应的实例，启动新实例。
   - 高负载时可增加更多实例以支持需求增长。

3. 可丢弃应用（Disposable Applications）
   - 可丢弃应用（Disposable Applications）
   - 快速启动：保证系统弹性，提升鲁棒性和弹性。
   - 优雅关闭，接收到终止信号后停止接受新请求；完成正在处理的请求后再退出；对于工作进程（Worker Processes），需将未完成的任务返回到队列后再退出。

4. 性能与可用性要求
   - 快速启动可避免性能和可用性问题。
   - 优雅关闭确保资源和任务的正确处理。

#### 八，外部服务

1. 定义支持服务（Backing Services）
   - 支持服务是应用程序用来提供功能的外部资源，例如：数据库 消息代理 缓存系 SMTP/FTP 服务器 RESTful 网络服务
2. 附加资源的概念
   - 将支持服务视为附加资源（Attached Resources）。
   - 附加资源可随时更换，而无需修改应用代码。
3. 环境适配
   - 不同开发阶段可使用不同的服务（如开发、测试和生产环境使用不同数据库）。
   - 通过附加资源的灵活性适应多环境需求。
4. 资源绑定（Resource Binding）
   - 附加资源通过绑定实现，例如：数据库的绑定信息包含 URL、用户名和密码。
   - 绑定机制确保支持服务的动态替换与配置。

#### 九，环境一致性

1. 环境一致性
   - 保持所有环境尽可能相似，减少差异，提升开发、测试和生产环境的一致性和稳定性。
2. 解决的问题
   - **时间差距**：代码变更与部署之间的时间较长，通过自动化和持续部署缩短从开发到生产的周期。
   - **人员差距**：开发者负责构建应用，运维人员负责生产部署，通过采用 DevOps 文化，促进开发与运维的协作，推行“你构建，你运行”的理念。
   - **工具差距**：不同环境间的支持服务使用方式不同，例如，开发环境使用 H2 数据库，生产环境使用 PostgreSQL，建议在所有环境中使用相同类型和版本的支持服务。

#### 十，管理流程

1. 管理任务定义
   - 管理任务包括数据库迁移、批处理作业、维护作业等。
   - 管理任务包括数据库迁移、批处理作业、维护作业等。

2. 管理任务的要求
   - 与应用程序代码一起被版本控制和交付。
   - 在与应用程序相同的环境中执行。

3. 管理任务的实现方式
   - 作为独立的服务：一次运行后即销毁。
   - 配置为无状态平台中的函数：在特定事件触发时运行。
   - 嵌入到应用程序中：通过调用特定端点激活。

4. 目标
   - 使管理任务高效执行并与应用程序保持一致性，同时保持灵活性和独立性。

#### 十一，端口绑定

1. 自包含应用程序
   - 遵循 15-Factor 方法的应用程序应为自包含的，通过端口绑定（Port Binding）导出其服务。
   - 不依赖执行环境中的外部服务器。
2. 与传统应用的区别
   - 传统的 Java Web 应用通常运行在服务器容器（如 Tomcat、Jetty 或 Undertow）中。
   - 云原生应用无需依赖环境中的服务器，而是自主管理其依赖项，如 Spring Boot 可以使用嵌入式服务器。
3. 一对一映射
   - 云原生方法实现应用与服务器的一一映射，而不是传统方法中多个应用共享同一服务器的部署方式。
4. 端口绑定服务导出
   - 应用程序通过端口绑定导出服务，例如，Web 应用将 HTTP 服务绑定到特定端口。
   - 应用程序可能成为其他应用的支持服务，这在云原生系统中非常常见。

#### 十二，无状态进程

1. 高可扩展性和云迁移
   - 高可扩展性是迁移到云端的主要原因之一。
2. 无状态应用程序
   - 设计应用程序为无状态（Stateless）进程，采用不共享任何状态的架构（Share-Nothing Architecture）。
   - 确保不同应用实例之间没有共享状态。
3. 检查是否无状态
   - 询问自己：如果应用程序实例被销毁并重新创建，是否会丢失数据？
   - 如果答案是“是”，说明该应用程序不是无状态的。
4. 状态管理
   - 应用程序必须在某些情况下保存状态，否则应用程序大多数情况下会变得无用。
   - 设计无状态应用程序时，将状态管理和存储委托给特定的有状态服务（如数据存储）。
   - 无状态应用程序将状态管理和存储外包给支持服务。

#### 十二，并发

1. 无状态应用程序不足以确保可扩展性，还需要支持并发处理。
2. 应用程序需要支持并发处理，以便同时服务多个用户。
3. 将进程视为一等公民，支持横向扩展（Horizontal Scalability），将工作负载分配到不同机器上的多个进程中；将进程视为一等公民，支持横向扩展（Horizontal Scalability），将工作负载分配到不同机器上的多个进程中。
4. 通过线程池中的多个线程来处理并发。
5. 进程可以根据类型进行分类，例如：**Web 进程**：处理 HTTP 请求；**Worker 进程**：在后台执行计划任务。

#### 十三，可观测

1. 可观察性是云原生应用程序的重要特性。
2. 在云中管理分布式系统很复杂，必须确保每个系统组件都提供正确的数据来远程监控系统行为。
3. 日志（Logs）指标（Metrics）跟踪（Traces）健康状态（Health Status）事件（Events）
4. Hoffman 使用了“将应用程序视为太空探测器”的比喻，强调遥测数据的重要性。
5. 需要收集和提供适当的遥测数据，以便远程监控和控制应用程序。

#### 十四，验证和授权

1. 安全性是软件系统的基本特性之一，但往往没有得到足够的重视。
2. 必须在系统的任何架构和基础设施层级上保障交互的安全，采用零信任方法。
3. 安全不仅仅是身份验证和授权，但它们是良好的起点；身份验证用于跟踪谁在使用应用程序；授权检查用户权限来验证是否允许用户执行特定操作。
4. 可用于实现身份与访问管理的标准包括 OAuth 2.1 和 OpenID Connect。







































































