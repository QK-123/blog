#### Spring Boot 项目镜像打包

##### 1. 打包编译并执行程序

```bash
./mvnw package && java -jar target/gs-spring-boot-docker-0.1.0.jar
```

2. ##### 使用用户做权限限制

```bash
FROM openjdk:8-jdk-alpine
RUN addgroup -S spring && adduser -S spring -G spring
USER spring:spring
ARG JAR_FILE=target/*.jar
COPY ${JAR_FILE} app.jar
ENTRYPOINT ["java","-jar","/app.jar"]
```

```bash
docker build -t springio/gs-spring-boot-docker .
```

##### 3. 将程序依赖和代码分层提高性能

```bash
FROM openjdk:8-jdk-alpine
RUN addgroup -S spring && adduser -S spring -G spring
USER spring:spring
ARG DEPENDENCY=target/dependency
COPY ${DEPENDENCY}/BOOT-INF/lib /app/lib # 依赖
COPY ${DEPENDENCY}/META-INF /app/META-INF
COPY ${DEPENDENCY}/BOOT-INF/classes /app
ENTRYPOINT ["java","-cp","app:app/lib/*","hello.Application"]
```

解压fatjar

```bash
mkdir -p target/dependency && (cd target/dependency; jar -xf ../*.jar)
```

##### 4. 使用maven插件构建docker 镜像

```bash
./mvnw spring-boot:build-image -Dspring-boot.build-image.imageName=springio/gs-spring-boot-docker
```

##### 5. 激活不同的环境

```bash
docker run -e "SPRING_PROFILES_ACTIVE=prod" -p 8080:8080 -t springio/gs-spring-boot-docker
```