# Spring Boot 云原生开发：入门指南

本教程涵盖了使用 Spring Boot、Spring MVC 及相关技术构建云原生应用的基础知识。我们将重点构建一个“目录服务”的 RESTful API，作为更大的“极地书店”系统的一部分。我们还将介绍测试和构建基本的 CI/CD 流水线。

## 技术栈

| 技术名称             | 版本     | 技术发布时间（大约） | 备注                                    |
| -------------------- | -------- | -------------------- | --------------------------------------- |
| Java                 | 17       | 2021 年 9 月         | 编程语言                                |
| Spring Boot          | 3.x      | 2022 年 11月         | 框架                                    |
| Spring MVC           | 6.x      | 2022 年 11月         | Web 框架 (Spring Boot 内置)             |
| Gradle               | 7.x/8.x  | 2021 年后            | 构建工具                                |
| Tomcat (Embedded)    | 9.x/10.x | 随 Spring Boot 版本  | 嵌入式 Servlet 容器                     |
| JUnit 5              | 5.x      | 2017 年 9 月         | 单元测试框架                            |
| Mockito              | 4.x/5.x  | 2021 年后            | Mocking 框架                            |
| AssertJ              | 3.x      | 2013 年后            | 断言库                                  |
| Spring Test          | 6.x      | 2022年11月           | 集成测试                                |
| WebTestClient        | 6.x      | 2022年11月           | 测试工具(Spring WebFlux)                |
| Java Bean Validation | 2.x/3.x  | -                    | 数据校验 API                            |
| Jackson              | 2.x      | -                    | JSON 序列化/反序列化 (Spring Boot 内置) |
| Grype                | 0.7x     | 2023年后             | 漏洞扫描工具                            |
| GitHub Actions       | -        | 2019 年 11 月        | CI/CD 平台                              |

## 3.1 构建云原生项目

开始一个新的云原生项目时，遵循最佳实践至关重要。15 因子应用程序方法论提供了出色的指导方针。这里，我们将重点关注：

*   **一个代码库，一个应用程序：** 每个应用程序都应该有一个在版本控制系统（如 Git）中跟踪的单一代码库。
*   **依赖管理：** 使用 Gradle 或 Maven 等工具显式管理依赖项。

### 3.1.1 一个代码库，一个应用程序

一个云原生应用程序映射到一个单一的代码库。此代码库生成可部署到多个环境的不可变工件（构建）。

![image-20250207101811773](/image-20250207101811773.png)

*图 3.1 每个应用程序都有其代码库，从中生成不可变的构建，然后部署到适当的环境，而无需更改代码。*

建议将每个代码库跟踪在其自己的 Git 存储库中，以提高可维护性和可部署性。对于本项目，目录服务将位于 `catalog-service` Git 存储库中。

### 3.1.2 依赖管理 (Gradle/Maven)

可靠且可移植的依赖管理是关键。在 Java 生态系统中，Gradle 和 Maven 是主要的选择。它们都：

*   在清单中声明依赖项（例如，Gradle 的 `build.gradle`，Maven 的 `pom.xml`）。
*   从中央存储库（例如，Maven Central）下载依赖项。

**技术选择：** 虽然两者都很优秀，但本教程使用 **Gradle**，因为它具有简洁的语法和灵活性。但是，这些概念可以直接转移到 Maven。示例使用 Gradle，但您也可以随意使用 Maven。

`build.gradle` 文件列出了所有必需的依赖项，确保应用程序不依赖于部署环境中的隐式库。

```gradle
// catalog-service/build.gradle
dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-web'
    testImplementation 'org.springframework.boot:spring-boot-starter-test'
    implementation 'org.springframework.boot:spring-boot-starter-validation' // 用于 Bean Validation
    testImplementation 'org.springframework.boot:spring-boot-starter-webflux' // 用于 WebTestClient
}
```

**包装器脚本 (gradlew/mvnw)：**

