# 云原生系统中的数据库 

## 1. 引言

在云原生架构中，应用程序服务通常设计为无状态（stateless），以便于水平扩展和动态调度。然而，大多数应用都需要持久化存储状态和数据。本章将探讨云原生环境中的数据持久化，重点关注数据库的选择、管理和与 Spring Boot 应用的集成。我们将构建一个名为"Catalog Service"的示例应用，演示如何使用 PostgreSQL 作为关系型数据库，并通过 Spring Data JDBC 进行数据访问。

## 2. 数据服务 vs. 应用服务

在云原生系统中，我们可以区分两种类型的服务：

*   **应用服务 (Application Services):** 通常是无状态的，处理业务逻辑，易于水平扩展。它们主要利用云基础设施的计算和网络资源。
*   **数据服务 (Data Services):** 有状态的组件，负责持久化数据。它们需要云基础设施的存储资源，例如 PostgreSQL, Cassandra, Redis, RabbitMQ, Apache Kafka 等。

下图展示了云基础架构构建块以及应用服务和数据服务如何使用它们：

![image-20250211130813158](/image-20250211130813158.png)

## 3. 云原生数据库的挑战 

在云环境中选择和管理数据服务时，需要考虑以下几个关键属性：

*   **可扩展性 (Scalability):** 数据服务应能动态地横向扩展，以适应不断变化的工作负载。
*   **弹性 (Resilience):** 数据服务应能容忍故障，并保证数据持久性和可用性。数据复制是实现弹性的关键策略。
*   **性能 (Performance):** 数据访问延迟、网络延迟和存储技术都会影响性能。
*   **合规性 (Compliance):**  持久化数据通常包含敏感信息，必须符合相关的法律、法规和客户协议（例如 GDPR, CCPA, HIPAA）。

## 4. 数据服务的类型

云原生环境中的数据服务可以分为两大类：

*   **云提供商托管 (Managed by a cloud provider):**
    *   云提供商负责管理所有方面，包括可扩展性、弹性、性能和安全性。
    *   示例：Amazon RDS, Azure Database, Google Cloud SQL, Google BigQuery, Azure Cosmos DB。
*   **自行管理 (Managed by you):**
    *   您需要自行管理数据服务，获得更多控制权，但也增加了复杂性。
    *   可以使用虚拟机或容器 (例如 Kubernetes) 来部署和管理。
    *   示例：在容器中运行 PostgreSQL, Redis, RabbitMQ, MongoDB 等。

下图展示了这两种数据服务类别的对比：

![image-20250211130848737](/image-20250211130848737.png)

**技术选型比较 (Trade-offs):**

| 方案               | 优点                                                         | 缺点                                                         |
| ------------------ | ------------------------------------------------------------ | ------------------------------------------------------------ |
| 云提供商托管服务   | 简化运维，降低成本，高可用性，弹性伸缩，自动备份和恢复       | 可能受限于云提供商的功能和特性，可能存在供应商锁定，成本可能较高（取决于使用情况） |
| 自行管理（容器）   | 更高的控制权，可移植性强，可利用 Kubernetes 的编排能力，成本可能较低（取决于优化情况） | 运维复杂性较高，需要自行管理高可用性、备份恢复、安全性等     |
| 自行管理（虚拟机） | 更高的控制权，技术成熟                                       | 运维复杂性最高，难以实现弹性伸缩，资源利用率较低             |

**对于初学者或小型项目，建议优先选择云提供商托管服务。对于需要高度定制化、特定性能要求或有合规性限制的大型项目，可以考虑自行管理。**

## 5. 使用 Docker 运行 PostgreSQL

为了在本地开发环境中模拟生产环境，我们将使用 Docker 运行 PostgreSQL 数据库。

**命令行命令 (Command Line Commands):**

```bash
docker run -d \
--name polar-postgres \
-e POSTGRES_USER=user \
-e POSTGRES_PASSWORD=password \
-e POSTGRES_DB=polardb_catalog \
-p 5432:5432 \
postgres:14.4
```

**命令解释:**

*   `docker run -d`: 在后台运行容器。
*   `--name polar-postgres`: 为容器指定名称。
*   `-e POSTGRES_USER=user`: 设置 PostgreSQL 用户名。
*   `-e POSTGRES_PASSWORD=password`: 设置 PostgreSQL 密码。
*   `-e POSTGRES_DB=polardb_catalog`: 设置要创建的数据库名称。
*   `-p 5432:5432`: 将容器的 5432 端口映射到主机的 5432 端口。
*   `postgres:14.4`: 使用的 PostgreSQL 镜像版本。

