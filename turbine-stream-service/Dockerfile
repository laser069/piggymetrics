FROM eclipse-temurin:8-jre
MAINTAINER Chi Dov <d.chiproeng@gmail.com>

ADD ./target/turbine-stream-service.jar /app/
ENV JAVA_OPTS="-Xmx200m"
CMD ["sh", "-c", "exec java $JAVA_OPTS -jar /app/turbine-stream-service.jar"]

EXPOSE 8989