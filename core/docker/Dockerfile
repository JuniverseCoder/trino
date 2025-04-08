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
    microdnf install -y tar gzip wget && \
    mkdir -p "${JAVA_HOME}" && \
    cd "${JAVA_HOME}" && \
    echo "Downloading JDK from ${JDK_DOWNLOAD_LINK}" && \
    wget -c --progress=bar:force "${JDK_DOWNLOAD_LINK}" -O jdk.tar.gz && \
    tar -zxvf jdk.tar.gz --strip 1 && \
    rm -f jdk.tar.gz

# Use ubi9 minimal as it's more secure
FROM registry.access.redhat.com/ubi9/ubi-minimal:latest
ARG JDK_VERSION
ENV JAVA_HOME="/usr/lib/jvm/jdk-${JDK_VERSION}"
ENV PATH=$PATH:$JAVA_HOME/bin
COPY --from=jdk-download $JAVA_HOME $JAVA_HOME

RUN \
    set -xeu && \
    echo -e "[ali_baseos]\nname=ali_baseos\nbaseurl=https://mirrors.aliyun.com/centos-stream/9-stream/BaseOS/x86_64/os/\ngpgcheck=0" > /etc/yum.repos.d/aliyun_yum.repo && \
    echo -e "\n[ali_appstream]\nname=ali_appstream\nbaseurl=https://mirrors.aliyun.com/centos-stream/9-stream/AppStream/x86_64/os/\ngpgcheck=0" >> /etc/yum.repos.d/aliyun_yum.repo && \
    microdnf update -y && \
    microdnf install -y \
        tar \
        less \
        python3 \
        shadow-utils \
        vim \
        procps-ng \
        net-tools \
        telnet \
        sudo && \
    update-alternatives --install /usr/bin/python python /usr/bin/python3 1 && \
    groupadd trino --gid 1000 && \
    useradd trino --uid 1000 --gid 1000 --create-home && \
    mkdir -p /usr/lib/trino /data/trino && \
    chown -R "trino:trino" /usr/lib/trino /data/trino && \
    echo "trino ALL=(ALL) NOPASSWD: ALL" >> /etc/sudoers && \
    microdnf clean all


ARG TRINO_VERSION
COPY --chown=trino:trino trino-cli-${TRINO_VERSION}-executable.jar /usr/bin/trino
COPY --chown=trino:trino trino-server-${TRINO_VERSION} /usr/lib/trino
COPY --chown=trino:trino default/etc /etc/trino
COPY --chown=trino:trino --from=jvmkill /libjvmkill.so /usr/lib/trino/bin
COPY --chown=trino:trino trino-base/mysql-connector-java-8.0.29.jar /usr/lib/trino/lib/mysql-connector-java-8.0.29.jar
COPY --chown=trino:trino trino-base/limits.conf /etc/security/limits.conf
COPY --chown=trino:trino trino-hdfs/*.jar /usr/lib/trino/plugin/hive/hdfs
COPY --chown=trino:trino trino-hdfs/*.jar /usr/lib/trino/plugin/iceberg/hdfs
COPY --chown=trino:trino ldap-group-provider /usr/lib/trino/plugin/ldap-group-provider


EXPOSE 8080
USER trino:trino
CMD ["/usr/lib/trino/bin/run-trino"]
HEALTHCHECK --interval=10s --timeout=5s --start-period=10s \
  CMD /usr/lib/trino/bin/health-check
