FROM eclipse-temurin:18-jdk-alpine
ARG JAR_FILE=*jar
COPY ${JAR_FILE} app.jar
EXPOSE 8080
ENTRYPOINT ["java","-jar","/app.jar"]