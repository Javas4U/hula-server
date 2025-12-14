# 使用 JRE 21 基础镜像
FROM openjdk:21-jdk-slim

# 设置工作目录
WORKDIR /app

# 挂载临时目录
VOLUME /tmp

# 将构建好的 jar 包复制到镜像中
# 注意：在 docker-compose中，上下文是 ../../../luohuo-cloud/xxx/xxx-server，所以 target 在上下文中
COPY target/*.jar app.jar

# 设置时区
ENV TZ=Asia/Shanghai
RUN ln -snf /usr/share/zoneinfo/$TZ /etc/localtime && echo $TZ > /etc/timezone

# 暴露端口 (根据实际情况，这里仅作声明)
# EXPOSE 8080

# 启动命令
ENTRYPOINT ["sh", "-c", "java $JAVA_OPTS -jar app.jar"]
