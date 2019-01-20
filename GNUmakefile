ROOT_DIR := $(CURDIR)

V ?= 0

RELEASE_NUM ?= 0

MUSL_CROSS_GIT_URL ?= https://github.com/richfelker/musl-cross-make
MUSL_CROSS_GIT_COMMIT ?= 040804dfa622738d723cab4163f19ba484a0aa67

BUSYBOX_GIT_URL ?= https://github.com/mirror/busybox
BUSYBOX_GIT_COMMIT ?= ef800e5441185585986f9b7aaf39010a926fbd5f

KERNEL_GIT_URL ?= https://git.kernel.org/pub/scm/linux/kernel/git/stable/linux.git
KERNEL_GIT_COMMIT ?= e9a713f77bb26886d7207a8bb6dd2c9c7b8e287c

BUILD_DIR ?= $(ROOT_DIR)/build-dir

MUSL_CROSS_DIR ?= $(BUILD_DIR)/musl-cross
MUSL_CROSS_OUT_DIR ?= $(MUSL_CROSS_DIR)/output
MUSL_CROSS_BIN_DIR ?= $(MUSL_CROSS_OUT_DIR)/bin

BUSYBOX_DIR ?= $(BUILD_DIR)/busybox

INITRAMFS_DIR ?= $(BUILD_DIR)/initramfs

KERNEL_DIR ?= $(BUILD_DIR)/kernel

GIT_HASH = $(shell git rev-parse --short HEAD)

.PHONY: all
all: $(BUILD_DIR)/lornix.img

$(BUILD_DIR)/.mkdir-done:
	[ -e "$(BUILD_DIR)" ] && \
		rm -fr "$(BUILD_DIR)" || \
		true
	mkdir "$(BUILD_DIR)"
	touch "$(@)"

$(BUILD_DIR)/.musl-cross-build-done: $(BUILD_DIR)/.mkdir-done
	[ -e "$(MUSL_CROSS_DIR)" ] && \
		rm -fr "$(MUSL_CROSS_DIR)" || \
		true

	git clone "$(MUSL_CROSS_GIT_URL)" "$(MUSL_CROSS_DIR)"
	cd "$(MUSL_CROSS_DIR)"; \
	git checkout "$(MUSL_CROSS_GIT_COMMIT)"

	cd "$(MUSL_CROSS_DIR)"; \
	$(MAKE) TARGET=i486-linux-musl; \
	$(MAKE) TARGET=i486-linux-musl install

	touch "$(@)"

$(BUILD_DIR)/.busybox-build-done: $(BUILD_DIR)/.musl-cross-build-done
	[ -e "$(BUSYBOX_DIR)" ] && \
		rm -fr "$(BUSYBOX_DIR)" || \
		true

	git clone "$(BUSYBOX_GIT_URL)" "$(BUSYBOX_DIR)"
	cd "$(BUSYBOX_DIR)"; \
	git checkout "$(BUSYBOX_GIT_COMMIT)"

	export PATH="$(MUSL_CROSS_BIN_DIR):$${PATH}"; \
	cd "$(BUSYBOX_DIR)"; \
	$(MAKE) CROSS_COMPILE=i486-linux-musl- CFLAGS="-march=i486 -mtune=i486" defconfig; \
	$(MAKE) CROSS_COMPILE=i486-linux-musl- CFLAGS="-march=i486 -mtune=i486"

	touch "$(@)"

$(BUILD_DIR)/.initramfs-tree-done: $(BUILD_DIR)/.busybox-build-done
	[ -e "$(INITRAMFS_DIR)" ] && \
		rm -fr "$(INITRAMFS_DIR)" || \
		true

	mkdir "$(INITRAMFS_DIR)"
	mkdir "$(INITRAMFS_DIR)/lib"
	mkdir -p "$(INITRAMFS_DIR)/usr/sbin"
	mkdir -p "$(INITRAMFS_DIR)/etc"

	cp -a "$(MUSL_CROSS_OUT_DIR)/i486-linux-musl/lib/libc.so" "$(INITRAMFS_DIR)/lib/"
	cp -a "$(BUSYBOX_DIR)/busybox" "$(INITRAMFS_DIR)/usr/sbin/"
	cp -a "$(ROOT_DIR)/init" "$(INITRAMFS_DIR)/"
	cp -a "$(ROOT_DIR)/inittab" "$(INITRAMFS_DIR)/etc/"
	cp -a "$(ROOT_DIR)/profile" "$(INITRAMFS_DIR)/etc/"

	cowsay -f "tux" "LORnix-v4.2-r$(RELEASE_NUM)-git-$(GIT_HASH)" \
		>"$(INITRAMFS_DIR)/etc/banner"

	touch "$(@)"

$(BUILD_DIR)/.kernel-build-done: $(BUILD_DIR)/.musl-cross-build-done $(BUILD_DIR)/.initramfs-tree-done
	[ -e "$(KERNEL_DIR)" ] && \
		rm -fr "$(KERNEL_DIR)" || \
		true

	git clone "$(KERNEL_GIT_URL)" "$(KERNEL_DIR)"
	cd "$(KERNEL_DIR)"; \
	git checkout "$(KERNEL_GIT_COMMIT)"

	cp "$(ROOT_DIR)/kernel-conf" "$(KERNEL_DIR)/.config"
	cp "$(ROOT_DIR)/initramfs-conf" "$(KERNEL_DIR)/"
	cp -rv "$(INITRAMFS_DIR)" "$(KERNEL_DIR)/initramfs"

	export PATH="$(MUSL_CROSS_BIN_DIR):$${PATH}"; \
	cd "$(KERNEL_DIR)"; \
	yes "" | $(MAKE) ARCH=x86 CROSS_COMPILE=i486-linux-musl- oldconfig; \
	$(MAKE) ARCH=x86 CROSS_COMPILE=i486-linux-musl- bzImage

	touch "$(@)"

$(BUILD_DIR)/lornix.img: $(BUILD_DIR)/.kernel-build-done
	$(ROOT_DIR)/guestfish-gen "$(ROOT_DIR)" "$(KERNEL_DIR)" "$(@)" || rm -f "$(@)"
