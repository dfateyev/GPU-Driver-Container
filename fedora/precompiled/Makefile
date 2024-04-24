FEDORA_VERSION = 39
KERNEL_SHORT = 6.5.11
KERNEL_BUILD = 300.fc39
KERNEL_ARCH = x86_64

DRIVER_VERSION = 550.54.15
DRIVER_TYPE = "passthrough"
CUDA_VERSION = 12.4.0
BASE_URL ?= https://us.download.nvidia.com/tesla

KERNEL_VERSION ?= $(KERNEL_SHORT)-$(KERNEL_BUILD).$(KERNEL_ARCH)
ifeq ($(DRIVER_TYPE),vgpu)
    IMAGE_NAME ?= vgpu-guest-driver
else
    IMAGE_NAME ?= driver
endif
CONTAINER_TAG ?= dfateyev/$(IMAGE_NAME):$(DRIVER_VERSION)-$(KERNEL_VERSION)-fedora$(FEDORA_VERSION)

validate:
	@if [ -z "$(DRIVER_VERSION)" ]; then \
		echo "DRIVER_VERSION cannot be empty, automatic detection has failed."; \
		exit 1; \
	fi;
	@if [ -z "$(KERNEL_VERSION)" ]; then \
		echo "KERNEL_VERSION cannot be empty, automatic detection has failed."; \
		exit 1; \
	fi;

build: validate
	echo "Building Docker Image ... " && \
	docker build \
		--rm=false \
		--network=host \
		--build-arg FEDORA_VERSION=$(FEDORA_VERSION) \
		--build-arg KERNEL_VERSION=$(KERNEL_VERSION) \
		--build-arg KERNEL_SHORT=$(KERNEL_SHORT) \
		--build-arg KERNEL_BUILD=$(KERNEL_BUILD) \
		--build-arg KERNEL_ARCH=$(KERNEL_ARCH) \
		--build-arg IMAGE_VERSION=$(IMAGE_VERSION) \
		--build-arg DRIVER_VERSION=$(DRIVER_VERSION) \
		--build-arg DRIVER_TYPE=$(DRIVER_TYPE) \
		--build-arg CUDA_VERSION=$(CUDA_VERSION) \
		--build-arg BASE_URL=$(BASE_URL) \
		--build-arg BUILDER_USER=Virtuozzo \
		--build-arg BUILDER_EMAIL=dfateyev@virtuozzo.com \
		--tag $(CONTAINER_TAG) \
		--file Dockerfile .

push: build
	if [ "$(DOCKER_USERNAME)" != "" ]; then \
		echo "$(DOCKER_PASSWORD)" | docker login --username="$(DOCKER_USERNAME)" --password-stdin; \
	fi; \
	docker push $(CONTAINER_TAG)

.PHONY: validate build push
