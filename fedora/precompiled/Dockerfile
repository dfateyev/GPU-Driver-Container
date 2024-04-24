
ARG FEDORA_VERSION

FROM fedora:${FEDORA_VERSION} as builder

ARG BASE_URL='https://us.download.nvidia.com/tesla'

ARG FEDORA_VERSION
ARG KERNEL_SHORT
ARG KERNEL_BUILD
ARG KERNEL_ARCH
ARG KERNEL_VERSION

ARG DRIVER_VERSION
ARG DRIVER_OPEN='false'
ARG DRIVER_PATH='kernel'
ARG BUILDER_USER
ARG BUILDER_EMAIL

COPY x509-configuration.ini /root/x509-configuration.ini

RUN dnf -y install curl git binutils cpp gcc make pkgconfig openssl unzip dkms elfutils-libelf-devel openssl-devel module-init-tools rpm-build

# todo: check kernel-headers requirement / set suitable kernel-headers available 
RUN curl -O https://kojipkgs.fedoraproject.org/packages/kernel/${KERNEL_SHORT}/${KERNEL_BUILD}/${KERNEL_ARCH}/kernel-devel-${KERNEL_VERSION}.rpm && \
    curl -O https://kojipkgs.fedoraproject.org/packages/kernel/${KERNEL_SHORT}/${KERNEL_BUILD}/${KERNEL_ARCH}/kernel-core-${KERNEL_VERSION}.rpm && \
    curl -O https://kojipkgs.fedoraproject.org/packages/kernel/${KERNEL_SHORT}/${KERNEL_BUILD}/${KERNEL_ARCH}/kernel-modules-${KERNEL_VERSION}.rpm && \
    curl -O https://kojipkgs.fedoraproject.org/packages/kernel/${KERNEL_SHORT}/${KERNEL_BUILD}/${KERNEL_ARCH}/kernel-modules-core-${KERNEL_VERSION}.rpm && \
    curl -O https://kojipkgs.fedoraproject.org/packages/kernel-headers/6.5.4/300.fc39/x86_64/kernel-headers-6.5.4-300.fc39.x86_64.rpm && \
    dnf localinstall -y kernel-*.rpm && \
    dnf clean all

RUN export DRIVER_STREAM=$(echo ${DRIVER_VERSION} | cut -d '.' -f 1) KREL=$(echo ${KERNEL_BUILD} | cut -d '.' -f 1) \
    && if [ "${DRIVER_OPEN}" == "true" ] ; then export DRIVER_PATH="${DRIVER_PATH}-open" ; fi \
    && git clone --depth 1 --single-branch -b fedora https://github.com/NVIDIA/yum-packaging-precompiled-kmod \
    && cd yum-packaging-precompiled-kmod \
    && mkdir BUILD BUILDROOT RPMS SRPMS SOURCES SPECS \
    && mkdir nvidia-kmod-${DRIVER_VERSION}-${KERNEL_ARCH} \
    && curl -sLOf ${BASE_URL}/${DRIVER_VERSION}/NVIDIA-Linux-${KERNEL_ARCH}-${DRIVER_VERSION}.run \
    && sh ./NVIDIA-Linux-${KERNEL_ARCH}-${DRIVER_VERSION}.run --extract-only --target tmp \
    && mv tmp/${DRIVER_PATH} nvidia-kmod-${DRIVER_VERSION}-${KERNEL_ARCH}/kernel \
    && tar -cJf SOURCES/nvidia-kmod-${DRIVER_VERSION}-${KERNEL_ARCH}.tar.xz nvidia-kmod-${DRIVER_VERSION}-${KERNEL_ARCH} \
    && mv kmod-nvidia.spec SPECS/ \
    && sed -i -e "s/\$USER/${BUILDER_USER}/" -e "s/\$EMAIL/${BUILDER_EMAIL}/" ${HOME}/x509-configuration.ini \
    && openssl req -x509 -new -nodes -utf8 -sha256 -days 36500 -batch \
      -config ${HOME}/x509-configuration.ini \
      -outform DER -out SOURCES/public_key.der \
      -keyout SOURCES/private_key.priv \
    && rpmbuild \
        --define "% _arch ${KERNEL_ARCH}" \
        --define "%_topdir $(pwd)" \
        --define "debug_package %{nil}" \
        --define "kernel ${KERNEL_SHORT}" \
        --define "kernel_release ${KREL}" \
        --define "kernel_dist .fc${FEDORA_VERSION}" \
        --define "driver ${DRIVER_VERSION}" \
        --define "driver_branch ${DRIVER_STREAM}" \
        -v -bb SPECS/kmod-nvidia.spec


FROM fedora:${FEDORA_VERSION}

