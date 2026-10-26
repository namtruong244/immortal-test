# Plan port MediaTek EasyMesh cho Viettel NR3053

Tài liệu bàn giao cho người/agent tiếp tục công việc. Đọc hết phần 0–3 trước khi
sửa bất kỳ file nào.

- Branch làm việc: `feature/easymesh-nr3053` (tách từ `main` @ `04d4395e53`).
- Ngày lập plan: 2026-10-26.
- Bối cảnh nghiên cứu: [docs/trien-khai-mediatek-easymesh.md](../docs/trien-khai-mediatek-easymesh.md).
- Firmware gốc (không EasyMesh) đã build và chạy ổn trên NR3053 → **không cần
  làm lại baseline**, dùng nó làm mốc so sánh và đường rollback.

---

## 0. Mục tiêu và tiêu chí hoàn thành

Build được image ImmortalWrt 25.12 cho **NR3053** có MediaTek EasyMesh R2/R3
(control plane IEEE 1905.1 + Multi-AP: `wapp`, `p1905_managerd`, `mapd`,
`mapfilter.ko`), cho phép một node làm **controller** hoặc **agent**.

Hoàn thành khi tất cả đúng:

1. `bash scripts/build-viettel.sh defconfig/viettel-nr3053-easymesh.config`
   build xong, image NR3053 chứa các package EasyMesh; image 32X6 **không** chứa.
2. Image boot được, mesh mặc định **tắt**, Wi-Fi hoạt động y như bản gốc.
3. Hai NR3053: một controller + một agent, wired backhaul và wireless backhaul
   đều lên topology (`mapd_cli`/`1905ctrl` thấy nhau), client roaming được.
4. Tắt mesh (UCI) → quay về Wi-Fi thường sau `wifi reload` / reboot.

Ngoài phạm vi: 32X6, LuCI đẹp (chỉ cần CLI/UCI tối thiểu), tương thích với node
stock (làm nếu còn thời gian, Phase 6).

---

## 1. Ràng buộc bắt buộc

| # | Ràng buộc | Lý do |
|---|---|---|
| R1 | **Không merge branch này vào `main`, không mở PR vào `main`/upstream, không tạo Release/CI artifact public.** | `1905daemon`, `wappd`, `mapd` mang license độc quyền MediaTek ("confidential and proprietary… reproduction… strictly prohibited"). Repo `origin` (`namtruong244/immortalwrt-mt798x-rebase`) là **public**. |
| R2 | Không sửa `.github/workflows/*`, `defconfig/viettel-only.config`, `scripts/verify-viettel-*.sh` theo cách làm thay đổi build release. | Release flow của `main` phải giữ nguyên. Lưu ý `ci.yml` chạy trên `pull_request` — đừng mở PR. |
| R3 | Không copy binary từ firmware stock (kernel 5.4). | Không tái lập được ABI/dependency. |
| R4 | Không ship SSID/passphrase/khóa mẫu. Mesh mặc định tắt. | An toàn khi flash nhầm. |
| R5 | Không sửa file trong `easymesh/dl/` — archive là input bất biến; mọi thay đổi source qua `patches/` của từng package. | Truy vết được và kiểm hash được. |
| R6 | Mỗi thay đổi một commit nhỏ, message dạng `easymesh: <pkg>: <việc>`. | Dễ bisect. |

Push branch lên remote chỉ khi chủ repo đồng ý rõ ràng (xem R1).

---

## 2. Hiện trạng đã xác minh (2026-10-26)

### 2.1 Những gì có trên branch

```
easymesh/
  PLAN.md                 ← file này
  SHA256SUMS              ← hash 5 archive
  dl/1905daemon-35234223.tar.bz2   (616 KB)
  dl/libmapd-0132d199.tar.bz2      (22 KB)
  dl/mapd-6eca4331.tar.bz2         (1.7 MB)
  dl/mapfilter-96f2afed.tar.bz2    (26 KB)
  dl/wappd-80ca5724.tar.bz2        (752 KB)
package/mtk/applications/1905daemon/Makefile
package/mtk/applications/libmapd/{Makefile,mapd_config}
package/mtk/applications/mapd/Makefile
package/mtk/applications/wappd/{Makefile,Config.in,files/*,patches/001-rtmpiapp.h.patch}
package/mtk/drivers/mapfilter/Makefile
```

