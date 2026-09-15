# build
FROM maven:3.9.9-eclipse-temurin-17 AS builder

ARG VERSION=1.0.0
ENV VERSION=${VERSION}

WORKDIR /usr/src/app

COPY ./src ./src
COPY *.xml ./

RUN mvn package -DskipTests

# release
FROM eclipse-temurin:17-jre-jammy

ARG VERSION=1.0.0
ENV VERSION=${VERSION}

WORKDIR /app

RUN groupadd -r jaruser \
    && useradd -r -g jaruser -d /app -s /sbin/nologin jaruser \
    && mkdir -p /logs/reports \
    && chown -R jaruser:jaruser /logs /app \
    && apt-get update && apt-get install -y --no-install-recommends dumb-init curl \
    && rm -rf /var/lib/apt/lists/*

COPY --chown=jaruser:jaruser \
    --from=builder /usr/src/app/target/sausage-store-${VERSION}.jar ./sausage-store.jar

EXPOSE 8080

USER jaruser

ENTRYPOINT ["dumb-init", "java", "-jar", "./sausage-store.jar"]