在您的代码库中包含 Gradle 包装器 (`gradlew`) 或 Maven 包装器 (`mvnw`)。这确保每个人（和 CI/CD 系统）使用*完全相同*的构建工具版本，避免“在我的机器上工作”的问题。使用 `./gradlew build` 而不是 `gradle build` 运行构建。

## 3.2 使用嵌入式服务器

Spring Boot 通过使用*嵌入式服务器*简化了 Web 应用程序开发。这与将 WAR/EAR 文件部署到外部应用程序服务器（如 Tomcat 或 WildFly）的传统方法形成对比。

**嵌入式服务器的优点：**

*   **自包含：** 没有外部服务器依赖项。
*   **可移植性：** 可在任何有 JVM 的地方运行。
*   **简化部署：** 打包为可执行 JAR。
*   **云原生友好：** 非常适合容器化。

**15 因子应用程序原则：**

*   **端口绑定：** 云原生应用程序绑定到可配置的端口。
*   **并发性：** 通过线程池处理并发；倾向于水平扩展（更多实例）而不是垂直扩展（每个实例更多资源）。

### 3.2.1 可执行 JAR 和嵌入式服务器

云原生 Java 应用程序通常打包为可执行 JAR（“胖 JAR”或“超级 JAR”）。这些 JAR 包含应用程序代码、依赖项*和*嵌入式服务器。

要将应用程序打包为 JAR：

```bash
./gradlew bootJar
```

这将在 `build/libs` 目录中创建一个可执行 JAR（例如，`catalog-service-0.0.1-SNAPSHOT.jar`）。像任何标准 Java 应用程序一样运行它：

```bash
java -jar build/libs/catalog-service-0.0.1-SNAPSHOT.jar
```

### 3.2.2 理解每个请求一个线程模型

Tomcat 是 Spring Boot 中默认的嵌入式服务器，它使用**每个请求一个线程**模型：

![image-20250207103146514](/image-20250207103146514.png)

1.  客户端发送 HTTP 请求。
2.  Tomcat 从其线程池中分配一个线程来处理请求。
3.  线程处理请求，可能会阻塞在 I/O 操作（数据库、网络调用）上。
4.  线程返回 HTTP 响应。
5.  线程返回到池中。
6.  DispatcherServlet 组件提供请求处理的中心入口点。
7.  DispatcherServlet 向 HandlerMapping 组件请求控制器。
8. 控制器处理请求。
9. 控制器将响应返回给 DispatcherServlet, 后者最终回复给客户端一个 HTTP 响应。

**阻塞和并发：** 这个模型是*同步和阻塞*的。Tomcat 池中的线程数限制了并发请求。对于高需求应用程序，请考虑响应式编程模型（在后面的章节中介绍）。

### 3.2.3 配置嵌入式 Tomcat

通过 `src/main/resources` 中的 `application.yml`（或 `application.properties`）中的属性自定义 Tomcat 的行为。

```yaml
# src/main/resources/application.yml
server:
  port: 9001  # 自定义端口
  tomcat:
    connection-timeout: 2s  # 连接超时
    keep-alive-timeout: 15s # Keep-alive 超时
    threads:
      max: 50     # 最大线程数
      min-spare: 5 # 最小线程数
```

**关键配置选项：**

*   `server.port`: 应用程序侦听的端口。
*   `server.tomcat.connection-timeout`: 建立连接的超时时间。
*   `server.tomcat.keep-alive-timeout`: 保持连接打开的时间。
*   `server.tomcat.threads.max`: 最大工作线程数。
*   `server.tomcat.threads.min-spare`: 最小工作线程数（始终运行）。

## 3.3 使用 Spring MVC 构建 RESTful 应用程序

我们将为目录服务构建一个 RESTful API，遵循**API 优先**模式：在实现业务逻辑*之前*定义 API。

### 3.3.1 REST API 优先，业务逻辑在后

**用例：**

*   查看图书列表。
*   按 ISBN 搜索图书。
*   添加新书。
*   编辑现有图书。
*   移除图书。

**API 规范 (表 3.1)：**

