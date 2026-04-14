FROM maven:3.9.6-eclipse-temurin-17 AS build

WORKDIR /app


COPY ./Docker-1 .

RUN mvn package

FROM eclipse-temurin:17-jre-alpine

WORKDIR /app

COPY --from=build /app/target/*.jar ./app_using_multi_stage.jar

EXPOSE 8080

CMD ["java", "-jar", "app_using_multi_stage.jar"]