**停止和删除容器:**
```bash
docker stop polar-postgres
docker rm -fv polar-postgres
```
## 6. 使用 Spring Data JDBC 进行数据持久化

Spring Data 项目提供了对多种数据持久化技术的支持，包括关系型数据库（JDBC, JPA, R2DBC）和非关系型数据库（Cassandra, Redis, Neo4J, MongoDB 等）。Spring Data 提供了通用的抽象和模式，使得在不同模块之间切换变得容易。本节将重点关注关系型数据库，特别是 Spring Data JDBC。

Spring Data JDBC 和 Spring Data JPA 的比较如下：

| 特性     | Spring Data JDBC                                             | Spring Data JPA                                              |
| -------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| 基础     | JDBC (Java Database Connectivity)                            | JPA (Java Persistence API), 通常使用 Hibernate 作为实现      |
| 理念     | 领域驱动设计 (DDD) 概念，如聚合（Aggregates）、聚合根（Aggregate Roots）和存储库（Repositories） | 对象关系映射 (ORM)                                           |
| 复杂性   | 相对简单、轻量级                                             | 相对复杂，功能强大，需要深入理解 JPA 和 Hibernate            |
| 控制力   | 对 SQL 查询有更多控制，支持不可变实体                        | 自动生成 SQL，对 SQL 的控制较少                              |
| 适用场景 | 微服务，领域模型简单，需要更多控制 SQL 的场景                | 企业级应用，领域模型复杂，需要 JPA 提供的丰富功能的场景      |
| 实体类   | 建议使用不可变对象（例如 Java Records), 使用 `@Id` 标注主键字段, 使用`org.springframework.data.annotation` 包内的注解。 | 使用可变对象（Java Classes），必须有无参构造函数, 使用`@Entity`标注实体，使用`javax.persistence`包内的注解。 |

下图描述了使用 Spring Data 的应用程序与数据库之间的交互：

![image-20250211131257316](/image-20250211131257316.png)

Spring Data交互的几个主要元素是：
* 数据库驱动程序
* 实体
* 存储库
本节将指导您如何使用 Spring Data JDBC 将数据持久化功能添加到 Spring Boot 应用程序（如 Catalog Service）。您将配置连接池以通过 JDBC 驱动程序与 PostgreSQL 数据库交互，定义要持久化的实体，使用存储库访问数据，并使用事务。
本节完成后的 Polar Bookshop 架构图将更新为下图所示：

![image-20250211131322891](/image-20250211131322891.png)

## 7. 配置数据库连接

首先，我们需要在 `build.gradle` 文件中添加 Spring Data JDBC 和 PostgreSQL 驱动程序的依赖：

```groovy
dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-data-jdbc'
    runtimeOnly 'org.postgresql:postgresql'
}
```

然后，在 `application.yml` 文件中配置数据库连接信息和连接池：

```yaml
spring:
  datasource:
    username: user
    password: password
    url: jdbc:postgresql://localhost:5432/polardb_catalog
    hikari:
      connection-timeout: 2000
      maximum-pool-size: 5
```

**配置解释:**

*   `spring.datasource.username`: 数据库用户名。
*   `spring.datasource.password`: 数据库密码。
*   `spring.datasource.url`: JDBC 连接 URL。
*  `spring.datasource.hikari.connection-timeout`: 连接超时时间（毫秒）。
*  `spring.datasource.hikari.maximum-pool-size`: 连接池最大连接数。

## 8. 定义持久化实体

在 Catalog Service 中，我们已经有一个 `Book` record 来表示领域实体。现在，我们将更新 `Book` record，使其成为一个持久化实体：

```java
package com.polarbookshop.catalogservice.domain;

import org.springframework.data.annotation.CreatedDate;
import org.springframework.data.annotation.Id;
import org.springframework.data.annotation.LastModifiedDate;
import org.springframework.data.annotation.Version;

import javax.validation.constraints.NotBlank;
import javax.validation.constraints.NotNull;
import javax.validation.constraints.Pattern;
import javax.validation.constraints.Positive;
import java.time.Instant;

public record Book (
    @Id
    Long id,

    @NotBlank(message = "The book ISBN must be defined.")
    @Pattern(
        regexp = "^([0-9]{10}|[0-9]{13})$",
        message = "The ISBN format must be valid."
    )
    String isbn,

    @NotBlank(message = "The book title must be defined.")
    String title,

    @NotBlank(message = "The book author must be defined.")
    String author,

    @NotNull(message = "The book price must be defined.")
    @Positive(message = "The book price must be greater than zero.")
    Double price,
        
    String publisher, //Optional

    @CreatedDate
    Instant createdDate,

    @LastModifiedDate
    Instant lastModifiedDate,

    @Version
    int version
){
  public static Book of(
          String isbn, String title, String author, Double price, String publisher
  ) {
    return new Book(null, isbn, title, author, price,publisher, null, null, 0);
  }
}
```

