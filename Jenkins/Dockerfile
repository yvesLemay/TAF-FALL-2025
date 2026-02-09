FROM jenkins/jenkins:lts

USER root
RUN apt-get update && \
    DEBIAN_FRONTEND=noninteractive apt-get install -y --no-install-recommends \
      curl unzip jq ca-certificates git openjdk-17-jdk && \
    rm -rf /var/lib/apt/lists/*
USER jenkins