| 端点            | HTTP 方法 | 请求体 | 状态码 | 响应体   | 描述                         |
| --------------- | --------- | ------ | ------ | -------- | ---------------------------- |
| `/books`        | GET       |        | 200    | `Book[]` | 获取所有图书。               |
| `/books`        | POST      | `Book` | 201    | `Book`   | 添加新书。                   |
| `/books`        | POST      | `Book` | 422    |          | 具有相同 ISBN 的图书已存在。 |
| `/books/{isbn}` | GET       |        | 200    | `Book`   | 按 ISBN 获取图书。           |
| `/books/{isbn}` | GET       |        | 404    |          | 未找到具有给定 ISBN 的图书。 |
| `/books/{isbn}` | PUT       | `Book` | 200    | `Book`   | 按 ISBN 更新图书。           |
| `/books/{isbn}` | PUT       | `Book` | 201    | `Book`   | 创建图书如果图书还不存在     |
| `/books/{isbn}` | DELETE    |        | 204    |          | 按 ISBN 删除图书。           |

**核心概念：**

*   **实体：** 表示领域中的名词（例如，`Book`）。
*   **服务：** 定义用例（例如，`BookService`）。
*   **存储库：** 数据访问的抽象（例如，`BookRepository`）。

**领域实体 (`Book.java`):**

```java
// com.polarbookshop.catalogservice.domain.Book.java
package com.polarbookshop.catalogservice.domain;

import javax.validation.constraints.NotBlank;
import javax.validation.constraints.NotNull;
import javax.validation.constraints.Pattern;
import javax.validation.constraints.Positive;

public record Book(
        @NotBlank(message = "必须定义图书 ISBN。")
        @Pattern(regexp = "^([0-9]{10}|[0-9]{13})$", message = "ISBN 格式必须有效。")
        String isbn,

        @NotBlank(message = "必须定义图书标题。")
        String title,

        @NotBlank(message = "必须定义图书作者。")
        String author,

        @NotNull(message = "必须定义图书价格。")
        @Positive(message = "图书价格必须大于零。")
        Double price
) {}
```

*   使用 Java `record` 实现不变性。
*   包含 Bean Validation 注解。

**服务 (`BookService.java`):**

```java
// com.polarbookshop.catalogservice.domain.BookService.java
package com.polarbookshop.catalogservice.domain;

import org.springframework.stereotype.Service;

@Service
public class BookService {

    private final BookRepository bookRepository;

    public BookService(BookRepository bookRepository) {
        this.bookRepository = bookRepository;
    }

    public Iterable<Book> viewBookList() {
        return bookRepository.findAll();
    }

    public Book viewBookDetails(String isbn) {
        return bookRepository.findByIsbn(isbn)
                .orElseThrow(() -> new BookNotFoundException(isbn));
    }

    public Book addBookToCatalog(Book book) {
        if (bookRepository.existsByIsbn(book.isbn())) {
            throw new BookAlreadyExistsException(book.isbn());
        }
        return bookRepository.save(book);
    }
    public void removeBookFromCatalog(String isbn) {
        bookRepository.deleteByIsbn(isbn);
    }
    public Book editBookDetails(String isbn, Book book) {
        return bookRepository.findByIsbn(isbn)
            .map(existingBook -> {
                var bookToUpdate = new Book(
                    existingBook.isbn(),
                    book.title(),
                    book.author(),
                    book.price());
                return bookRepository.save(bookToUpdate);
           })
            .orElseGet(() -> addBookToCatalog(book));
    }
}
```

*   使用基于构造函数的依赖注入。
*   抛出自定义异常 (`BookNotFoundException`, `BookAlreadyExistsException`)。

**存储库接口 (`BookRepository.java`):**

```java
// com.polarbookshop.catalogservice.domain.BookRepository.java
package com.polarbookshop.catalogservice.domain;

import java.util.Optional;

public interface BookRepository {
    Iterable<Book> findAll();
    Optional<Book> findByIsbn(String isbn);
    boolean existsByIsbn(String isbn);
    Book save(Book book);
    void deleteByIsbn(String isbn);
}
```

