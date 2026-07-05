# https://github.com/zephyrproject-rtos/docker-image/tree/main

#FROM ghcr.io/zephyrproject-rtos/ci-base:main
# For zephyr 3.7 use v0.26 branch
FROM ghcr.io/zephyrproject-rtos/ci-base:v0.26.18 


# Explicit ARGs and their defaults. Defaults get overwritten by 
# devcontainer.json (local build) or GitHub workflow (CI)
ARG TARGET="stm32"
ARG TOOLCHAIN="arm-zephyr-eabi"
ARG ZSDK_VERSION="1.0.1"
ARG ZEPHYR_VERSION="v4.4.0"
ARG ZEPHYR_ALLOW_LIST="cmsis, hal_stm32"
ARG WGET_ARGS="-q --show-progress --progress=bar:force:noscroll"
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
	apt-get -y install --no-install-recommends \
        udev \
        libmagic1

# Hack udev to run inside a container
# https://stackoverflow.com/questions/62060604/why-udev-init-script-default-disable-container-support-while-in-fact-it-works
RUN sed -i.bak -e '/if \[ ! -w \/sys \]/,+3 s/^/#/' /etc/init.d/udev

# udev rules (for Nordic boards)
RUN apt-get -y install libunistring2
RUN wget https://github.com/NordicSemiconductor/nrf-udev/releases/download/v1.0.1/nrf-udev_1.0.1-all.deb -O nrf-udev.deb && \
    dpkg -i nrf-udev.deb && \
    rm ./nrf-udev.deb

# Install the st-util GDB server
# https://manpages.ubuntu.com/manpages/focal/en/man1/st-util.1.html
# TODO: make it conditional based on TARGET
RUN apt-get -y install stlink-tools

# Jlink
# https://github.com/ScoopInstaller/Scoop/issues/4336#issue-864466149
RUN apt-get -y install \
    libxkbcommon-x11-0 \
	libxcb-image0 \
	libxcb-keysyms1 \
	libxcb-icccm4 \
	libxcb-shape0 \
	libxcb-render-util0
RUN wget --post-data "accept_license_agreement=accepted" https://www.segger.com/downloads/jlink/JLink_Linux_x86_64.deb && \
	dpkg -i ./JLink_Linux_x86_64.deb && \
	rm ./JLink_Linux_x86_64.deb

# nrf Util
RUN wget https://files.nordicsemi.com/artifactory/swtools/external/nrfutil/executables/x86_64-unknown-linux-gnu/nrfutil -O nrfutil && \
    chmod +x nrfutil && \
    mv nrfutil /usr/bin


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
    mkdir -p /lib/udev/rules.d && \
    cp $(find . -name 60-openocd.rules | head -n 1) /lib/udev/rules.d/

# Run the Zephyr SDK setup script as 'user' in order to ensure that the
# `Zephyr-sdk` CMake package is located in the package registry under the
# user's home directory.
USER $USERNAME
WORKDIR /home/$USERNAME

RUN sudo -E -- bash -c ' \
	${ZEPHYR_SDK}/setup.sh -c && \
	chown -R ${USERNAME}:${USERNAME} /home/${USERNAME}/.cmake'

# Define a hidden staging directory outside the bind-mount path
ENV STAGE_DIR=/home/${USERNAME}/.stage
RUN mkdir -p ${STAGE_DIR}
WORKDIR ${STAGE_DIR}

# Generate the west.yml manifest dynamically inside the staging directory
RUN mkdir .manifest && \
    echo "manifest:" > .manifest/west.yml && \
    echo "  projects:" >> .manifest/west.yml && \
    echo "    - name: zephyr" >> .manifest/west.yml && \
    echo "      url: https://github.com/zephyrproject-rtos/zephyr" >> .manifest/west.yml && \
    echo "      revision: ${ZEPHYR_VERSION}" >> .manifest/west.yml && \
    echo "      west-commands: scripts/west-commands.yml" >> .manifest/west.yml && \
    echo "      import:" >> .manifest/west.yml && \
    echo "        name-allowlist:" >> .manifest/west.yml && \
    echo "${ZEPHYR_ALLOW_LIST}" | tr ',' '\n' | sed 's/ //g' | while read -r module; do \
        if [ ! -z "$module" ]; then \
            echo "          - $module" >> .manifest/west.yml; \
        fi; \
    done && \
    echo "  self:" >> .manifest/west.yml && \
    echo "    path: .manifest" >> .manifest/west.yml

# Initialize and pull all code into the staging directory during image build
RUN west init -l .manifest && \
    west update

WORKDIR /home/$USERNAME

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

# Set the final default working directory back to your active workspace
WORKDIR /home/${USERNAME}/workspace