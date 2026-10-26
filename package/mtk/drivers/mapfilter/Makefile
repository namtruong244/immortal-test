# All rights reserved.
#
# This is free software, licensed under the GNU General Public License v2.
# See /LICENSE for more information.

include $(TOPDIR)/rules.mk
include $(INCLUDE_DIR)/kernel.mk

PKG_NAME:=mapfilter
PKG_REVISION:=96f2afed
PKG_SOURCE:=$(PKG_NAME)-$(PKG_REVISION).tar.bz2
PKG_SOURCE_URL:=file://$(TOPDIR)/easymesh/dl
PKG_HASH:=3928c6614b84e74f1f1762cee61de8e422299e792f5f363dc369dfc0ed0f1714
PKG_LICENSE:=BSD-3-Clause OR GPL-2.0-only

include $(INCLUDE_DIR)/package.mk

TARGET_CFLAGS += -DCONFIG_SUPPORT_OPENWRT
MAKE_FLAGS += CONFIG_SUPPORT_OPENWRT=y

define KernelPackage/mapfilter
  CATEGORY:=MTK Properties
  TITLE:=MTK map filter
  FILES:=$(PKG_BUILD_DIR)/mapfilter.ko
  AUTOLOAD:=$(call AutoProbe,mapfilter)
  SUBMENU:=Drivers
endef



EXTRA_KCONFIG:= \
	CONFIG_MAP_FILTER_SUPPORT=m

EXTRA_CFLAGS:= \
	$(patsubst CONFIG_%, -DCONFIG_%=1, $(patsubst %=m,%,$(filter %=m,$(EXTRA_KCONFIG)))) \
	$(patsubst CONFIG_%, -DCONFIG_%=1, $(patsubst %=y,%,$(filter %=y,$(EXTRA_KCONFIG))))

define Build/Compile
	+$(MAKE) $(PKG_JOBS) -C "$(LINUX_DIR)" \
		ARCH="$(LINUX_KARCH)" \
		CROSS_COMPILE="$(TARGET_CROSS)" \
		M="$(PKG_BUILD_DIR)" \
		EXTRA_CFLAGS="$(EXTRA_CFLAGS)" \
		$(EXTRA_KCONFIG) \
		modules
endef

$(eval $(call KernelPackage,mapfilter))
