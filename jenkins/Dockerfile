ARG DOCKER_VERSION=29.8.1
ARG JENKINS_VERSION=2.568.3-lts-jdk21

FROM docker:${DOCKER_VERSION}-cli AS docker-cli

FROM jenkins/jenkins:${JENKINS_VERSION}

# 讓 pipeline 可透過掛載的 docker.sock 操作主機 Docker
COPY --from=docker-cli /usr/local/bin/docker /usr/local/bin/docker
COPY --from=docker-cli /usr/local/libexec/docker/cli-plugins /usr/local/libexec/docker/cli-plugins
