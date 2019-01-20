ROOT_DIR := $(CURDIR)

V ?= 0

RELEASE_NUM ?= 0

MUSL_CROSS_GIT_URL ?= https://github.com/richfelker/musl-cross-make
MUSL_CROSS_GIT_COMMIT ?= 040804dfa622738d723cab4163f19ba484a0aa67

BUSYBOX_URL ?= https://busybox.net/downloads/busybox-1.30.0.tar.bz2
BUSYBOX_SHA512 ?= c494278f6655cb855e8bd3a316d77b879cf6ee70fa5b0408705391b1108f298d45ab4c2921d939c17122f50c4a9d7b5c77e57bacf5e6c7ac4dc4f78c1bd70a79

KERNEL_URL ?= https://cdn.kernel.org/pub/linux/kernel/v4.x/linux-4.20.3.tar.xz
KERNEL_SHA512 ?= 73461cc34491e2bd9ea7638225e6d06ade8f092efa66fbe7f38c3d7143d4f78b852175bcc1149970958fbcdeacb0ae7507a8544d0bbb55d3d3826787b8f3d615

BUILD_DIR ?= $(ROOT_DIR)/build-dir

MUSL_CROSS_DIR ?= $(BUILD_DIR)/musl-cross
MUSL_CROSS_OUT_DIR ?= $(MUSL_CROSS_DIR)/output
MUSL_CROSS_BIN_DIR ?= $(MUSL_CROSS_OUT_DIR)/bin

BUSYBOX_DIR ?= $(BUILD_DIR)/busybox

INITRAMFS_DIR ?= $(BUILD_DIR)/initramfs

KERNEL_DIR ?= $(BUILD_DIR)/kernel

MUSL_MAKE_OPTS ?=

git_hash = $(shell git rev-parse --short HEAD)
release_ver = v4.2-r$(RELEASE_NUM)-git-$(git_hash)

.PHONY: all
all: $(BUILD_DIR)/lornix-$(release_ver).img

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
	$(MAKE) $(MUSL_MAKE_OPTS) TARGET=i486-linux-musl; \
	$(MAKE) $(MUSL_MAKE_OPTS) TARGET=i486-linux-musl install

	touch "$(@)"

$(BUILD_DIR)/.busybox-build-done: $(BUILD_DIR)/.musl-cross-build-done
	[ -e "$(BUSYBOX_DIR)" ] && \
		rm -fr "$(BUSYBOX_DIR)" || \
		true
	[ -e $(BUILD_DIR)/busybox-* ] && \
		rm -fr $(BUILD_DIR)/busybox-* || \
		true

	wget -t 5 -T 15 -O "$(BUILD_DIR)/bb.tar.bz2" "$(BUSYBOX_URL)"
	echo "$(BUSYBOX_SHA512) $(BUILD_DIR)/bb.tar.bz2" | sha512sum -c -
	tar -C "$(BUILD_DIR)" -xf "$(BUILD_DIR)/bb.tar.bz2"
	mv -v $(BUILD_DIR)/busybox-* "$(BUSYBOX_DIR)"

	cp "$(ROOT_DIR)/busybox-conf" "$(BUSYBOX_DIR)/.config"

	export PATH="$(MUSL_CROSS_BIN_DIR):$${PATH}"; \
	cd "$(BUSYBOX_DIR)"; \
	yes "" | $(MAKE) CROSS_COMPILE=i486-linux-musl- oldconfig; \
	$(MAKE) CROSS_COMPILE=i486-linux-musl-

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

	cowsay -f "$(ROOT_DIR)/lortux.cow" "LORnix-$(release_ver)" \
		>"$(INITRAMFS_DIR)/etc/banner"

	touch "$(@)"

$(BUILD_DIR)/.kernel-build-done: $(BUILD_DIR)/.musl-cross-build-done $(BUILD_DIR)/.initramfs-tree-done
	[ -e "$(KERNEL_DIR)" ] && \
		rm -fr "$(KERNEL_DIR)" || \
		true
	[ -e $(BUILD_DIR)/linux-* ] && \
		rm -fr $(BUILD_DIR)/linux-* || \
		true

	wget -t 5 -T 15 -O "$(BUILD_DIR)/k.tar.xz" "$(KERNEL_URL)"
	echo "$(KERNEL_SHA512) $(BUILD_DIR)/k.tar.xz" | sha512sum -c -
	tar -C "$(BUILD_DIR)" -xf "$(BUILD_DIR)/k.tar.xz"
	mv -v $(BUILD_DIR)/linux-* "$(KERNEL_DIR)"

	cp "$(ROOT_DIR)/kernel-conf" "$(KERNEL_DIR)/.config"
	cp "$(ROOT_DIR)/initramfs-conf" "$(KERNEL_DIR)/"
	cp -rv "$(INITRAMFS_DIR)" "$(KERNEL_DIR)/initramfs"

	export PATH="$(MUSL_CROSS_BIN_DIR):$${PATH}"; \
	cd "$(KERNEL_DIR)"; \
	yes "" | $(MAKE) ARCH=x86 CROSS_COMPILE=i486-linux-musl- oldconfig; \
	$(MAKE) ARCH=x86 CROSS_COMPILE=i486-linux-musl- bzImage

	touch "$(@)"

$(BUILD_DIR)/lornix-$(release_ver).img: $(BUILD_DIR)/.kernel-build-done
	$(ROOT_DIR)/guestfish-gen "$(ROOT_DIR)" "$(KERNEL_DIR)" "$(@)" || rm -f "$(@)"
