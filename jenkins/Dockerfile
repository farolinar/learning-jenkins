FROM jenkins/inbound-agent:latest-jdk21

USER root
RUN apt-get update \
 && apt-get install -y --no-install-recommends \
      git \
      curl \
      jq \
      ca-certificates \
 && rm -rf /var/lib/apt/lists/*
USER jenkins