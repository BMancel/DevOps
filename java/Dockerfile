FROM openjdk:17
COPY Main.java /app/Main.java
WORKDIR /app
RUN javac Main.java

FROM eclipse-temurin:17-jre-alpine
COPY --from=0 /app/Main.class .

CMD ["java", "Main"]


# Build
FROM maven:3.8.6-amazoncorretto-17 AS myapp-build
ENV MYAPP_HOME /opt/myapp
WORKDIR $MYAPP_HOME
COPY pom.xml .
COPY src ./src
RUN mvn package -DskipTests

# Run
FROM amazoncorretto:17
ENV MYAPP_HOME /opt/myapp
WORKDIR $MYAPP_HOME
COPY --from=myapp-build $MYAPP_HOME/target/*.jar $MYAPP_HOME/myapp.jar

ENTRYPOINT java -jar myapp.jar