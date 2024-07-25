#
# Licensed under the Apache License, Version 2.0 (the "License");
# you may not use this file except in compliance with the License.
# You may obtain a copy of the License at
#
#     http://www.apache.org/licenses/LICENSE-2.0
#
# Unless required by applicable law or agreed to in writing, software
# distributed under the License is distributed on an "AS IS" BASIS,
# WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
# See the License for the specific language governing permissions and
# limitations under the License.
#
FROM ghcr.io/airlift/jvmkill:latest AS jvmkill

FROM registry.access.redhat.com/ubi9/ubi-minimal:latest AS jdk-download
ARG JDK_DOWNLOAD_LINK
ARG JDK_VERSION
ENV JAVA_HOME="/usr/lib/jvm/jdk-${JDK_VERSION}"

RUN \
    set -xeuo pipefail && \
    microdnf install -y tar gzip && \
    # Install JDK from the provided archive link \
    echo "Downloading JDK from ${JDK_DOWNLOAD_LINK}" && \
    mkdir -p "${JAVA_HOME}" && \
    curl -#LfS "${JDK_DOWNLOAD_LINK}" | tar -zx --strip 1 -C "${JAVA_HOME}"

# Use ubi9 minimal as it's more secure
FROM registry.access.redhat.com/ubi9/ubi-minimal:latest
ARG JDK_VERSION
ENV JAVA_HOME="/usr/lib/jvm/jdk-${JDK_VERSION}"
ENV PATH=$PATH:$JAVA_HOME/bin
COPY --from=jdk-download $JAVA_HOME $JAVA_HOME

RUN \
    set -xeu && \
    microdnf update -y && \
    microdnf install -y tar less python3 shadow-utils && \
    update-alternatives --install /usr/bin/python python /usr/bin/python3 1 && \
    groupadd trino --gid 1000 && \
    useradd trino --uid 1000 --gid 1000 --create-home && \
    mkdir -p /usr/lib/trino /data/trino && \
    chown -R "trino:trino" /usr/lib/trino /data/trino

ARG TRINO_VERSION
COPY --chown=trino:trino trino-cli-${TRINO_VERSION}-executable.jar /usr/bin/trino
COPY --chown=trino:trino trino-server-${TRINO_VERSION} /usr/lib/trino
COPY --chown=trino:trino default/etc /etc/trino
COPY --chown=trino:trino --from=jvmkill /libjvmkill.so /usr/lib/trino/bin
COPY --chown=trino:trino mucfc/mysql-connector-java-8.0.29.jar /usr/lib/trino/lib/mysql-connector-java-8.0.29.jar
COPY --chown=trino:trino mucfc/limits.conf /etc/security/limits.conf

ARG JUICEFS_JAR_URL=https://s.juicefs.com/static/Linux/juicefs-hadoop-4.9.12.jar
ARG HADOOP_ALIYUN_JAR_URL=https://repo1.maven.org/maven2/org/apache/hadoop/hadoop-aliyun/3.4.0/hadoop-aliyun-3.4.0.jar
ARG ALIYUN_SDK_OSS_JAR_URL=https://repo1.maven.org/maven2/com/aliyun/oss/aliyun-sdk-oss/3.17.2/aliyun-sdk-oss-3.17.2.jar
ARG JDOM2_JAR_URL=https://repo1.maven.org/maven2/org/jdom/jdom2/2.0.6.1/jdom2-2.0.6.1.jar
ARG HADOOP_CLIENT_RUNTIME_URL=https://repo1.maven.org/maven2/org/apache/hadoop/hadoop-client-runtime/3.3.4/hadoop-client-runtime-3.3.4.jar
ARG COMMONS_COLLECTIONS_JAR_URL=https://repo1.maven.org/maven2/commons-collections/commons-collections/3.2.2/commons-collections-3.2.2.jar

# Add JuiceFS, Hadoop Aliyun, Aliyun SDK OSS, and JDOM2 JARs
RUN mkdir -p /usr/lib/trino/plugin/hive && \
    curl -#LfS ${JUICEFS_JAR_URL} -o /usr/lib/trino/plugin/hive/juicefs-hadoop-4.9.12.jar && \
    curl -#LfS ${HADOOP_ALIYUN_JAR_URL} -o /usr/lib/trino/plugin/hive/hadoop-aliyun-3.4.0.jar && \
    curl -#LfS ${ALIYUN_SDK_OSS_JAR_URL} -o /usr/lib/trino/plugin/hive/aliyun-sdk-oss-3.17.2.jar && \
    curl -#LfS ${JDOM2_JAR_URL} -o /usr/lib/trino/plugin/hive/jdom2-2.0.6.1.jar && \
    curl -#LfS ${HADOOP_CLIENT_RUNTIME_URL} -o /usr/lib/trino/plugin/hive/hadoop-client-runtime-3.3.4.jar && \
    curl -#LfS ${COMMONS_COLLECTIONS_JAR_URL} -o /usr/lib/trino/plugin/hive/commons-collections-3.2.2.jar





EXPOSE 8080
USER trino:trino
CMD ["/usr/lib/trino/bin/run-trino"]
HEALTHCHECK --interval=10s --timeout=5s --start-period=10s \
  CMD /usr/lib/trino/bin/health-check
