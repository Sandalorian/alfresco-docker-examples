FROM ubuntu:18.04

# Builds the paa url check https://launchpad.net/~libreoffice
ARG LIBREOFFICE_VERSION=6-4

RUN apt update && \
    apt install software-properties-common -y && \
    add-apt-repository ppa:libreoffice/libreoffice-${LIBREOFFICE_VERSION} -y && \
    apt update && \
    apt install libreoffice -y