**内存存储库实现 (`InMemoryBookRepository.java`):**

```java
// com.polarbookshop.catalogservice.persistence.InMemoryBookRepository.java
package com.polarbookshop.catalogservice.persistence;

import com.polarbookshop.catalogservice.domain.Book;
import com.polarbookshop.catalogservice.domain.BookRepository;
import org.springframework.stereotype.Repository;

import java.util.Map;
import java.util.Optional;
import java.util.concurrent.ConcurrentHashMap;

@Repository
public class InMemoryBookRepository implements BookRepository {

    private static final Map<String, Book> books = new ConcurrentHashMap<>();

    @Override
    public Iterable<Book> findAll() {
        return books.values();
    }

    @Override
    public Optional<Book> findByIsbn(String isbn) {
        return existsByIsbn(isbn) ? Optional.of(books.get(isbn)) : Optional.empty();
    }

    @Override
    public boolean existsByIsbn(String isbn) {
        return books.get(isbn) != null;
    }

    @Override
    public Book save(Book book) {
        books.put(book.isbn(), book);
        return book;
    }

    @Override
    public void deleteByIsbn(String isbn) {
        books.remove(isbn);
    }
}
```

*   使用 `ConcurrentHashMap` 进行内存存储（在本例中为简单起见）。在实际应用程序中，您将使用数据库。
*   使用 `@Repository` 构造型注解

**异常：**

```java
// com.polarbookshop.catalogservice.domain.BookAlreadyExistsException.java
package com.polarbookshop.catalogservice.domain;

public class BookAlreadyExistsException extends RuntimeException {
    public BookAlreadyExistsException(String isbn) {
        super("ISBN 为 " + isbn + " 的图书已存在。");
    }
}
```

```java
//com.polarbookshop.catalogservice.domain.BookNotFoundException.java
package com.polarbookshop.catalogservice.domain;

public class BookNotFoundException extends RuntimeException {
    public BookNotFoundException(String isbn) {
        super("未找到 ISBN 为 " + isbn + " 的图书。");
    }
}
```

### 3.3.2 使用 Spring MVC 实现 REST API

使用 `@RestController` 类来处理传入的 HTTP 请求。

```java
// com.polarbookshop.catalogservice.web.BookController.java
package com.polarbookshop.catalogservice.web;

import com.polarbookshop.catalogservice.domain.Book;
import com.polarbookshop.catalogservice.domain.BookService;
import org.springframework.http.HttpStatus;
import org.springframework.web.bind.annotation.*;
import javax.validation.Valid;

@RestController
@RequestMapping("books")
public class BookController {

    private final BookService bookService;

    public BookController(BookService bookService) {
        this.bookService = bookService;
    }

    @GetMapping
    public Iterable<Book> get() {
        return bookService.viewBookList();
    }

    @GetMapping("{isbn}")
    public Book getByIsbn(@PathVariable String isbn) {
        return bookService.viewBookDetails(isbn);
    }

    @PostMapping
    @ResponseStatus(HttpStatus.CREATED)
    public Book post(@Valid @RequestBody Book book) {
        return bookService.addBookToCatalog(book);
    }

    @DeleteMapping("{isbn}")
    @ResponseStatus(HttpStatus.NO_CONTENT)
    public void delete(@PathVariable String isbn) {
        bookService.removeBookFromCatalog(isbn);
    }

    @PutMapping("{isbn}")
    public Book put(@PathVariable String isbn, @Valid @RequestBody Book book) {
        return bookService.editBookDetails(isbn, book);
    }
}
```

*   `@RestController`: 将类标记为 REST 控制器。
*   `@RequestMapping("books")`: 将对 `/books` 的请求映射到此控制器。
*   `@GetMapping`, `@PostMapping`, `@PutMapping`, `@DeleteMapping`: 将 HTTP 方法映射到处理程序方法。
*   `@PathVariable`: 从 URI 中提取值（例如，ISBN）。
*   `@RequestBody`: 将请求体绑定到 Java 对象（例如，`Book`）。
*   `@Valid`: 启用 Bean Validation。
*   `@ResponseStatus`: 设置HTTP状态码

