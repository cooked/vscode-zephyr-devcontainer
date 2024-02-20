FROM ghcr.io/zephyrproject-rtos/ci-base:main

ARG WORKDIR 
ARG TARGET
ARG TOOLCHAIN

ARG ZEPHYR_VERSION
ARG ZSDK_VERSION

ARG WGET_ARGS="-q --show-progress --progress=bar:force:noscroll"

# Install Zephyr SDK (minimal + toolchain + host tools)
RUN mkdir -p /opt/toolchains && \
	cd /opt/toolchains && \
	wget ${WGET_ARGS} https://github.com/zephyrproject-rtos/sdk-ng/releases/download/v${ZSDK_VERSION}/zephyr-sdk-${ZSDK_VERSION}_linux-${HOSTTYPE}_minimal.tar.xz && \
	tar xf zephyr-sdk-${ZSDK_VERSION}_linux-${HOSTTYPE}_minimal.tar.xz && \
	rm zephyr-sdk-${ZSDK_VERSION}_linux-${HOSTTYPE}_minimal.tar.xz && \
	# toolchain
	cd zephyr-sdk-${ZSDK_VERSION} && \
	wget https://github.com/zephyrproject-rtos/sdk-ng/releases/download/v0.16.1/toolchain_linux-x86_64_${TOOLCHAIN}.tar.xz && \
    tar xvf toolchain_linux-x86_64_${TOOLCHAIN}.tar.xz && \
    rm toolchain_linux-x86_64_${TOOLCHAIN}.tar.xz && \
    # host tools
    wget https://github.com/zephyrproject-rtos/sdk-ng/releases/download/v0.16.1/hosttools_linux-x86_64.tar.xz && \
    tar xvf hosttools_linux-x86_64.tar.xz && \
    rm hosttools_linux-x86_64.tar.xz
    # run it
    #./zephyr-sdk-${HOSTTYPE}-hosttools-standalone-0.9.sh

# Run the Zephyr SDK setup script as 'user' in order to ensure that the
# `Zephyr-sdk` CMake package is located in the package registry under the
# user's home directory.
USER user

RUN sudo -E -- bash -c ' \
	/opt/toolchains/zephyr-sdk-${ZSDK_VERSION}/setup.sh -c && \
	chown -R user:user /home/user/.cmake \
	'

WORKDIR /${WORKDIR}
RUN \
	west init -m https://github.com/cooked/vscode-zephyr-template --mf .manifests/west-${TARGET}.yml && \
    #west init -l manifest && \
    west update && \
    python3 -m pip install -r zephyr/scripts/requirements.txt

# Set the locale
ENV PKG_CONFIG_PATH=/usr/lib/i386-linux-gnu/pkgconfig
ENV OVMF_FD_PATH=/usr/share/ovmf/OVMF.fd

ENV LC_ALL=C.UTF-8
ENV LANG=C.UTF-8
ENV XDG_CACHE_HOME=/${WORKDIR}/.cache
ENV ZEPHYR_TOOLCHAIN_VARIANT=zephyr
ENV ZEPHYR_SDK_INSTALL_DIR=/${WORKDIR}/zephyr-sdk
ENV ZEPHYR_BASE=/${WORKDIR}/zephyr
ENV PATH="${ZEPHYR_BASE}/scripts:${PATH}"