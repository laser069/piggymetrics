FROM eclipse-temurin:8-jre
MAINTAINER Alexander Lukyanchikov <sqshq@sqshq.com>

ADD ./target/account-service.jar /app/
ENV JAVA_OPTS="-Xmx200m"
CMD ["sh", "-c", "exec java $JAVA_OPTS -jar /app/account-service.jar"]

EXPOSE 6000