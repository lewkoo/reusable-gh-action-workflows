# Must supply an argument to `docker build` command like this: docker build ... --build-arg _DISTRO=<distro>
ARG _DISTRO
# Base off the basic ubuntu images
FROM ubuntu:$_DISTRO

# Сonfigure locale
RUN apt-get update && DEBIAN_FRONTEND=noninteractive apt-get install -y locales && \
    sed -i -e 's/# en_US.UTF-8 UTF-8/en_US.UTF-8 UTF-8/' /etc/locale.gen && \
    dpkg-reconfigure --frontend=noninteractive locales && \
    update-locale LANG=en_US.UTF-8

# Install required packages and upgrade git
RUN dpkg-reconfigure debconf --frontend=noninteractive && \ 
    apt-get update && \ 
    apt-get install -yq --no-install-recommends software-properties-common apt-utils debhelper build-essential wget && \
    add-apt-repository -y -u ppa:git-core/ppa && \
    apt-get install -yq git && \
    rm -rf /var/lib/apt/lists/* && \
    apt-get purge --auto-remove && \
    apt-get clean