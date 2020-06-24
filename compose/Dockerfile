FROM lachlanevenson/k8s-kubectl:v1.17.0 AS KUBECTL 

FROM jenkins/jenkins:lts-alpine

USER root

RUN apk add --no-cache docker~=18.09 openrc~=0.39
RUN rc-update add docker boot

USER jenkins

# Install kubectl from Docker Hub.
COPY --from=KUBECTL /usr/local/bin/kubectl /usr/local/bin/kubectl
