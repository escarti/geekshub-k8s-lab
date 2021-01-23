FROM lachlanevenson/k8s-kubectl:v1.17.0 AS KUBECTL 

FROM jenkins/jenkins:lts-alpine

USER root

RUN apk add --no-cache docker~=19 openrc~=0.42
RUN rc-update add docker boot
RUN curl -L "https://github.com/docker/compose/releases/download/1.28.0/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
RUN chmod +x /usr/local/bin/docker-compose

USER jenkins

# Install kubectl from Docker Hub.
COPY --from=KUBECTL /usr/local/bin/kubectl /usr/local/bin/kubectl
