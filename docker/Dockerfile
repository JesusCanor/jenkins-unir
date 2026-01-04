FROM jenkins/jenkins:latest-jdk25

USER root
# System dependencies
RUN apt-get update \
 && apt-get install -y --no-install-recommends \
    python3 \
    python3-pip \
    maven \
 && rm -rf /var/lib/apt/lists/*

#System versions
ARG JMETER_VERSION=5.6.3
ARG WIREMOCK_VERSION=2.35.0
 
#Jekins plugins auto installation
RUN jenkins-plugin-cli \
    --plugins \
    warnings-ng \
    coverage \
    performance \
    cucumber-reports

# Python dependencies
COPY requirements.txt /tmp/requirements.txt
RUN pip3 install --no-cache-dir --break-system-packages -r /tmp/requirements.txt

#Wiremock jar
RUN mkdir -p /opt/wiremock \
 && curl -fsSL -o /opt/wiremock/wiremock-standalone.jar \
    "https://repo1.maven.org/maven2/com/github/tomakehurst/wiremock-jre8-standalone/${WIREMOCK_VERSION}/wiremock-jre8-standalone-${WIREMOCK_VERSION}.jar"

#Jmeter installation
RUN mkdir -p /opt/jmeter \
 && curl -fsSL "https://archive.apache.org/dist/jmeter/binaries/apache-jmeter-${JMETER_VERSION}.tgz" -o /tmp/jmeter.tgz \
 && tar -xzf /tmp/jmeter.tgz -C /opt/jmeter --strip-components=1 \
 && rm /tmp/jmeter.tgz \
 && ln -sfn /opt/jmeter/bin/jmeter /usr/local/bin/jmeter \
 && chown -R jenkins:jenkins /opt/jmeter

USER jenkins
