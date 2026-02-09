FROM jenkins/jenkins:lts-jdk17

USER root
RUN apt-get update && \
    DEBIAN_FRONTEND=noninteractive apt-get install -y --no-install-recommends \
      curl unzip jq ca-certificates git && \
    rm -rf /var/lib/apt/lists/*
USER jenkins
