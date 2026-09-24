ARG DOCKER_VERSION=29.8.1
ARG AGENT_VERSION=3391.va_37fa_a_305d6d-3-jdk21

FROM docker:${DOCKER_VERSION}-cli AS docker-cli

FROM jenkins/inbound-agent:${AGENT_VERSION}

# 讓 pipeline 可透過掛載的 docker.sock 操作主機 Docker
COPY --from=docker-cli /usr/local/bin/docker /usr/local/bin/docker
COPY --from=docker-cli /usr/local/libexec/docker/cli-plugins /usr/local/libexec/docker/cli-plugins
