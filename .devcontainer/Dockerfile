# CI Image (ci) 
# -- only selected toolchains

#FROM debian:11-slim
FROM ubuntu:22.04

ARG UID=1000
ARG GID=1000

# Set default shell during Docker image build to bash
SHELL ["/bin/bash", "-c"]

# Set non-interactive frontend for apt-get to skip any user confirmations
ENV DEBIAN_FRONTEND=noninteractive

# Install base packages
RUN apt-get -y update && \
	apt-get -y upgrade && \
	apt-get install --no-install-recommends -y \
		software-properties-common \
		lsb-release \
		autoconf \
		automake \
		bison \
		build-essential \
		ca-certificates \
		cargo \
		ccache \
		chrpath \
		cmake \
		cpio \
		device-tree-compiler \
		dfu-util \
		diffstat \
		dos2unix \
		doxygen \
		file \
		flex \
		g++ \
		gawk \
		gcc \
		gcovr \
		git \
		git-core \
		gnupg \
		gperf \
		gtk-sharp2 \
		help2man \
		iproute2 \
		lcov \
		libcairo2-dev \
		libglib2.0-dev \
		libgtk2.0-0 \
		liblocale-gettext-perl \
		libncurses5-dev \
		libpcap-dev \
		libpopt0 \
		libsdl1.2-dev \
		libsdl2-dev \
		libssl-dev \
		libtool \
		libtool-bin \
		locales \
		make \
		net-tools \
		ninja-build \
		openssh-client \
		parallel \
		pkg-config \
		python3-dev \
		python3-pip \
		python3-ply \
		python3-setuptools \
		python-is-python3 \
		qemu \
		rsync \
		socat \
		srecord \
		sudo \
		texinfo \
		unzip \
		valgrind \
		wget \
		ovmf \
		xz-utils \
		thrift-compiler

# Install multi-lib gcc (x86 only)
RUN if [ "${HOSTTYPE}" = "x86_64" ]; then \
	apt-get install --no-install-recommends -y \
		gcc-multilib \
		g++-multilib \
	; fi

# Install i386 packages (x86 only)
RUN if [ "${HOSTTYPE}" = "x86_64" ]; then \
	dpkg --add-architecture i386 && \
	apt-get -y update && \
	apt-get -y upgrade && \
	apt-get install --no-install-recommends -y \
		libsdl2-dev:i386 \
	; fi

# Initialise system locale
RUN locale-gen en_US.UTF-8
ENV LANG=en_US.UTF-8
ENV LANGUAGE=en_US:en
ENV LC_ALL=en_US.UTF-8

# Install Python dependencies
RUN python3 -m pip install -U --no-cache-dir pip && \
	pip3 install -U --no-cache-dir wheel setuptools && \
	pip3 install --no-cache-dir pygobject && \
	pip3 install --no-cache-dir \
		-r https://raw.githubusercontent.com/zephyrproject-rtos/zephyr/main/scripts/requirements.txt \
		-r https://raw.githubusercontent.com/zephyrproject-rtos/mcuboot/main/scripts/requirements.txt \
		GitPython imgtool junitparser numpy protobuf PyGithub \
		pylint sh statistics west && \
	pip3 check

# Clean up stale packages
RUN apt-get clean -y && \
	apt-get autoremove --purge -y && \
	rm -rf /var/lib/apt/lists/*

# Create 'user' account
RUN groupadd -g $GID -o user

RUN useradd -u $UID -m -g user -G plugdev user \
	&& echo 'user ALL = NOPASSWD: ALL' > /etc/sudoers.d/user \
	&& chmod 0440 /etc/sudoers.d/user

USER root




### second stage




ARG TOOLCHAIN=arm-zephyr-eabi

ARG ZEPHYR_VERSION=3.3.0
ARG ZSDK_VERSION=0.16.1
ARG DOXYGEN_VERSION=1.9.4
ARG RENODE_VERSION=1.13.3
ARG LLVM_VERSION=16
ARG BSIM_VERSION=v2.1
ARG SPARSE_VERSION=9212270048c3bd23f56c20a83d4f89b870b2b26e
ARG PROTOC_VERSION=21.7
ARG WGET_ARGS="-q --show-progress --progress=bar:force:noscroll"

# Install Zephyr SDK (minimal + toolchain + host tools)
RUN mkdir -p /opt/toolchains && \
	cd /opt/toolchains && \
	wget ${WGET_ARGS} https://github.com/zephyrproject-rtos/sdk-ng/releases/download/v${ZSDK_VERSION}/zephyr-sdk-${ZSDK_VERSION}_linux-${HOSTTYPE}_minimal.tar.xz && \
	tar xf zephyr-sdk-${ZSDK_VERSION}_linux-${HOSTTYPE}_minimal.tar.xz && \
	rm zephyr-sdk-${ZSDK_VERSION}_linux-${HOSTTYPE}_minimal.tar.xz && \
	# toolchain
	cd zephyr-sdk-${ZSDK_VERSION} && \
	wget https://github.com/zephyrproject-rtos/sdk-ng/releases/download/v0.16.1/toolchain_linux-x86_64_arm-zephyr-eabi.tar.xz && \
    tar xvf toolchain_linux-x86_64_arm-zephyr-eabi.tar.xz && \
    rm toolchain_linux-x86_64_arm-zephyr-eabi.tar.xz && \
    # host tools
    wget https://github.com/zephyrproject-rtos/sdk-ng/releases/download/v0.16.1/hosttools_linux-x86_64.tar.xz && \
    tar xvf hosttools_linux-x86_64.tar.xz && \
    rm hosttools_linux-x86_64.tar.xz
    # run it
    #./zephyr-sdk-${HOSTTYPE}-hosttools-standalone-0.9.sh


# Clean up stale packages
RUN apt-get clean -y && \
	apt-get autoremove --purge -y && \
	rm -rf /var/lib/apt/lists/*

# Run the Zephyr SDK setup script as 'user' in order to ensure that the
# `Zephyr-sdk` CMake package is located in the package registry under the
# user's home directory.
USER user

RUN sudo -E -- bash -c ' \
	/opt/toolchains/zephyr-sdk-${ZSDK_VERSION}/setup.sh -c && \
	chown -R user:user /home/user/.cmake \
	'

WORKDIR /workdir

RUN \
    west init -m https://github.com/zephyrproject-rtos/zephyr --mr v${ZEPHYR_VERSION} && \
    west update --narrow -o=--depth=1 && \
    echo "Installing requirements: zephyr/scripts/requirements.txt" && \
    python3 -m pip install -r zephyr/scripts/requirements.txt
    #echo "Installing requirements: bootloader/mcuboot/scripts/requirements.txt" && \
    #python3 -m pip install -r bootloader/mcuboot/scripts/requirements.txt

RUN mkdir /workdir/project

WORKDIR /workdir/project


# Set the locale
ENV PKG_CONFIG_PATH=/usr/lib/i386-linux-gnu/pkgconfig
ENV OVMF_FD_PATH=/usr/share/ovmf/OVMF.fd

ENV LC_ALL=C.UTF-8
ENV LANG=C.UTF-8
ENV XDG_CACHE_HOME=/workdir/.cache
ENV ZEPHYR_TOOLCHAIN_VARIANT=zephyr
ENV ZEPHYR_SDK_INSTALL_DIR=/workdir/zephyr-sdk
ENV ZEPHYR_BASE=/workdir/zephyr
ENV PATH="${ZEPHYR_BASE}/scripts:${PATH}"