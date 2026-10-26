#
# Copyright (C) 2006-2012 OpenWrt.org
#
# This is free software, licensed under the GNU General Public License v2.
# See /LICENSE for more information.
#

include $(TOPDIR)/rules.mk

PKG_NAME:=libmapd
PKG_REVISION:=0132d199
PKG_SOURCE:=$(PKG_NAME)-$(PKG_REVISION).tar.bz2
PKG_SOURCE_URL:=file://$(TOPDIR)/easymesh/dl
PKG_HASH:=d325b4a8dc2bf44bd3bd639a0c300380c0a6240f407807f8f2eea80d006c8dba
PKG_LICENSE:=Proprietary
PKG_RELEASE:=1

PKG_BUILD_DIR:=$(BUILD_DIR)/$(PKG_NAME)

include $(INCLUDE_DIR)/package.mk
include $(INCLUDE_DIR)/kernel.mk

define Package/libmapd
  SECTION:=MTK Properties
  CATEGORY:=MTK Properties
  SUBMENU:=Libraries
  TITLE:=mtk mapd library
endef

define Package/libmapd/description
  mtk mapd library
endef

include ./mapd_config

TARGET_CFLAGS += $(FPIC)

# TODO(easymesh): source MTK viết cho GCC 8; GCC 14 coi các lỗi này là error.
TARGET_CFLAGS += -Wno-error=implicit-function-declaration -Wno-error=incompatible-pointer-types

ifeq ($(CONFIG_MULTIAP),y)
TARGET_CFLAGS += -DSUPPORT_MULTI_AP
endif
ifeq (y, $(filter y, $(CONFIG_MTK_MAP_R2_6E_SUPPORT) $(CONFIG_MTK_WIFI7_MAP_R2_6E_SUPPORT)))
TARGET_CFLAGS += -DMAP_6E_SUPPORT
endif
ifeq ($(CONFIG_MTK_WIFI7_MAP_HOSTAPD_SUPPORT), y)
TARGET_CFLAGS += -DMTK_MLO_MAP_SUPPORT
endif
define Build/Compile
	$(MAKE) -C $(PKG_BUILD_DIR) \
		$(MAKE_FLAGS) \
		$(TARGET_CONFIGURE_OPTS) \
		CFLAGS="$(TARGET_CFLAGS)" \
		shared
endef

define Build/InstallDev
	$(INSTALL_DIR)  $(1)/usr/include
	$(INSTALL_DATA) $(PKG_BUILD_DIR)/include/mapd_interface_ctrl.h  $(1)/usr/include/
	$(INSTALL_DIR)  $(1)/usr/lib
	$(CP) $(PKG_BUILD_DIR)/libmapd_interface_client.so* $(1)/usr/lib/
endef

define Package/libmapd/install
	$(INSTALL_DIR) $(1)/usr/lib
	$(CP) $(PKG_BUILD_DIR)/libmapd_interface_client.so* $(1)/usr/lib/
endef

$(eval $(call BuildPackage,libmapd))
