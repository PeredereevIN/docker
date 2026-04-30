FROM ubuntu:24.04

ENV DEBIAN_FRONTEND=noninteractive

# Установка необходимых пакетов (Java 8, SSH, wget, sudo и др.)
RUN apt-get update && apt-get install -y \
    openjdk-8-jdk \
    openssh-server \
    wget \
    pdsh \
    net-tools \
    sudo \
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/*

# Переменные окружения (Java, Hadoop, и пользователи для root)
ENV JAVA_HOME=/usr/lib/jvm/java-8-openjdk-amd64
ENV PATH=$PATH:$JAVA_HOME/bin

ENV HADOOP_HOME=/opt/apache/hadoop-3.4.1
ENV PATH=$PATH:$HADOOP_HOME/bin:$HADOOP_HOME/sbin
ENV HADOOP_MAPRED_HOME=$HADOOP_HOME
ENV YARN_HOME=$HADOOP_HOME
ENV HADOOP_CONF_DIR=$HADOOP_HOME/etc/hadoop
ENV HADOOP_COMMON_LIB_NATIVE_DIR=$HADOOP_HOME/lib/native
ENV HADOOP_OPTS="-Djava.library.path=$HADOOP_HOME/lib/native"
ENV HADOOP_STREAMING=$HADOOP_HOME/share/hadoop/tools/lib/hadoop-streaming-3.4.1.jar
ENV HADOOP_LOG_DIR=$HADOOP_HOME/logs
ENV PDSH_RCMD_TYPE=ssh
ENV HDFS_NAMENODE_USER=root
ENV HDFS_DATANODE_USER=root
ENV HDFS_SECONDARYNAMENODE_USER=root
ENV YARN_RESOURCEMANAGER_USER=root
ENV YARN_NODEMANAGER_USER=root

# Создание директории для Hadoop
RUN mkdir -p /opt/apache

COPY hadoop-3.4.1.tar.gz /tmp/hadoop.tar.gz


RUN tar -xzf /tmp/hadoop.tar.gz -C /opt/apache/ && rm /tmp/hadoop.tar.gz


# Копируем JAVA_HOME в hadoop-env.sh
RUN echo "export JAVA_HOME=$JAVA_HOME" >> $HADOOP_CONF_DIR/hadoop-env.sh

# --- СОЗДАНИЕ КОНФИГУРАЦИОННЫХ ФАЙЛОВ ---

# core-site.xml
RUN cat > $HADOOP_CONF_DIR/core-site.xml <<EOF
<configuration>
    <property>
        <name>fs.defaultFS</name>
        <value>hdfs://localhost:9000</value>
    </property>
    <property>
        <name>hadoop.proxyuser.dataflair.groups</name>
        <value>*</value>
    </property>
    <property>
        <name>hadoop.proxyuser.dataflair.hosts</name>
        <value>*</value>
    </property>
    <property>
        <name>hadoop.proxyuser.server.hosts</name>
        <value>*</value>
    </property>
    <property>
        <name>hadoop.proxyuser.server.groups</name>
        <value>*</value>
    </property>
</configuration>
EOF

# hdfs-site.xml
RUN cat > $HADOOP_CONF_DIR/hdfs-site.xml <<EOF
<configuration>
    <property>
        <name>dfs.replication</name>
        <value>1</value>
    </property>
</configuration>
EOF

# mapred-site.xml
RUN cat > $HADOOP_CONF_DIR/mapred-site.xml <<EOF
<configuration>
    <property>
        <name>mapreduce.framework.name</name>
        <value>yarn</value>
    </property>
    <property>
        <name>mapreduce.application.classpath</name>
        <value>\$HADOOP_MAPRED_HOME/share/hadoop/mapreduce/*:\$HADOOP_MAPRED_HOME/share/hadoop/mapreduce/lib/*</value>
    </property>
</configuration>
EOF

# yarn-site.xml
RUN cat > $HADOOP_CONF_DIR/yarn-site.xml <<EOF
<configuration>
    <property>
        <name>yarn.nodemanager.aux-services</name>
        <value>mapreduce_shuffle</value>
    </property>
    <property>
        <name>yarn.nodemanager.env-whitelist</name>
        <value>JAVA_HOME,HADOOP_COMMON_HOME,HADOOP_HDFS_HOME,HADOOP_CONF_DIR,CLASSPATH_PREPEND_DISTCACHE,HADOOP_YARN_HOME,HADOOP_MAPRED_HOME</value>
    </property>
</configuration>
EOF
# --- КОНЕЦ БЛОКА СОЗДАНИЯ КОНФИГОВ ---

# Настройка SSH для беспарольного доступа root@localhost
RUN ssh-keygen -t rsa -P '' -f ~/.ssh/id_rsa \
    && cat ~/.ssh/id_rsa.pub >> ~/.ssh/authorized_keys \
    && chmod 700 ~/.ssh && chmod 600 ~/.ssh/authorized_keys \
    && sed -i 's/#PermitRootLogin prohibit-password/PermitRootLogin yes/' /etc/ssh/sshd_config \
    && sed -i 's/#PubkeyAuthentication yes/PubkeyAuthentication yes/' /etc/ssh/sshd_config \
    && sed -i 's/#PasswordAuthentication yes/PasswordAuthentication no/' /etc/ssh/sshd_config

# Создаем entrypoint.sh прямо здесь
RUN echo '#!/bin/bash\n\
set -e\n\
service ssh start\n\
NAMENODE_DIR="/tmp/hadoop-root/dfs/name"\n\
if [ ! -f "$NAMENODE_DIR/current/VERSION" ]; then\n\
    echo "Форматирование NameNode..."\n\
    echo "Y" | hdfs namenode -format\n\
else\n\
    echo "NameNode уже отформатирована."\n\
fi\n\
start-all.sh\n\
echo "Запущенные Java-процессы:"\n\
jps\n\
tail -f $HADOOP_HOME/logs/*' > /entrypoint.sh && chmod +x /entrypoint.sh

EXPOSE 9870 8088 22

ENTRYPOINT ["/entrypoint.sh"]
