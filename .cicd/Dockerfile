# java based application
FROM openjdk:18

RUN mkdir -p /opt/i27

WORKDIR /opt/i27

ARG JAR_SOURCE

COPY ["${JAR_SOURCE}", "/opt/i27/i27-user.jar"]

EXPOSE 8232

CMD ["java", "-jar", "/opt/i27/i27-user.jar"]