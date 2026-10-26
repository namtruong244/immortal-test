#
# This is free software, licensed under the GNU General Public License v2.
#

include $(TOPDIR)/rules.mk

PKG_NAME:=easymesh-nr3053
PKG_VERSION:=0.1
PKG_RELEASE:=1
PKG_LICENSE:=GPL-2.0-only

include $(INCLUDE_DIR)/package.mk

define Package/easymesh-nr3053
  SECTION:=MTK Properties
  CATEGORY:=MTK Properties
  SUBMENU:=Applications
  TITLE:=EasyMesh glue (UCI, init, wificonf shim) for Viettel NR3053
  DEPENDS:=+wapp +1905daemon +libmapd +mapd +kmod-mapfilter +kvcedit +datconf-lua +luasocket +mtwifi-cfg-ucode
  PKGARCH:=all
endef

define Package/easymesh-nr3053/description
  UCI front-end (/etc/config/easymesh) and procd init for MediaTek EasyMesh.
  Mesh is disabled by default. Contains no MediaTek code.
endef

define Package/easymesh-nr3053/conffiles
/etc/config/easymesh
endef

define Build/Compile
endef

define Package/easymesh-nr3053/install
	$(INSTALL_DIR) $(1)/etc/config $(1)/etc/init.d $(1)/usr/bin $(1)/usr/libexec/easymesh
	$(INSTALL_CONF) ./files/etc/config/easymesh $(1)/etc/config/easymesh
	$(INSTALL_BIN) ./files/etc/init.d/easymesh $(1)/etc/init.d/easymesh
	$(INSTALL_BIN) ./files/usr/bin/wificonf $(1)/usr/bin/wificonf
	$(INSTALL_BIN) ./files/usr/bin/easymesh $(1)/usr/bin/easymesh
	$(INSTALL_BIN) ./files/usr/libexec/easymesh/prepare $(1)/usr/libexec/easymesh/prepare
endef

$(eval $(call BuildPackage,easymesh-nr3053))
