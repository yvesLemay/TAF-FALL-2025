FROM jenkins/jenkins:lts-jdk17

USER root

# Install essential tools
RUN apt-get update && apt-get install -y --no-install-recommends \
    wget \
    ca-certificates \
    gnupg \
    python3 \
    python3-pip \
 && rm -rf /var/lib/apt/lists/*

# Install Temurin JDK 21
RUN mkdir -p /opt/jdk21 \
 && wget -qO- https://api.adoptium.net/v3/binary/latest/21/ga/linux/x64/jdk/hotspot/normal/eclipse \
    | tar xz --strip-components=1 -C /opt/jdk21

# Set Java 21 as the default
ENV JAVA_HOME=/opt/jdk21
ENV PATH="${JAVA_HOME}/bin:${PATH}"

# Verify Java installation
RUN java -version && javac -version

USER jenkins
