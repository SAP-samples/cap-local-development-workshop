# syntax=docker/dockerfile:1

ARG VARIANT="22"
FROM mcr.microsoft.com/devcontainers/javascript-node:${VARIANT}

# Install some generally useful tools
RUN apt-get update && apt-get -y install --no-install-recommends curl git sqlite3

# Install the cf CLI (linux64) v8
RUN curl -L "https://packages.cloudfoundry.org/stable?release=linux64-binary&version=v8&source=github" \
  | tar -C /usr/local/bin/ -z -x cf cf8

# Install SAP CAP SDK globally
USER node
RUN npm install -g @sap/cds-dk

# Make environment look as much like the BAS version as possible
RUN mkdir /home/node/projects
RUN echo 'export PS1="\[\e]0;\w\a\]\[\033[33;1m\]\u: \[\033[36m\]\$(basename \w) \$\[\033[m\] "' >> /home/node/.bashrc

WORKDIR /home/node