ARG BASE_URL='https://us.download.nvidia.com/tesla'

ARG FEDORA_VERSION
ARG KERNEL_ARCH
ARG KERNEL_VERSION
ENV TARGETARCH=${KERNEL_ARCH}
ARG CUDA_VERSION

ARG DRIVER_TYPE=passthrough
ENV DRIVER_TYPE=${DRIVER_TYPE}

ARG DRIVER_VERSION
ENV DRIVER_VERSION=${DRIVER_VERSION}

ARG BUILDER_USER
ARG BUILDER_EMAIL

# Disable vGPU version compability check by default
ARG DISABLE_VGPU_VERSION_CHECK=true
ENV DISABLE_VGPU_VERSION_CHECK=$DISABLE_VGPU_VERSION_CHECK

COPY --chmod=744 nvidia-driver /usr/local/bin
COPY --chmod=744 common.sh /usr/local/bin

# Copy the built NVIDIA driver RPM from the builder
COPY --from=builder /yum-packaging-precompiled-kmod/RPMS/${KERNEL_ARCH}/*.rpm /rpms/

# Copy the firmware files
COPY --from=builder --chmod=444 /yum-packaging-precompiled-kmod/tmp/firmware/*.bin /opt/lib/firmware/nvidia/${DRIVER_VERSION}/

# Kernel packages needed to build drivers / kmod
RUN dnf -y install kmod binutils util-linux

# Prevent modprobe from giving a WARNING about missing files
RUN rm -rf /lib/modules/${KERNEL_VERSION} \
    && mkdir -p /lib/modules/${KERNEL_VERSION}/proc \
    && touch /lib/modules/${KERNEL_VERSION}/modules.order \
    && touch /lib/modules/${KERNEL_VERSION}/modules.builtin \
    && depmod ${KERNEL_VERSION}

# Copy the DRM module dependency from the builder
# todo: probably needed only for FCOS w/o ignition
COPY --from=builder /lib/modules/${KERNEL_VERSION}/kernel/drivers /lib/modules/${KERNEL_VERSION}/kernel/drivers

# Install the Driver modules
RUN dnf install -y /rpms/kmod-nvidia-*.rpm

RUN export DRIVER_STREAM=$(echo ${DRIVER_VERSION} | cut -d '.' -f 1) \
    && CUDA_VERSION_ARRAY=(${CUDA_VERSION//./ }) && CUDA_DASHED_VERSION=${CUDA_VERSION_ARRAY[0]}-${CUDA_VERSION_ARRAY[1]} \
    && curl -sLf http://developer.download.nvidia.com/compute/cuda/repos/fedora${FEDORA_VERSION}/x86_64/cuda-fedora${FEDORA_VERSION}.repo \
    -o /etc/yum.repos.d/cuda.repo \
    && dnf -y install --skip-broken dnf-plugin-nvidia \
    && dnf -y module enable nvidia-driver:${DRIVER_STREAM}/default \
    && dnf install -y \
        nvidia-driver-cuda-${DRIVER_VERSION} \
        nvidia-driver-libs-${DRIVER_VERSION} \
        nvidia-driver-NVML-${DRIVER_VERSION} \
        cuda-compat-${CUDA_DASHED_VERSION} \
        cuda-cudart-${CUDA_DASHED_VERSION} \
        nvidia-persistenced-${DRIVER_VERSION} \
    && if [ "$DRIVER_TYPE" != "vgpu" ] && [ "$TARGETARCH" != "arm64" ]; then \
        versionArray=(${DRIVER_VERSION//./ }); \
        DRIVER_BRANCH=${versionArray[0]}; \
        dnf module enable -y nvidia-driver:${DRIVER_BRANCH} && \
        dnf install -y nvidia-fabric-manager-${DRIVER_VERSION} libnvidia-nscq-${DRIVER_BRANCH}-${DRIVER_VERSION} ; \
    fi; \
    dnf clean all

LABEL io.k8s.display-name="NVIDIA Driver Container"
LABEL name="NVIDIA Driver Container"
LABEL vendor="NVIDIA"
LABEL maintainer="${BUILDER_USER} <${BUILDER_EMAIL}>"
LABEL atomic.type="system"
LABEL version="${DRIVER_VERSION}"
LABEL architecture="${KERNEL_ARCH}"
LABEL release="N/A"
LABEL summary="Provision the NVIDIA driver through containers"
LABEL description="See summary"

# Remove cuda repository to avoid GPG errors
RUN rm -f /etc/yum.repos.d/cuda.repo

RUN ls /opt/lib/firmware/nvidia/ && ls /opt/lib/firmware/nvidia/${DRIVER_VERSION}/

ENTRYPOINT ["nvidia-driver", "init"]
