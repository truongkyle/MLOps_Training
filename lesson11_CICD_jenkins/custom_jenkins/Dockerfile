# Ref: https://hackmamba.io/blog/2022/04/running-docker-in-a-jenkins-container/
FROM jenkins/jenkins:lts

# Install Docker CLI so pipelines can run `docker` inside the Jenkins container
USER root
RUN curl -fsSL https://get.docker.com > /tmp/dockerinstall \
    && chmod +x /tmp/dockerinstall \
    && /tmp/dockerinstall \
    && rm -f /tmp/dockerinstall

# Preinstall essential Jenkins plugins to avoid online installation issues
# The jenkins-plugin-cli resolves dependencies automatically.
COPY plugins.txt /usr/share/jenkins/ref/plugins.txt
RUN jenkins-plugin-cli --plugin-file /usr/share/jenkins/ref/plugins.txt

USER jenkins
