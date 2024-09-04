# https://github.com/zephyrproject-rtos/docker-image/tree/main

FROM ghcr.io/zephyrproject-rtos/ci-base:main

# Args from devcontainer.json (local build) or GitHub workflow (CI) (or default)
ARG TARGET="stm32"
ARG TOOLCHAIN="arm-zephyr-eabi"
ARG ZSDK_VERSION="0.16.8"
ARG WGET_ARGS="-q --show-progress --progress=bar:force:noscroll"
ARG WEST_MANIFEST=".manifests/west-${TARGET}.yml"
ARG ZEPHYR_SDK_DIR="/opt"
ARG USERNAME="user"

ENV PKG_CONFIG_PATH=/usr/lib/i386-linux-gnu/pkgconfig
ENV OVMF_FD_PATH=/usr/share/ovmf/OVMF.fd
ENV LC_ALL=C.UTF-8
ENV LANG=C.UTF-8
ENV ZEPHYR_TOOLCHAIN_VARIANT=zephyr
ENV ZEPHYR_SDK_VERSION=${ZSDK_VERSION}
ENV ZEPHYR_SDK=${ZEPHYR_SDK_DIR}/zephyr-sdk-${ZSDK_VERSION}

# Install udev and git
RUN apt-get -y update && \
	apt-get -y install udev git
	
# Hack udev to run inside a container
# https://stackoverflow.com/questions/62060604/why-udev-init-script-default-disable-container-support-while-in-fact-it-works
RUN sed -i.bak -e '/if \[ ! -w \/sys \]/,+3 s/^/#/' /etc/init.d/udev

# Install Jlink
RUN apt-get -y install libxkbcommon-x11-0 \
	libxcb-image0 \
	libxcb-keysyms1 \
	libxcb-icccm4 \
	libxcb-shape0 \
	libxcb-render-util0
RUN wget --post-data "accept_license_agreement=accepted" https://www.segger.com/downloads/jlink/JLink_Linux_x86_64.deb && \
	dpkg -i ./JLink_Linux_x86_64.deb && \
	rm ./JLink_Linux_x86_64.deb

# Install the st-util GDB server
# https://manpages.ubuntu.com/manpages/focal/en/man1/st-util.1.html
# TODO: make it conditional based on TARGET
RUN apt-get -y install stlink-tools


# install minimal Zephyr SDK
WORKDIR "${ZEPHYR_SDK_DIR}"
RUN wget ${WGET_ARGS} https://github.com/zephyrproject-rtos/sdk-ng/releases/download/v${ZSDK_VERSION}/zephyr-sdk-${ZSDK_VERSION}_linux-${HOSTTYPE}_minimal.tar.xz && \
	tar xf zephyr-sdk-${ZSDK_VERSION}_linux-${HOSTTYPE}_minimal.tar.xz && \
	rm zephyr-sdk-${ZSDK_VERSION}_linux-${HOSTTYPE}_minimal.tar.xz

# install toolchain and host tools
# install OpenOCD udev rules
# https://docs.zephyrproject.org/latest/develop/toolchains/zephyr_sdk.html	
WORKDIR "${ZEPHYR_SDK}"
RUN ./setup.sh -t ${TOOLCHAIN} -h && \
	cp ./sysroots/x86_64-pokysdk-linux/usr/share/openocd/contrib/60-openocd.rules /lib/udev/rules.d/60-openocd.rules


# Run the Zephyr SDK setup script as 'user' in order to ensure that the
# `Zephyr-sdk` CMake package is located in the package registry under the
# user's home directory.
USER $USERNAME
WORKDIR /home/$USERNAME

RUN sudo -E -- bash -c ' \
	${ZEPHYR_SDK}/setup.sh -c && \
	chown -R ${USERNAME}:${USERNAME} /home/${USERNAME}/.cmake'

# test local manifest
RUN mkdir .manifest
COPY --chown=${USERNAME} ${WEST_MANIFEST} .manifest/west.yml

RUN west init -l .manifest && \
	west update && \
    python3 -m pip install -r zephyr/scripts/requirements.txt
# TODO: drop requirements.txt.... already from the base image?

# Install minimal SVD files
# https://stackoverflow.com/a/52269934
# TODO: make it dynamic based on TARGET
RUN git clone -n --depth=1 --filter=tree:0 https://github.com/cmsis-svd/cmsis-svd-data.git && \
	cd cmsis-svd-data && \
	git sparse-checkout set --no-cone data/STMicro && \
	git checkout
ENV SVD_DIR=/home/${USERNAME}/cmsis-svd-data/data/STMicro


ENV XDG_CACHE_HOME=/home/${USERNAME}/.cache
ENV ZEPHYR_BASE=/home/${USERNAME}/zephyr
ENV PATH="${ZEPHYR_BASE}/scripts:${PATH}"
# add openocd to PATH
ENV PATH="${ZEPHYR_SDK}/sysroots/x86_64-pokysdk-linux/usr/bin:${PATH}"
# add toolchain to PATH
ENV PATH="${ZEPHYR_SDK}/${TOOLCHAIN}/bin:${PATH}"