Recipe và archive lấy **nguyên văn** từ
`https://github.com/immortalwrt-mt798x/immortalwrt` branch `openwrt-21.02`,
commit `c114d749185be6b846e8d38265549049d49b994d` (thư mục `dl/` và
`package/mtk/...`). Chưa sửa gì. Recipe viết cho OpenWrt 21.02 / kernel 5.4.

> Đính chính tài liệu nghiên cứu: archive **có** nằm trong tree Git upstream
> (`dl/`), và `scripts/audit-easymesh-sources.sh` mà tài liệu nhắc **không tồn
> tại**. Sửa tài liệu ở Phase 7.

### 2.2 Tree hiện tại (ImmortalWrt 25.12, kernel 6.12)

| Thứ | Trạng thái | Vị trí |
|---|---|---|
| Kernel | 6.12 | `target/linux/mediatek/Makefile` (`KERNEL_PATCHVER`) |
| Package format | APK. `VERSION` mặc định = `PKG_RELEASE` nếu thiếu `PKG_VERSION` → recipe không cần `PKG_VERSION` để đóng gói | `include/package-defaults.mk:24-31` |
| Driver `mt_wifi` 7.6.7.3 | Bật `MTK_MAP_SUPPORT`, `MTK_MAP_R2_VER_SUPPORT`, `MTK_MAP_R3_VER_SUPPORT` | `defconfig/viettel-only.config:141-143`, Kconfig `package/mtk/drivers/mt_wifi/config.in:774-800` |
| UAPI WAPP | Có. So với bản 5.4 chỉ khác trường `uplink_rate` trong `#ifdef MAP_MLO_UPLINK` (MT7981 không dùng) | `target/linux/mediatek/files-6.12/include/uapi/linux/wapp/` |
| `libkvcutil`, `libdatconf`, `kvcedit`, `datconf-lua` | Có package | `package/mtk/applications/datconf/Makefile` |
| `libnl-tiny`, `openssl`, `lua` | Có | `package/libs/`, `package/utils/lua` |
| `ip-bridge` | Đã bật | `defconfig/viettel-only.config` |
| `l1profile.dat` MT7981 (`ra0;rax0`, `apcli;apclix`) | Có trong `wifi-dats` (đã bật) | `package/mtk/drivers/wifi-profile/files/mt7981/` |
| Stack cấu hình Wi-Fi | **`mtwifi-cfg-ucode`** (UCI → `.dat` bằng ucode, đọc l1profile qua `ucode-mod-l1parser`). Không có lua stack `wifi-profile`. Không xử lý key `MapMode`. | `package/mtk/applications/mtwifi-cfg-ucode/files/` |
| NR3053 network | DSA, `lan1 lan2 lan3` + `wan` | `target/linux/mediatek/filogic/base-files/etc/board.d/02_network:88` |
| NR3053 image/packages | Override fork | `target/linux/mediatek/image/filogic-ext-viettel-fork.mk:37` |
| `CONFIG_TARGET_PER_DEVICE_ROOTFS=y` | Có → có thể thêm package chỉ cho NR3053 | defconfig |

### 2.3 Nội dung archive (đã kiểm)

| Archive | Ghi chú quan trọng |
|---|---|
| `wappd` | Source C. `Makefile` link `-ldatconf -lkvcutil -lssl -lcrypto`, include `$(UAPD_INCLUDE_PATH)` và `$(ROOTDIR)/$(LINUXDIR)/include/uapi/linux/wapp/`. File `LICENSE` độc quyền MTK. Script `wapp_openwrt.sh`, `dpp_config_agent_impl.lua` gọi `wificonf`. |
| `1905daemon` | Source C, `LICENSE` độc quyền MTK. Recipe build với `GENL_NETLINK_SUPPORT=y`, `SWTICH_DSA_ARCH=y`, include `$(LINUX_DIR)/drivers/net/ethernet/raeth` (đường dẫn này không tồn tại trên 6.12 — kiểm xem có thực sự cần). |
| `libmapd` | Source C nhỏ, build `libmapd_interface_client.so`. |
| `mapd` | **Có object build sẵn** (aarch64, `OpenWrt GCC 8.4.0 r0+16841-cbe73ea33d`): `libmapd_core.a`, `libmapd_steer_core.a`, `libmapd_bs_core.a`, `libmapd_bs_steer_core.a`, `mapd_lib/*/*.o` (ch_planning_r2, networkoptimization, topologySrv, steerAction, apEstimator, lib_score, ApRoamAlgo). Phần còn lại là source. Script: `EasyMesh_openwrt5x.sh` (1597 dòng), `easymesh_init`, `easymesh_bridge_init`, `config_agent.lua`, `map_config_agent.lua`. |
| `mapfilter` | Kernel module, `MODULE_LICENSE("Dual BSD/GPL")`. Dùng netfilter bridge hooks (`NF_BR_PRE_ROUTING`, `NF_BR_FORWARD`, `NF_BR_LOCAL_OUT`), `nf_register_hooks`/`nf_register_net_hooks` (chọn theo `LINUX_VERSION_CODE`), `netlink_kernel_create`, `nf_register_sockopt`. |

