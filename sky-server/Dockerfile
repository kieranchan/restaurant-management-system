# 使用一个轻量级的 Java 8 运行时作为基础镜像
FROM openjdk:8-jre-slim

# 安装 curl 以用于健康检查
RUN apt-get update && apt-get install -y curl && rm -rf /var/lib/apt/lists/*

# 设置工作目录
WORKDIR /app

# 将构建好的 Spring Boot JAR 包从构建上下文复制到镜像中
# 注意：这里的 JAR 包名称需要和 Maven 构建产生的名称完全一致
COPY sky-server/target/sky-server-1.0-SNAPSHOT.jar .

# 暴露应用程序运行的端口
EXPOSE 8080

# 容器启动时执行的命令
# 通过 Java 系统属性（-D）来传递环境变量，Spring Boot 会自动加载它们
# 这样可以灵活地在 docker run 或 docker-compose.yml 中配置数据库、Redis等信息
ENTRYPOINT ["java", \
            "-Dsky.datasource.host=${SKY_DATASOURCE_HOST}", \
            "-Dsky.datasource.port=${SKY_DATASOURCE_PORT}", \
            "-Dsky.datasource.database=${SKY_DATASOURCE_DATABASE}", \
            "-Dsky.datasource.username=${SKY_DATASOURCE_USERNAME}", \
            "-Dsky.datasource.password=${SKY_DATASOURCE_PASSWORD}", \
            "-Dspring.redis.host=${SPRING_REDIS_HOST}", \
            "-Dspring.redis.port=${SPRING_REDIS_PORT}", \
            "-Dspring.redis.password=${SPRING_REDIS_PASSWORD}", \
            "-Dspring.redis.database=${SPRING_REDIS_DATABASE}", \
            "-Dsky.wechat.appid=${SKY_WECHAT_APPID}", \
            "-Dsky.wechat.secret=${SKY_WECHAT_SECRET}", \
            "-Dsky.wechat.mchid=${SKY_WECHAT_MCHID}", \
            "-Dsky.wechat.mch-serial-no=${SKY_WECHAT_MCH_SERIAL_NO}", \
            "-Dsky.wechat.private-key-file-path=${SKY_WECHAT_PRIVATE_KEY_FILE_PATH}", \
            "-Dsky.wechat.api-v3-key=${SKY_WECHAT_API_V3_KEY}", \
            "-Dsky.wechat.we-chat-pay-cert-file-path=${SKY_WECHAT_WE_CHAT_PAY_CERT_FILE_PATH}", \
            "-Dsky.wechat.notify-url=${SKY_WECHAT_NOTIFY_URL}", \
            "-Dsky.wechat.refund-notify-url=${SKY_WECHAT_REFUND_NOTIFY_URL}", \
            "-Dcos.secret-id=${COS_SECRET_ID}", \
            "-Dcos.secret-key=${COS_SECRET_KEY}", \
            "-Dcos.region=${COS_REGION}", \
            "-Dcos.bucket-name=${COS_BUCKET_NAME}", \
            "-jar", "sky-server-1.0-SNAPSHOT.jar"]