### 3.3.3 数据校验和错误处理

使用 Java Bean Validation 来校验传入的数据, 使用 `@Valid` 注解来触发校验。

**集中式异常处理 (`@RestControllerAdvice`):**

```java
// com.polarbookshop.catalogservice.web.BookControllerAdvice.java
package com.polarbookshop.catalogservice.web;

import com.polarbookshop.catalogservice.domain.BookAlreadyExistsException;
import com.polarbookshop.catalogservice.domain.BookNotFoundException;
import org.springframework.http.HttpStatus;
import org.springframework.validation.FieldError;
import org.springframework.web.bind.MethodArgumentNotValidException;
import org.springframework.web.bind.annotation.ExceptionHandler;
import org.springframework.web.bind.annotation.ResponseStatus;
import org.springframework.web.bind.annotation.RestControllerAdvice;

import java.util.HashMap;
import java.util.Map;

@RestControllerAdvice
public class BookControllerAdvice {

    @ExceptionHandler(BookNotFoundException.class)
    @ResponseStatus(HttpStatus.NOT_FOUND)
    String bookNotFoundHandler(BookNotFoundException ex) {
        return ex.getMessage();
    }

    @ExceptionHandler(BookAlreadyExistsException.class)
    @ResponseStatus(HttpStatus.UNPROCESSABLE_ENTITY)
    String bookAlreadyExistsHandler(BookAlreadyExistsException ex) {
        return ex.getMessage();
    }

    @ExceptionHandler(MethodArgumentNotValidException.class)
    @ResponseStatus(HttpStatus.BAD_REQUEST)
    public Map<String, String> handleValidationExceptions(MethodArgumentNotValidException ex) {
        var errors = new HashMap<String, String>();
        ex.getBindingResult().getAllErrors().forEach(error -> {
            String fieldName = ((FieldError) error).getField();
            String errorMessage = error.getDefaultMessage();
            errors.put(fieldName, errorMessage);
        });
        return errors;
    }
}
```

*   `@RestControllerAdvice`: 集中处理 REST 控制器的异常。
*   `@ExceptionHandler`: 将特定异常映射到处理程序方法。
*   处理 `BookNotFoundException`、`BookAlreadyExistsException` 和 `MethodArgumentNotValidException`。
*   返回适当的 HTTP 状态码 (404, 422, 400) 和错误消息。

### 3.3.4 为未来需求演进API
* 对API进行向后兼容的更改
* 当需要重大更改时，请使用API版本控制。
* 考虑使用REST架构的超媒体方面，也称为HATEOAS

## 3.4 使用 Spring 测试 RESTful 应用程序

自动化测试对于云原生开发至关重要。

### 3.4.1 使用 JUnit 5 进行单元测试

单元测试隔离验证各个组件。它们*不需要* Spring 上下文。

```java
//src/test/java/com/polarbookshop/catalogservice/domain/BookValidationTests.java
package com.polarbookshop.catalogservice.domain;

import org.junit.jupiter.api.BeforeAll;
import org.junit.jupiter.api.Test;

import javax.validation.ConstraintViolation;
import javax.validation.Validation;
import javax.validation.Validator;
import javax.validation.ValidatorFactory;
import java.util.Set;

import static org.assertj.core.api.Assertions.assertThat;

class BookValidationTests {

    private static Validator validator;

    @BeforeAll
    static void setUp() {
        ValidatorFactory factory = Validation.buildDefaultValidatorFactory();
        validator = factory.getValidator();
    }

    @Test
    void whenAllFieldsCorrectThenValidationSucceeds() {
        var book = new Book("1234567890", "Title", "Author", 9.90);
        Set<ConstraintViolation<Book>> violations = validator.validate(book);
        assertThat(violations).isEmpty();
    }

    @Test
    void whenIsbnDefinedButIncorrectThenValidationFails() {
        var book = new Book("a234567890", "Title", "Author", 9.90);
        Set<ConstraintViolation<Book>> violations = validator.validate(book);
        assertThat(violations).hasSize(1);
        assertThat(violations.iterator().next().getMessage())
                .isEqualTo("ISBN 格式必须有效。");
    }
}
```

