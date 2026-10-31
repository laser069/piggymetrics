FROM eclipse-temurin:8-jre
MAINTAINER Alexander Lukyanchikov <sqshq@sqshq.com>

ADD ./target/registry.jar /app/
ENV JAVA_OPTS="-Xmx200m"
CMD ["sh", "-c", "exec java $JAVA_OPTS -jar /app/registry.jar"]

EXPOSE 8761