Phụ thuộc runtime mà script MTK cần nhưng tree **chưa có**:

- `wificonf` (binary MTK, cú pháp `wificonf -f <file> set|get <key> [value]`) —
  dùng ở `mapd/config_agent.lua` (26 lần), `mapd/cert_7622.sh`,
  `mapd/src/networkoptimization/network_optimization.c`, `wappd/dpp_config_agent_impl.lua`.
- lua module `socket` (`require("socket")`) → package `luasocket` (feed packages).
- Script đọc `/etc/wireless/l1profile.dat` (66 lần), dùng `datconf`/`l1dat` (lua).

---

## 3. Cách làm việc / build

Checkout này **chưa có** `feeds/`, `.config`, `tmp/`. Lần đầu:

```bash
bash scripts/install-deps.sh          # nếu máy chưa có deps
bash scripts/prepare-build.sh defconfig/viettel-nr3053-easymesh.config   # sau Phase 1
make download -j8
make tools/install toolchain/install -j"$(nproc)"
make target/linux/compile -j"$(nproc)"     # cần cho kmod + LINUX_DIR headers
```

Build từng package (lặp nhanh):

```bash
make package/<pkg>/{clean,compile} V=s -j1 2>&1 | tee logs/<pkg>.log
# <pkg> là thư mục: libmapd, wappd, 1905daemon, mapd, mapfilter
```

Output: `bin/packages/aarch64_cortex-a53/base/` hoặc `bin/targets/mediatek/filogic/packages/` (kmod).
Kiểm tra ELF: `file build_dir/target-aarch64_cortex-a53_musl/<pkg>/<binary>`,
`staging_dir/toolchain-*/bin/aarch64-openwrt-linux-musl-readelf -d <binary>` để xem `NEEDED`.

Build image cuối: `bash scripts/build-viettel.sh defconfig/viettel-nr3053-easymesh.config`
(bước bootloader clone repo ngoài, có thể bỏ qua khi lặp bằng
`make -j"$(nproc)"` rồi lấy `bin/targets/mediatek/filogic/*viettel_nr3053*`).

Test: boot `*-initramfs-recovery.itb` qua TFTP trước (không ghi NAND), xem
[docs/huong-dan-nap-firmware.md](../docs/huong-dan-nap-firmware.md).

---

## 4. Các phase

Mỗi phase có **Việc**, **Xong khi**, **Nếu kẹt**. Làm tuần tự; Phase 2.x có thể
song song sau khi 2.1 xong.

### Phase 1 — Nối source và defconfig

**Việc**

1. Mỗi recipe (`libmapd`, `wappd`, `1905daemon`, `mapd`, `mapfilter`) thêm:
   ```make
   PKG_SOURCE_URL:=file://$(TOPDIR)/easymesh/dl
   PKG_HASH:=<sha256 từ easymesh/SHA256SUMS>
   ```
   `scripts/download.pl:170` hỗ trợ mirror `file://` (tìm file trong thư mục,
   copy vào `dl/`). Nếu `file://` + `$(TOPDIR)` gặp vấn đề, phương án B là
   `PKG_SOURCE_URL:=https://raw.githubusercontent.com/immortalwrt-mt798x/immortalwrt/c114d749185be6b846e8d38265549049d49b994d/dl`
   (giống cách `mt_wifi` đang tải driver), hash giữ nguyên.
2. Thêm `PKG_LICENSE:=Proprietary` (wappd, 1905daemon, mapd, libmapd) và
   `PKG_LICENSE:=BSD-3-Clause OR GPL-2.0-only` (mapfilter) để metadata trung thực.
