# Shared service infrastructure

This Compose project provides the shared development dependencies for Notebook LM services:

- **Kafka** (KRaft, no ZooKeeper)
- **Kafka UI** (browser-based Kafka administration)
- **MinIO** (S3-compatible object storage)
- a stable Docker network: `notebook-lm`

## Quick start

```sh
cp .env.example .env
docker compose -f docker-compose.yml -f docker-compose.dev.yml up -d
docker compose -f docker-compose.yml -f docker-compose.dev.yml ps
```

Stop the shared development dependencies:

```sh
docker compose -f docker-compose.yml -f docker-compose.dev.yml down
```

Start the production-shaped stack (ports remain private to the Docker network):

```sh
docker compose -f docker-compose.yml -f docker-compose.prod.yml up -d
```

> Keep `.env` out of version control and replace the sample MinIO credentials before any non-local deployment.

## Endpoints

| Dependency | From another container | From your host (development) |
| --- | --- | --- |
| Kafka bootstrap server | `kafka:9092` | `localhost:9094` |
| Kafka UI | `http://kafka-ui:8080` | `http://localhost:8080` |
| MinIO S3 API | `http://minio:9000` | `http://localhost:9000` |
| MinIO Console | not normally needed | `http://localhost:9001` |

The development MinIO Console signs in with `MINIO_ROOT_USER` and `MINIO_ROOT_PASSWORD`. The configured `MINIO_BUCKET` is created by `minio-init`.

## Connect an application service

Attach every service Compose project to the existing network instead of declaring duplicate Kafka or MinIO containers:

```yaml
services:
  your-service:
    environment:
      KAFKA_BOOTSTRAP_SERVERS: kafka:9092
      MINIO_ENDPOINT: http://minio:9000
      MINIO_BUCKET: notebook-lm
    networks:
      - shared

networks:
  shared:
    external: true
    name: ${SHARED_NETWORK_NAME:-notebook-lm}
```


## Operational checks

Validate the combined development configuration:

```sh
docker compose -f docker-compose.yml -f docker-compose.dev.yml config
```

Check Kafka metadata and the MinIO bucket:

```sh
docker compose -f docker-compose.yml -f docker-compose.dev.yml exec kafka \
  /opt/kafka/bin/kafka-topics.sh --bootstrap-server localhost:9092 --list

docker compose -f docker-compose.yml -f docker-compose.dev.yml run --rm minio-init \
  mc ls local
```

Create application topics explicitly—the broker disables automatic topic creation by default:

```sh
docker compose -f docker-compose.yml -f docker-compose.dev.yml exec kafka \
  /opt/kafka/bin/kafka-topics.sh --bootstrap-server localhost:9092 \
  --create --if-not-exists --topic your.topic --partitions 1 --replication-factor 1
```