**代码解释:**

*   `@Id`: 标记该字段为主键。
*   `@Version`: 标记该字段为乐观锁版本号。
*   `@CreatedDate`: 标记该字段为创建时间。
*   `@LastModifiedDate`: 标记该字段为最后修改时间。
*   `of()`：静态工厂方法，方便创建 `Book` 对象。

还需要修改`BookService`类中`editBookDetails()`方法：
```java
    public Book editBookDetails(String isbn, Book book) {
        return bookRepository.findByIsbn(isbn)
                .map(existingBook -> {
                    var bookToUpdate = new Book(
                            existingBook.id(),
                            existingBook.isbn(),
                            book.title(),
                            book.author(),
                            book.price(),
                            existingBook.publisher(),//
                            existingBook.createdDate(),
                            existingBook.lastModifiedDate(),
                            existingBook.version());
                    return bookRepository.save(bookToUpdate);
                })
                .orElseGet(() -> addBookToCatalog(book));
    }
```

接下来，创建 `src/main/resources/schema.sql` 文件，定义数据库表结构：

```sql
DROP TABLE IF EXISTS book;
CREATE TABLE book (
    id BIGSERIAL PRIMARY KEY NOT NULL,
    author varchar(255) NOT NULL,
    isbn varchar(255) UNIQUE NOT NULL,
    price float8 NOT NULL,
    title varchar(255) NOT NULL,
    publisher varchar(255), --
    created_date timestamp NOT NULL,
    last_modified_date timestamp NOT NULL,
    version integer NOT NULL
);
```

最后，在`application.yml`中启用schema初始化。
```
spring:
  sql:
    init:
      mode: always
```
## 9. 启用和配置 JDBC 审计

为了捕获审计事件（创建时间、最后修改时间等），我们需要启用 JDBC 审计。

创建 `com.polarbookshop.catalogservice.config.DataConfig` 类：

```java
package com.polarbookshop.catalogservice.config;

import org.springframework.context.annotation.Configuration;
import org.springframework.data.jdbc.repository.config.EnableJdbcAuditing;

@Configuration
@EnableJdbcAuditing
public class DataConfig {}
```

## 10. 数据存储库

Spring Data 提供了存储库（Repository）模式，用于抽象数据访问逻辑。我们只需要定义接口，Spring Data 会在运行时自动生成实现。

修改 `BookRepository` 接口：

```java
package com.polarbookshop.catalogservice.domain;

import org.springframework.data.jdbc.repository.query.Modifying;
import org.springframework.data.jdbc.repository.query.Query;
import org.springframework.data.repository.CrudRepository;
import org.springframework.transaction.annotation.Transactional;

import java.util.Optional;

public interface BookRepository extends CrudRepository<Book, Long> {

    Optional<Book> findByIsbn(String isbn);

    boolean existsByIsbn(String isbn);

    @Modifying
    @Transactional
    @Query("delete from Book where isbn = :isbn")
    void deleteByIsbn(String isbn);
}
```

**代码解释:**

*   `extends CrudRepository<Book, Long>`: 继承 `CrudRepository` 接口，提供基本的 CRUD 操作。
*   `findByIsbn(String isbn)`: 根据 ISBN 查询 Book。
*   `existsByIsbn(String isbn)`: 检查 ISBN 是否存在。
*   `deleteByIsbn(String isbn)`: 根据 ISBN 删除 Book。
*    `@Modifying`: 标识该方法会修改数据库状态。
*   `@Transactional`: 标识该方法在事务中执行。
*   `@Query`: 自定义 SQL 查询。

修改`BookDataLoader`:
```java
    @EventListener(ApplicationReadyEvent.class)
    public void loadBookTestData() {
        bookRepository.deleteAll();
        var book1 = Book.of("1234567891", "Northern Lights", "Lyra Silverstar", 9.90, "Polarsophia");
        var book2 = Book.of("1234567892", "Polar Journey", "Iorek Polarson", 12.90, "Polarsophia");
        bookRepository.saveAll(List.of(book1, book2));
    }
```