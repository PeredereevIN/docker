FROM ubuntu:24.04

ENV DEBIAN_FRONTEND=noninteractive

# Установка Java 17 и необходимых пакетов
RUN apt-get update && apt-get install -y \
    openjdk-17-jdk \
    wget \
    sudo \
    python3 \
    python3-pip \
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/*

# Переменные окружения Java
ENV JAVA_HOME=/usr/lib/jvm/java-17-openjdk-amd64
ENV PATH=$PATH:$JAVA_HOME/bin

# Версия Spark (можно изменить)
ENV SPARK_VERSION=4.1.1
ENV HADOOP_VERSION=3
ENV SPARK_HOME=/opt/spark
ENV PATH=$PATH:$SPARK_HOME/bin:$SPARK_HOME/sbin

# Копируем локальный архив Spark (положите его рядом с Dockerfile)
# Если архива нет, раскомментируйте строку с wget и закомментируйте COPY
COPY spark-${SPARK_VERSION}-bin-hadoop${HADOOP_VERSION}.tgz /tmp/spark.tgz

# Распаковка Spark
RUN tar -xzf /tmp/spark.tgz -C /opt/ && rm /tmp/spark.tgz && mv /opt/spark-${SPARK_VERSION}-bin-hadoop${HADOOP_VERSION} $SPARK_HOME

# (Опционально) Установка Python-пакетов, если нужны
# RUN pip3 install numpy pandas

# --- СОЗДАНИЕ КОНФИГУРАЦИОННЫХ ФАЙЛОВ (если нужно) ---
# Spark не требует сложной конфигурации, но можно задать параметры через spark-defaults.conf
RUN cat > $SPARK_HOME/conf/spark-defaults.conf <<EOF
spark.master                     spark://localhost:7077
spark.eventLog.enabled           true
spark.eventLog.dir                /tmp/spark-events
spark.history.fs.logDirectory     /tmp/spark-events
EOF

# Создаем директорию для логов событий
RUN mkdir -p /tmp/spark-events

# --- СОЗДАНИЕ ENTRYPOINT.SH ---
# Скрипт запускает Spark в standalone режиме (master + worker в одном контейнере)
RUN echo '#!/bin/bash\n\
set -e\n\
\n\
# Функция для запуска master\n\
start_master() {\n\
    echo "Запуск Spark Master..."\n\
    $SPARK_HOME/sbin/start-master.sh\n\
}\n\
\n\
# Функция для запуска worker\n\
start_worker() {\n\
    echo "Запуск Spark Worker..."\n\
    $SPARK_HOME/sbin/start-worker.sh spark://localhost:7077\n\
}\n\
\n\
# Определяем режим запуска по переменной окружения SPARK_MODE\n\
# По умолчанию запускаем и master, и worker (standalone кластер)\n\
if [ "$SPARK_MODE" = "master" ]; then\n\
    start_master\n\
elif [ "$SPARK_MODE" = "worker" ]; then\n\
    if [ -z "$SPARK_MASTER_URL" ]; then\n\
        echo "Ошибка: для worker необходимо указать SPARK_MASTER_URL"\n\
        exit 1\n\
    fi\n\
    $SPARK_HOME/sbin/start-worker.sh $SPARK_MASTER_URL\n\
else\n\
    # Режим "all" (по умолчанию)\n\
    start_master\n\
    sleep 3\n\
    start_worker\n\
fi\n\
\n\
# Держим контейнер активным\n\
tail -f $SPARK_HOME/logs/*' > /entrypoint.sh && chmod +x /entrypoint.sh

# Открываем порты веб-интерфейсов Spark
EXPOSE 8080 8081 7077 4040

ENTRYPOINT ["/entrypoint.sh"]