3. Tạo `defconfig/viettel-nr3053-easymesh.config` = copy `viettel-only.config` +:
   ```
   CONFIG_PACKAGE_wapp=m
   CONFIG_PACKAGE_1905daemon=m
   CONFIG_PACKAGE_libmapd=m
   CONFIG_PACKAGE_mapd=m
   CONFIG_PACKAGE_kmod-mapfilter=m
   CONFIG_MAP_R2_SUPPORT=y
   CONFIG_MAP_R3_SUPPORT=y
   CONFIG_PACKAGE_luasocket=m
   CONFIG_PACKAGE_datconf-lua=m
   CONFIG_PACKAGE_kvcedit=m
   CONFIG_TARGET_DEVICE_PACKAGES_mediatek_filogic_DEVICE_viettel_nr3053="wapp 1905daemon libmapd mapd kmod-mapfilter luasocket datconf-lua kvcedit easymesh-nr3053"
   ```
   Giữ nguyên 2 device để `scripts/verify-viettel-config.sh` (yêu cầu đúng 2
   device) vẫn pass; nhờ `PER_DEVICE_ROOTFS`, 32X6 không nhận các package này.
   `easymesh-nr3053` là package glue tạo ở Phase 4 — thêm vào chuỗi khi nó tồn tại.
4. `make defconfig` với file mới rồi chạy `scripts/verify-viettel-config.sh .config`.

**Xong khi**: `make package/{libmapd,wappd,1905daemon,mapd,mapfilter}/download V=s`
thành công, hash khớp; `.config` có đủ symbol trên; verify script pass.

**Nếu kẹt**: symbol không xuất hiện trong `.config` → `./scripts/feeds install -a`
và kiểm `tmp/.packageinfo` có package chưa (`grep -A3 'Package: mapd' tmp/.packageinfo`).

### Phase 2 — Compile từng package trên 6.12

#### 2.1 `libmapd`

- Build `make package/libmapd/compile V=s`.
- Recipe dùng target `shared` và `include ./mapd_config` — giữ nguyên.
- Xong khi: `staging_dir/target-*/usr/lib/libmapd_interface_client.so*` và
  `usr/include/mapd_interface_ctrl.h` tồn tại.

#### 2.2 `wappd` (package tên `wapp`)

- `MAKE_FLAGS` hiện có:
  `UAPD_INCLUDE_PATH=$(LINUX_DIR)/include/uapi/linux/wapp` và
  `NL80211_INCLUDE_PATH=$(LINUX_DIR)/include/uapi/linux/mtk_nl80211_inc`.
  - Kiểm UAPI WAPP có được copy vào `$(LINUX_DIR)/include/uapi/linux/wapp/`
    sau `target/linux/compile` (từ `files-6.12`). Nếu không, trỏ thẳng tới
    `$(TOPDIR)/target/linux/mediatek/files-6.12/include/uapi/linux/wapp`.
  - Source wappd **không** include `mtk_nl80211_inc` (đã grep) → nếu đường dẫn
    không tồn tại mà build vẫn pass thì để nguyên hoặc bỏ dòng.
  - Makefile gốc dùng `$(ROOTDIR)/$(LINUXDIR)/include/uapi/linux/wapp/` → có
    thể phải truyền `ROOTDIR=` rỗng và `LINUXDIR=$(LINUX_DIR)`, hoặc patch.
- `DEPENDS` cần `+libkvcutil +libdatconf`? Recipe gốc chỉ có `+libkvcutil`;
  Makefile source link `-ldatconf` → thêm dependency package cung cấp
  `libdatconf.so` (xem `package/mtk/applications/datconf/Makefile` — package
  `datconf` hoặc tách lib). Kiểm `readelf -d` sau build.
- Áp patch `patches/001-rtmpiapp.h.patch` (đã có).
- Lỗi thường gặp GCC 14: `-Werror`, implicit function declaration, `incompatible-pointer-types`
  → ưu tiên sửa source bằng patch; tạm thời có thể thêm
  `TARGET_CFLAGS += -Wno-error -Wno-implicit-function-declaration -Wno-incompatible-pointer-types`
  và ghi TODO.
- `Config.in`: để `WAPP_WPACTRL_SUPPORT=n` (không có `libwpactrl` trong tree).
- Xong khi: `wapp`, `wappctrl` là ELF aarch64, `NEEDED` chỉ chứa lib có package.

#### 2.3 `1905daemon`

