FROM unityci/editor:ubuntu-2021.3.2f1-mac-mono-1

LABEL maintainer="devops@home.com"

# Adding nexus certificate
RUN apt-get update && apt-get install -y --no-install-recommends \
    sudo tar bash git vim

# Adding user nonroot
RUN adduser nonroot \
    && usermod -aG sudo nonroot \
    && echo "%sudo ALL=(ALL) NOPASSWD: ALL" >> /etc/sudoers

USER nonroot

RUN mkdir ~/.ssh ~/.local

COPY ./docker-entrypoint.sh /
ENTRYPOINT ["/docker-entrypoint.sh"]