*   使用 JUnit 5、Mockito 和 AssertJ（默认包含在 `spring-boot-starter-test` 中）。
*   测试 `Book` 类上的 Bean Validation 约束。

### 3.4.2 使用 `@SpringBootTest` 进行集成测试

集成测试验证组件之间的交互。它们*确实需要* Spring 上下文。

```java
//src/test/java/com/polarbookshop/catalogservice/CatalogServiceApplicationTests.java
package com.polarbookshop.catalogservice;

import com.polarbookshop.catalogservice.domain.Book;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.test.web.reactive.server.WebTestClient;

import static org.assertj.core.api.Assertions.assertThat;

@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
class CatalogServiceApplicationTests {

	@Autowired
	private WebTestClient webTestClient;

	@Test
	void whenPostRequestThenBookCreated() {
		var expectedBook = new Book("1231231231", "Title", "Author", 9.90);

		webTestClient
				.post()
				.uri("/books")
				.bodyValue(expectedBook)
				.exchange()
				.expectStatus().isCreated()
				.expectBody(Book.class).value(actualBook -> {
					assertThat(actualBook).isNotNull();
					assertThat(actualBook.isbn()).isEqualTo(expectedBook.isbn());
				});
	}
}
```

*   `@SpringBootTest`: 引导 Spring 应用程序上下文。
*   `webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT`: 在随机端口上启动嵌入式服务器。
*   `WebTestClient`: 用于向应用程序发出 REST 调用（需要 `spring-boot-starter-webflux`）。比 `MockMvc` 和 `TestRestTemplate` 更现代、更灵活。

### 3.4.3 使用 `@WebMvcTest` 测试 REST 控制器

`@WebMvcTest` 专注于测试 Spring MVC 层，*无需*启动完整的服务器。

```java
// src/test/java/com/polarbookshop/catalogservice/web/BookControllerMvcTests.java

package com.polarbookshop.catalogservice.web;

import com.polarbookshop.catalogservice.domain.BookNotFoundException;
import com.polarbookshop.catalogservice.domain.BookService;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.web.servlet.WebMvcTest;
import org.springframework.boot.test.mock.mockito.MockBean;
import org.springframework.test.web.servlet.MockMvc;

import static org.mockito.BDDMockito.given;
import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.get;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.status;

@WebMvcTest(BookController.class)
class BookControllerMvcTests {

    @Autowired
    private MockMvc mockMvc;

    @MockBean
    private BookService bookService;

    @Test
    void whenGetBookNotExistingThenShouldReturn404() throws Exception {
        String isbn = "73737313940";
        given(bookService.viewBookDetails(isbn)).willThrow(BookNotFoundException.class);
        mockMvc
                .perform(get("/books/" + isbn))
                .andExpect(status().isNotFound());
    }
}
```

*   `@WebMvcTest(BookController.class)`: 专注于 `BookController`。
*   `@MockBean`: 为 `BookService` 创建一个 mock 并将其添加到 Spring 上下文中。
*   `MockMvc`: 用于发出模拟 HTTP 请求。
### 3.4.4 使用@JsonTest测试JSON序列化
`@JsonTest`: 关注JSON序列化和反序列化。