- `PKG_KCONFIG` liệt kê symbol SoC cũ (`RALINK_MT7621`...) — vô hại, bỏ qua.
- `-I$(LINUX_DIR)/drivers/net/ethernet/raeth` không tồn tại trên 6.12: tìm
  trong source header nào cần (`grep -rn raeth`, `ra_ioctl.h`, `raether`).
  Với `SWTICH_DSA_ARCH=y` có thể không cần; nếu cần, thêm stub header qua patch.
- `-I$(LINUX_DIR)/user_headers/include` — kiểm đường dẫn tồn tại trên 25.12
  (`ls build_dir/target-*/linux-*/linux-6.12*/user_headers`); nếu không, dùng
  `$(STAGING_DIR)/usr/include` (kernel headers đã có trong toolchain).
- Dòng `-I$(PKG_BUILD_DIR) \` và `-lnl-tiny -lm \` có `\` cuối dòng nhưng
  dòng kế là dòng trống nên vô hại; dọn lại khi sửa recipe.
- Xong khi: `p1905_managerd`, `1905ctrl`, `ethernet/libeth_1905ops.so` build được.

#### 2.4 `mapd`

- Phụ thuộc `libmapd` (build dep) và `wapp`, `1905daemon` (runtime).
- **Rủi ro lớn nhất**: link object GCC 8.4 với toolchain hiện tại (GCC 14,
  musl 1.2.x). Trên aarch64 `time_t` vốn 64-bit nên không có đổi tên symbol
  time64; vẫn kiểm:
  - Link lỗi `undefined reference` tới symbol libc/libgcc cũ → xác định symbol,
    thêm shim nhỏ (file C trong patch) hoặc `-lgcc`.
  - `-fno-common` mặc định từ GCC 10 → nếu lỗi `multiple definition` ở object
    *source* (không phải prebuilt) thêm `TARGET_CFLAGS += -fcommon`.
  - Prebuilt có thể giả định layout struct từ header trong source (vd
    `wapp_cmm_type.h`): **không** đổi struct dùng chung với prebuilt.
- Bỏ install `EasyMesh_openwrt5x.sh` bằng `INSTALL_BIN` vào `EasyMesh_openwrt.sh`
  như recipe gốc — giữ, nhưng sẽ được thay/bọc ở Phase 4.
- Xong khi: `mapd`, `mapd_cli`, `bs20`, `mapd_user_iface` link được; chạy
  `qemu-aarch64 -L staging_dir/target-*/root-mediatek ./mapd -h` (nếu có qemu-user)
  không crash ở loader.

**Nếu kẹt hẳn ở link prebuilt**: dừng, ghi lại symbol thiếu và báo chủ repo —
không có source để build lại các object này; phương án còn lại là tìm
archive `mapd` revision khác từ SDK MTK.

#### 2.5 `mapfilter` (kmod)

- Recipe build với `$(MAKE) -C $(LINUX_DIR) M=...` — chuyển sang dạng chuẩn
  `$(KERNEL_MAKE) M="$(PKG_BUILD_DIR)" ...` và `KernelPackage/mapfilter` cần
  `DEPENDS:=+kmod-br-netfilter` (hoặc `@KERNEL_BRIDGE_NETFILTER`) vì dùng
  `NF_BR_*` hooks.
- API kernel 6.12 cần kiểm trong `mtkmapfilter.c`, `service_prioritization.c`,
  `traffic_separation.c`, `load_balance.c`, `dual_bh.c`:
  - `nf_register_hooks` đã bị xóa từ 4.13; cả 5 file đều đã có nhánh
    `LINUX_VERSION_CODE` chọn `nf_register_net_hooks(&init_net, ...)` (đã kiểm)
    — chỉ cần xác nhận nhánh `#else` tương ứng cho `nf_unregister_*`.
  - Prototype hook: `unsigned int fn(void *priv, struct sk_buff *skb, const struct nf_hook_state *state)`.
  - `netlink_kernel_create(&init_net, unit, &cfg)` — nhánh cuối đã đúng.
  - `nf_register_sockopt`: struct `nf_sockopt_ops` — `get`/`set` nhận `sockptr_t`
    cho `set` từ 5.9 → patch.
  - `MAP_NETLINK` protocol number không đụng số đã dùng trong 6.12 (`include/uapi/linux/netlink.h`, max 31).
  - `skb->...`, `dev_get_by_name`, `timer_setup`, `proc_create` (proc_ops từ 5.6), `ktime`/`do_gettimeofday` (xóa) — grep và sửa.
