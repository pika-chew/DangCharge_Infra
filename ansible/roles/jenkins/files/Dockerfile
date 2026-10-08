FROM jenkins/jenkins:lts-jdk21

USER root

# docker CLI만 설치 (데몬은 호스트 것을 docker.sock으로 사용)
RUN apt-get update \
    && apt-get install -y --no-install-recommends lsb-release curl ca-certificates \
    && curl -fsSLo /usr/share/keyrings/docker-archive-keyring.asc \
       https://download.docker.com/linux/debian/gpg \
    && echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.asc] https://download.docker.com/linux/debian $(lsb_release -cs) stable" \
       > /etc/apt/sources.list.d/docker.list \
    && apt-get update \
    && apt-get install -y --no-install-recommends docker-ce-cli \
    && rm -rf /var/lib/apt/lists/*

USER jenkins

# 플러그인 사전 설치 (이미지에 포함되므로 컨테이너를 새로 만들어도 재현됨)
COPY plugins.txt /usr/share/jenkins/ref/plugins.txt
RUN jenkins-plugin-cli --plugin-file /usr/share/jenkins/ref/plugins.txt
