FROM maven:3.9.8-eclipse-temurin-21 AS builder
WORKDIR /build
COPY . .
RUN mvn clean package -DskipTests

FROM eclipse-temurin:21-jre-alpine AS final
WORKDIR /opt/app
RUN addgroup --system javauser && adduser -S -s /usr/sbin/nologin -G javauser javauser
COPY --from=builder /build/target/smart-home-monolith-1.0-SNAPSHOT.jar app.jar
RUN chown -R javauser:javauser .
USER javauser
ENTRYPOINT ["java", "-jar", "app.jar"]