- Kiểm `mapfilter` có gọi symbol export của `mt_wifi` không (`grep -n EXPORT_SYMBOL` trong driver / `extern` trong mapfilter).
- Xong khi: `mapfilter.ko` build, `modinfo` đúng vermagic 6.12; `insmod` trên thiết bị không oops (Phase 5).

### Phase 3 — Runtime dependencies

1. **`wificonf`**: chưa có trong tree. Hai hướng, ưu tiên (a):
   - (a) Shim shell `/usr/bin/wificonf` bọc `kvcedit` (package `kvcedit` từ
     `datconf`). Đọc source `kvcutil` (tải về sau `make package/datconf/prepare`,
     xem `build_dir/target-*/datconf*/kvcutil/`) để map cú pháp:
     `wificonf -f FILE get KEY` / `wificonf -f FILE set KEY VALUE`. Viết test
     shell nhỏ chạy trên host với file `.dat` mẫu.
   - (b) Tìm recipe `wificonf` trong `mtk-openwrt-feeds`
     (`https://git01.mediatek.com/plugins/gitiles/openwrt/feeds/mtk-openwrt-feeds`)
     — chưa xác minh có tồn tại.
2. **`luasocket`**: từ feed `packages` (`./scripts/feeds install luasocket`).
3. **`datconf-lua`, `lua`, `libuci-lua`**: xác nhận các script `.lua` chạy được
   (`lua -e 'require "datconf"'`, `require "uci"`).
4. `/etc/wireless/l1profile.dat` đã có từ `wifi-dats`; kiểm nó vẫn được cài
   khi dùng `mtwifi-cfg-ucode`.

**Xong khi**: trên thiết bị (initramfs), `wificonf -f /tmp/test.dat set A 1 && wificonf -f /tmp/test.dat get A` in `1`; `lua -e 'require "socket"'` không lỗi.

### Phase 4 — Glue: package `easymesh-nr3053`

Tạo `package/mtk/applications/easymesh-nr3053/` (fork-owned, license GPL-2.0, không chứa code MTK):

1. UCI `/etc/config/easymesh`:
   ```
   config easymesh 'main'
       option role 'disabled'      # disabled | controller | agent
       option backhaul 'auto'      # auto | wired | wireless
   ```
   Không có SSID/khóa mặc định (R4). SSID/khóa fronthaul lấy từ `/etc/config/wireless` hiện có.
2. Init `/etc/init.d/easymesh` (procd, START sau `network`):
   - `role=disabled` → không chạy gì, đảm bảo `MapMode=0` trong 2 file `.dat`.
   - Khác → set key MAP vào `.dat`, `wifi reload`, insmod `mapfilter`, khởi
     động `wapp` → `p1905_managerd` → `mapd` theo thứ tự stock
     (tham khảo `EasyMesh_openwrt5x.sh`, `wapp.init`, `easymesh_bridge_init`).
   - Ghi đè/disable `/etc/init.d/easymesh` gốc của package `mapd` (đang trỏ
     tới `EasyMesh_openwrt.sh`) — đổi tên file install trong recipe `mapd`
     để tránh trùng (vd `easymesh-mtk`), init của glue mới là entry point.
3. **Xung đột `mtwifi-cfg-ucode`**: nó sinh lại `.dat` từ UCI mỗi lần
   `wifi up` → mất key MAP. Hướng sửa (chọn một, ghi lý do vào commit):
   - (a) Thêm hook: sau khi `mtwifi-cfg-ucode` ghi `.dat`, chạy
     `/usr/libexec/easymesh/apply-dat` để chèn key MAP khi role ≠ disabled.
     Tìm điểm ghi `.dat` trong `usr/share/ucode/mtwifi/config.uc`/`driver.uc`.
   - (b) Thêm option UCI `map_mode` cho `wifi-device` vào `dat-defs.json` +
     `converter.uc` (thay đổi chạm package của upstream — tách patch rõ).
   Danh sách key MAP cần có: lấy từ `EasyMesh_openwrt5x.sh` (grep `MapMode`,
   `MAP_Turnkey`, `MAP_Ext`, `BhBss`, `FhBss`...), và đối chiếu
   `/etc/wireless/mediatek/*.dat` của firmware stock khi `MapMode=1` nếu có.
