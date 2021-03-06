#阶段一 修改mysql数据库连接版本 并构建

FROM alpine AS build

ARG NACOS_VERSION=1.3.2
ARG SPRING_BOOT_VERSION=2.3.4
ARG MYSQL_VERSION=8.0.21
ARG MINA_IO_VERSION=2.1.4
ARG MYSQL_OLD_CLASS=com.mysql.jdbc.jdbc2.optional.MysqlDataSource
ARG MYSQL_NEW_CLASS=com.mysql.cj.jdbc.MysqlDataSource

RUN apk add --no-cache wget openjdk8 maven tar
ENV JAVA_HOME=/usr/lib/jvm/java-1.8-openjdk
ENV CLASSPATH=.:$JAVA_HOME/lib
ENV PATH=$PATH:$JAVA_HOME/bin

WORKDIR /tmp
RUN  wget https://github.com/alibaba/nacos/archive/${NACOS_VERSION}.tar.gz ;\
        tar -xzf ${NACOS_VERSION}.tar.gz

RUN cd nacos-${NACOS_VERSION}; \
        sed -i "$(sed -n -e '/<spring-boot-dependencies.version>/=' pom.xml)s/<spring-boot-dependencies.version>.*/<spring-boot-dependencies.version>${SPRING_BOOT_VERSION}.RELEASE<\/spring-boot-dependencies.version>/g" pom.xml;\
        sed -i "$(sed -n -e '/<mysql-connector-java.version>/=' pom.xml)s/<mysql-connector-java.version>.*/<mysql-connector-java.version>${MYSQL_VERSION}<\/mysql-connector-java.version>/g" pom.xml;\
        sed -i "$(sed -n -e '/<mina-core.version>/=' pom.xml)s/<mina-core.version>.*/<mina-core.version>${MINA_IO_VERSION}<\/mina-core.version>/g" pom.xml;\
        sed -i "s/${MYSQL_OLD_CLASS}/${MYSQL_NEW_CLASS}/g" naming/src/main/java/com/alibaba/nacos/naming/healthcheck/MysqlHealthCheckProcessor.java;\
        mvn -Prelease-nacos -Dmaven.test.skip=true clean install -U

#阶段二 拉取nacos官方镜像
FROM nacos/nacos-server:1.3.2 AS standard

#RUN sed -i '$c $JAVA ${JAVA_OPT}' bin/docker-startup.sh

#阶段三 重新发布
FROM  justtin/alpine-openjdk8-jre AS release

ARG NACOS_VERSION=1.3.2


ENV MODE="cluster" \
        PREFER_HOST_MODE="ip"\
        BASE_DIR="/home/nacos" \
        CLASSPATH=".:/home/nacos/conf:$CLASSPATH" \
        CLUSTER_CONF="/home/nacos/conf/cluster.conf" \
        FUNCTION_MODE="all" \
        NACOS_USER="nacos" \
        JVM_XMS="384m" \
        JVM_XMX="1g" \
        JVM_MS="128m" \
        JVM_MMS="320m" \
        JVM_XMN="256m" \
        NACOS_DEBUG="n" \
        JAVA="java" \
        TOMCAT_ACCESSLOG_ENABLED="false" \
        LANG=zh_CN.UTF-8 \
        JAVA_OPT_EXT="--server.tomcat.basedir=/home/nacos/logs"

WORKDIR /$BASE_DIR

COPY --from=standard $BASE_DIR/bin/ bin/
COPY --from=standard $BASE_DIR/conf/ conf/
COPY --from=standard $BASE_DIR/init.d/ init.d/
COPY --from=standard $BASE_DIR/logs/ logs/
COPY --from=build /tmp/nacos-${NACOS_VERSION}/distribution/target/nacos-server-${NACOS_VERSION}/nacos/target/nacos-server.jar target/

RUN  apk add --no-cache dos2unix \
	&& dos2unix bin/docker-startup.sh conf/* \
	&& chmod +x bin/docker-startup.sh \
	&& apk del dos2unix \
    mkdir -p logs; \
	touch logs/start.out; \
	ln -sf /dev/stdout logs/start.out; \
	ln -sf /dev/stderr logs/start.out

EXPOSE 8848
ENTRYPOINT ["bash", "bin/docker-startup.sh"]
