# Triển khai MediaTek EasyMesh cho Viettel NR3053

> **Trạng thái:** nghiên cứu, chưa phát hành. Firmware public hiện không có package hay giao diện MediaTek EasyMesh; tài liệu này không phải hướng dẫn cài mesh.

Xem [README](../README.md) để biết các tính năng đang có trong firmware, hoặc [hướng dẫn tính năng mở rộng](huong-dan-tinh-nang-mo-rong.md) cho WireGuard, DDNS và Adblock.

## Mục tiêu

Đưa MediaTek EasyMesh/Multi-AP native vào bản ImmortalWrt cho **NR3053** để một
node custom có thể làm controller hoặc agent cùng các node dùng stack MediaTek
tương thích. Đây không phải 802.11s; nó cần control plane IEEE 1905.1/Multi-AP
bên cạnh driver Wi-Fi. Phạm vi hiện tại không bao gồm 32X6.

## Hiện trạng đã xác minh

### Firmware stock NR3053

Stock có đủ stack MediaTek EasyMesh Release 3:

- `1905daemon` cung cấp `p1905_managerd` và lớp IEEE 1905.1.
- `mapd` và `libmapd` thực hiện controller/agent Multi-AP.
- `wapp`, `mesh_utils`, `meshled`, `wifi-profile`, `wificonf`, `ated_ext` và
  `8021xd` là các thành phần phụ trợ.
- `mapfilter` là kernel module dùng cho một số chế độ mesh.

Tại thời điểm audit, stock để hai radio ở `MapMode=0`. Vì vậy package và UBus
mesh có mặt nhưng `mapd`/`p1905_managerd` không chạy. Đây là chế độ Wi-Fi bình
thường, không phải lỗi hoặc bằng chứng mesh đang hoạt động.

### ImmortalWrt fork

Image custom hiện dùng driver MediaTek `mt_wifi` 7.6.7.3 và đã bật:

```
CONFIG_MTK_MAP_SUPPORT=y
CONFIG_MTK_MAP_R2_VER_SUPPORT=y
CONFIG_MTK_MAP_R3_VER_SUPPORT=y
```

Driver có năng lực MAP R3, nhưng image chưa package control plane `mapd`,
`1905daemon`, `wapp`, `libmapd` hoặc `mapfilter`. `wpad-openssl` có thể phục vụ
802.11s/Multi-AP ở mức hostapd, nhưng một mình nó không thay thế controller
IEEE 1905.1 của MediaTek EasyMesh.

## Nguồn đã tìm thấy

Repository công khai sau chứa recipe khớp driver 7.6.7.3:

- https://github.com/immortalwrt-mt798x/immortalwrt
- Branch: `openwrt-21.02`.
- Revision đã audit: `c114d749185be6b846e8d38265549049d49b994d`.
- Driver pin: `mt79xx_20250408-705eb4.tar.xz`, version `7.6.7.3`.

Các recipe liên quan là `1905daemon`, `mapd`, `wappd`, `libmapd` và
`mapfilter`. Năm archive tương ứng **có** nằm trong tree Git upstream (thư mục
`dl/`) và đã được vendor vào `easymesh/dl/` của branch này (hash trong
`easymesh/SHA256SUMS`). Recipe tại revision trên không khai báo
`PKG_SOURCE_URL`; branch này trỏ nó tới `file://$(TOPDIR)/easymesh/dl`. Điều
này chứng minh stack cùng thế hệ driver tồn tại; nó không tự chứng minh tương
thích với kernel 6.12.

## Ràng buộc license và ABI

Đã kiểm file `LICENSE` trong archive: `wappd`, `1905daemon` và `mapd` mang
license độc quyền MediaTek ("confidential and proprietary"; cấm sao chép/phân
phối). `libmapd` là source MTK, ghi `Proprietary` trong recipe. `mapfilter` khai
báo `MODULE_LICENSE("Dual BSD/GPL")`. Archive `mapd` còn chứa object build sẵn
(aarch64, GCC 8.4) không có source. Không đưa source archive, binary, hay
firmware chứa chúng vào repository/release public; branch `feature/easymesh-*`
không được merge vào `main` hay mở PR.

Các recipe tìm được viết cho OpenWrt 21.02/kernel 5.4. Port sang fork hiện tại
cần kiểm tra và có thể phải sửa:

1. UAPI WAPP/netlink của kernel 6.12.
2. `mapfilter` kernel module và API netlink của nó.
3. Dependency user-space của WAPP/MAP, bao gồm thư viện MTK đi kèm.
4. DSA/switch interface mapping của NR3053.
5. Service lifecycle `procd`, khởi tạo radio, rollback và disable mesh.

Không copy binary từ firmware stock 5.4: cách đó không tái lập được dependency
hoặc ABI, khó audit và không phù hợp để phát hành.

### Kiểm kê nguồn trước khi port

Kiểm hash archive bằng `cd easymesh && sha256sum -c SHA256SUMS`. Script
`scripts/audit-easymesh-sources.sh` không tồn tại; kế hoạch chi tiết nằm trong
[easymesh/PLAN.md](../easymesh/PLAN.md).

## Lộ trình triển khai khi có quyền dùng SDK

1. Tìm source archive/SDK đúng revision, lưu hash và kiểm tra license đi kèm.
   Chỉ chốt nguồn phân phối công khai khi license hoặc quyền phân phối cho phép.
2. Port package recipes tối thiểu: `libmapd`, `wapp`, `1905daemon`, `mapd` và
   `mapfilter`; build riêng từng package trên kernel 6.12.
3. Viết adapter UCI/procd cho NR3053. Không ship SSID, passphrase hoặc định danh
   mẫu; chỉ tạo chúng tại runtime từ cấu hình do người dùng nhập.
4. Đưa UI/CLI tối thiểu cho các role disabled/controller/agent, có nút disable
   và đường rollback về Wi-Fi thường.
5. Kiểm thử hai NR3053: boot khi mesh tắt, wired backhaul, wireless backhaul,
   WPS onboarding, roaming, restart và factory reset.
6. Chỉ công bố artifact sau khi kiểm tra license, build, boot và phần cứng đều
   pass.

## Phương án thay thế

`802.11s` cùng `mesh11sd` là hướng OpenWrt hoàn toàn độc lập và dễ phân phối
hơn. Nó không tương thích trực tiếp với MediaTek EasyMesh stock, nên chỉ phù
hợp khi mọi node trong mesh đều chuyển sang cấu hình OpenWrt tương ứng.

## Tham khảo

- IEEE 1905.1: https://standards.ieee.org/ieee/1905.1/4995/
- OpenWrt 802.11s: https://openwrt.org/docs/guide-user/network/wifi/mesh/802-11s
- Thảo luận firmware MTK SDK 7.6.7.3/EasyMesh:
  https://www.right.com.cn/forum/thread-8461412-1-1.html
