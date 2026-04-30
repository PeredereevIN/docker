# syntax=docker/dockerfile:1
FROM ubuntu:24.04

ENV DEBIAN_FRONTEND=noninteractive

# Установка Java 17 (требование Kafka 4.x) и необходимых утилит
RUN apt-get update && apt-get install -y \
    openjdk-17-jre-headless \
    wget \
    sudo \
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/*

# Переменные окружения Java
ENV JAVA_HOME=/usr/lib/jvm/java-17-openjdk-amd64
ENV PATH=$PATH:$JAVA_HOME/bin

# Версии Kafka и Scala (для имени архива)
ENV KAFKA_VERSION=4.2.0
ENV SCALA_VERSION=2.13
ENV KAFKA_HOME=/opt/kafka
ENV PATH=$PATH:$KAFKA_HOME/bin

# Копируем локальный архив Kafka (положите его рядом с Dockerfile)
# Если архива нет, раскомментируйте строку с wget и закомментируйте COPY
COPY kafka_${SCALA_VERSION}-${KAFKA_VERSION}.tgz /tmp/kafka.tgz

# Распаковка Kafka
RUN tar -xzf /tmp/kafka.tgz -C /opt/ && rm /tmp/kafka.tgz && mv /opt/kafka_${SCALA_VERSION}-${KAFKA_VERSION} $KAFKA_HOME

# --- КОНФИГУРАЦИЯ ДЛЯ РЕЖИМА STANDALONE (KRaft) ---
# Генерируем UUID кластера и настраиваем server.properties
RUN KAFKA_CLUSTER_ID="$($KAFKA_HOME/bin/kafka-storage.sh random-uuid)" \
    && echo "cluster.id=${KAFKA_CLUSTER_ID}" >> $KAFKA_HOME/config/server.properties \
    && echo "node.id=1" >> $KAFKA_HOME/config/server.properties \
    && echo "process.roles=broker,controller" >> $KAFKA_HOME/config/server.properties \
    && echo "listeners=PLAINTEXT://0.0.0.0:9092,CONTROLLER://0.0.0.0:9093" >> $KAFKA_HOME/config/server.properties \
    && echo "advertised.listeners=PLAINTEXT://localhost:9092" >> $KAFKA_HOME/config/server.properties \
    && echo "log.dirs=/tmp/kraft-combined-logs" >> $KAFKA_HOME/config/server.properties

# Форматирование логов (выполняется при сборке) - standalone режим
RUN $KAFKA_HOME/bin/kafka-storage.sh format --standalone -t $(grep cluster.id $KAFKA_HOME/config/server.properties | cut -d'=' -f2) -c $KAFKA_HOME/config/server.properties

# --- СОЗДАНИЕ ENTRYPOINT.SH ---
RUN echo '#!/bin/bash\n\
set -e\n\
\n\
echo "Запуск Kafka (KRaft mode)..."\n\
$KAFKA_HOME/bin/kafka-server-start.sh $KAFKA_HOME/config/server.properties\n\
\n\
# Держим контейнер активным (на случай, если Kafka уйдёт в фон, но в режиме standalone он остаётся на переднем плане)\n\
# Эта строка не обязательна, так как kafka-server-start.sh не демонизируется\n\
wait' > /entrypoint.sh && chmod +x /entrypoint.sh

# Порт для клиентов (9092) и для контроллера (9093, нужен для кластера, но можно не пробрасывать наружу)
EXPOSE 9092 9093

ENTRYPOINT ["/entrypoint.sh"]