```java
//src/test/java/com/polarbookshop/catalogservice/web/BookJsonTests.java
package com.polarbookshop.catalogservice.web;

import com.polarbookshop.catalogservice.domain.Book;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.json.JsonTest;
import org.springframework.boot.test.json.JacksonTester;

import static org.assertj.core.api.Assertions.assertThat;

@JsonTest
class BookJsonTests {

    @Autowired
    private JacksonTester<Book> json;

    @Test
    void testSerialize() throws Exception {
        var book = new Book("1234567890", "Title", "Author", 9.90);
        var jsonContent = json.write(book);
        assertThat(jsonContent).extractingJsonPathStringValue("@.isbn")
                .isEqualTo(book.isbn());
        assertThat(jsonContent).extractingJsonPathStringValue("@.title")
                .isEqualTo(book.title());
        assertThat(jsonContent).extractingJsonPathStringValue("@.author")
                .isEqualTo(book.author());
        assertThat(jsonContent).extractingJsonPathNumberValue("@.price")
                .isEqualTo(book.price());
    }
    @Test
    void testDeserialize() throws Exception {
        var content = """
                {
                    "isbn": "1234567890",
                    "title": "Title",
                    "author": "Author",
                    "price": 9.90
                }
                """;
        assertThat(json.parse(content))
                .usingRecursiveComparison()
                .isEqualTo(new Book("1234567890", "Title", "Author", 9.90));
    }
}
```
*   `@JsonTest`: 专注于 Book。
*    `JacksonTester`: 用于检查 JSON 映射。

## 3.5 部署流水线：构建和测试

部署流水线自动化了从代码提交到可发布软件的过程。我们将使用 GitHub Actions 创建一个基本的流水线。

### 3.5.1 了解部署流水线的提交阶段

**提交阶段**是流水线的第一部分。它通常包括：

1.  检出源代码。
2.  漏洞扫描。
3.  构建应用程序。
4.  运行单元测试。
5.  运行集成测试。

![image-20250207103324252](/image-20250207103324252.png)

### 3.5.2 使用 GitHub Actions 实现提交阶段

GitHub Actions 是一个与 GitHub 集成的 CI/CD 平台。工作流在 `.github/workflows` 目录中的 YAML 文件中定义。

**.github/workflows/build-and-test.yml:**

```yaml
name: Commit Stage
on: push
jobs:
  build:
    name: Build and Test
    runs-on: ubuntu-22.04
    permissions:
      contents: read
      security-events: write
    steps:
      - name: Checkout source code
        uses: actions/checkout@v3

      - name: Set up JDK
        uses: actions/setup-java@v3
        with:
          distribution: temurin
          java-version: 17
          cache: gradle

      - name: Code vulnerability scanning
        uses: anchore/scan-action@v3
        id: scan
        with:
          path: "${{ github.workspace }}"
          fail-build: false  # 在实际项目中应设置为 true
          severity-cutoff: high
          acs-report-enable: true

      - name: Upload vulnerability report
        uses: github/codeql-action/upload-sarif@v2
        if: success() || failure() # 即使扫描失败也上传
        with:
          sarif_file: ${{ steps.scan.outputs.sarif }}

      - name: Build, unit tests and integration tests
        run: |
          chmod +x gradlew
          ./gradlew build
```

*   **`name`:** 工作流的名称。
*   **`on: push`:** 在每次推送到存储库时触发工作流。
*   **`jobs`:** 定义要运行的作业。
*   **`runs-on`:** 指定运行器环境 (Ubuntu 22.04)。
*   **`permissions`:** 设置权限。
*   **`steps`:** 要执行的步骤序列。
*   **`uses`:** 使用预构建的操作（例如，`actions/checkout@v3`、`actions/setup-java@v3`、`anchore/scan-action@v3`）。
*   **漏洞扫描 (grype):**
    *   使用 `anchore/scan-action@v3` 操作（由 Grype 提供支持）。
    *   `fail-build: false`: **重要提示：** 在实际项目中，您应该将其设置为 `true`，以便在发现漏洞时使构建失败。
    *   将 SARIF 报告上传到 GitHub 的安全选项卡。
*   **构建和测试：**
    *   使用`chmod +x gradlew` 确保Gradle Wrapper可执行，并运行 `./gradlew build`, 该命令编译代码，运行单元和集成测试。

本教程为使用 Spring Boot 构建云原生应用程序提供了坚实的基础。它强调了 API 优先设计、依赖管理、嵌入式服务器、全面测试以及 CI/CD 流水线的起步等最佳实践。后续章节将扩展这些概念，包括外部化配置、数据持久化和更高级的部署策略。