4. Bridge/DSA: `easymesh_bridge_init` và `1905d.cfg`/`ethernet_cfg.txt` giả
   định tên interface của SDK. Map sang `br-lan`, `lan1..3`, `wan`, `ra0`,
   `rax0`, `apcli0`, `apclix0`. Cấu hình wired backhaul: port LAN nào làm
   uplink cho agent.
5. CLI tối thiểu: `easymesh status` (role, tiến trình, `mapd_cli` topology).

**Xong khi**: role `disabled` → boot y như bản gốc (so `ps`, `iwinfo`, file
`.dat` với bản gốc). Role `controller` → `wapp`, `p1905_managerd`, `mapd`
chạy ổn định ≥10 phút, `logread` không lặp lỗi.

### Phase 5 — Test phần cứng (2 × NR3053)

Chạy từ `initramfs-recovery.itb`, có UART. Ghi kết quả vào `easymesh/TESTLOG.md`.

| # | Ca | Mong đợi |
|---|---|---|
| T1 | Boot, role=disabled | Như firmware gốc, không có tiến trình mesh, không kmod mapfilter |
| T2 | `insmod mapfilter` / `rmmod` | Không oops trong `dmesg` |
| T3 | A=controller, B=agent, cáp LAN↔LAN | `mapd_cli`/`1905ctrl` thấy 2 node; B phát cùng SSID fronthaul |
| T4 | Rút cáp, wireless backhaul (5 GHz) | B lên lại qua `apclix0` |
| T5 | Onboarding WPS (nút / `wps_pbc`) | Agent mới join không cần cấu hình tay |
| T6 | Client roaming giữa A và B | Steering/roaming hoạt động, không mất kết nối lâu |
| T7 | Reboot từng node, restart service | Topology tự phục hồi |
| T8 | Chuyển role → disabled | Quay về Wi-Fi thường sau `wifi reload` |
| T9 | `firstboot` / factory reset | Mesh tắt, không sót cấu hình |
| T10 | Đo RAM/CPU (`free`, `top`) | Ghi số; NR3053 có 512 MB nên chỉ cần không rò rỉ |

### Phase 6 (tùy chọn) — Tương thích node stock

Thử NR3053 custom làm agent cho NR3053 stock làm controller (bật EasyMesh trên
stock) và ngược lại. Ghi version R2/R3 mỗi bên.

### Phase 7 — Tài liệu

1. Sửa [docs/trien-khai-mediatek-easymesh.md](../docs/trien-khai-mediatek-easymesh.md):
   archive có trong `dl/` upstream; bỏ tham chiếu `scripts/audit-easymesh-sources.sh`
   (hoặc viết script đó); ghi kết quả license đã kiểm (mục 2.3).
2. Cập nhật `easymesh/PLAN.md` mục "Tiến độ" bên dưới sau mỗi phase.

---

## 5. Rủi ro và điểm dừng

| Rủi ro | Dấu hiệu | Hành động |
|---|---|---|
| Object prebuilt `mapd` không link/chạy với toolchain mới | Undefined symbol libc, crash sớm | Thử shim; nếu không được → dừng, báo chủ repo |
| `mapfilter` cần sửa nhiều cho 6.12 | Nhiều lỗi API netfilter | Sửa dần bằng patch; nếu mesh chạy được không cần mapfilter (traffic separation tắt) thì để sau |
| `mtwifi-cfg-ucode` ghi đè `.dat` | MapMode về 0 sau `wifi reload` | Phase 4.3 |
| Driver cần build flag khác khi MAP chạy (vd `WAPP_SUPPORT` trong driver) | `wapp` không nhận event từ driver | grep `WAPP_SUPPORT`/`MAP_SUPPORT` trong source driver `build_dir/.../mt_wifi*/` và Kconfig, so với stock |
| License | — | Giữ R1 tuyệt đối |

---

## 6. Tiến độ

| Phase | Trạng thái | Ghi chú |
|---|---|---|
| 0 Baseline | Bỏ qua | Firmware gốc đã chạy ổn |
| Chuẩn bị branch | Xong | Import recipe + archive nguyên văn |
| 1 Source + defconfig | Chưa | |
| 2.1 libmapd | Chưa | |
| 2.2 wappd | Chưa | |
| 2.3 1905daemon | Chưa | |
| 2.4 mapd | Chưa | |
| 2.5 mapfilter | Chưa | |
| 3 Runtime deps | Chưa | |
| 4 Glue | Chưa | |
| 5 Test HW | Chưa | |
| 6 Stock interop | Chưa | |
| 7 Tài liệu | Chưa | |
